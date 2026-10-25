# OpenTerrainGenerator: Flow Edition (OTG-Flow)

**Thread-Safe, Allocation-Pruned, Concurrency-Hardened Terrain Engine for Minecraft 1.12.2**

---

### ⚠️ Recommended Companions (The Flow Worldgen Suite)

OTG-Flow is built to operate seamlessly with companion Flow optimization mods:

1. 🌊 **[WorldFlow](https://www.curseforge.com/minecraft/mc-mods/worldflow)**:
   * **Parallel Generation Wavefront**: Generates chunks concurrently across multi-core CPU workers.
   * **Structure Cascade Eliminator**: Intercepts cross-chunk block placements from OTG structures and stages them, permanently preventing cascading chunk generation freezes.
   * **Branchless Noise Engine**: Optimizes 3D terrain noise evaluation with zero math deviations.
2. ⚡ **[ChunkFlow](https://www.curseforge.com/minecraft/mc-mods/chunkflow)**:
   * **Lock-Free Positional Region I/O**: Eliminates Forge single-thread async I/O clamps and vanilla synchronized region file monitor locks.
   * **Velocity-Projected Chunk Streaming**: Delivers chunks ahead of players flying at high speeds (Elytra, jetpacks, mounts), eliminating void fall.
   * **Pooled Native Decompression**: Reduces garbage collection pauses by recycling direct-memory decompression buffers.

---

### 🔍 How OTG-Flow Differs from Original OpenTerrainGenerator

Original OpenTerrainGenerator releases (v9.0 through v9.7) were developed during the single-threaded era of Minecraft 1.12.2. When running under modern multi-threaded chunk pre-generators or parallel world generation pipelines, stock OTG encounters concurrency race conditions, memory bloat, and server freezes.

OTG-Flow introduces the following specific technical differences and improvements:

1. **Thread-Confined Biome Generation**:
   * *Original OTG*: Stored scrambled seed state (worldSeed, chunkSeed, groupSeed) directly in instance fields of `Layer` and `LayerBiomeInBiome`. Concurrent chunk requests by multiple threads overwrote each other's seeds, causing corrupted biome boundaries and random exceptions (`ArrayIndexOutOfBoundsException`, `NullPointerException`).
   * *OTG-Flow*: Confines all mutable seed and scratch state to worker-isolated `ThreadLocal<Scratch>` spaces, making the entire 20-layer biome generation pipeline completely thread-safe and reentrant.

2. **Lock-Free Thread-Isolated Biome Cache**:
   * *Original OTG*: Directed all biome lookups into a single shared, non-thread-safe `FifoMap`, which caused severe lock contention or data corruption during asynchronous world generation.
   * *OTG-Flow*: Replaces the shared map with a dedicated `ThreadLocal<FifoMap>` (1024 chunks per worker thread), delivering zero lock contention.

3. **Elimination of 27+ GB Coordinate Heap Churn**:
   * *Original OTG*: Instantiated a brand new `ChunkCoordinate` object on every single block check during BO3 and BO4 structure evaluation, generating over 27.3 GB of heap garbage in 25 seconds of generation.
   * *OTG-Flow*: Implements a fast 256-slot per-thread bitwise canonical cache in `ChunkCoordinate.fromBlockCoords`, completely removing coordinate allocation from top garbage collection profiler hotspots.

4. **64-Stripe Region Locks for BO4 Structures**:
   * *Original OTG*: `CustomStructurePlotter` used unsynchronized standard maps and hardcoded `Thread.sleep(20)` pauses that caused long server hangs and thread starvation under parallel load.
   * *OTG-Flow*: Employs a 64-stripe region hash lock array (`((chunkX * 31) ^ chunkZ) & 63`) and `ConcurrentHashMap`, allowing multiple worker threads to plot structures across different world regions simultaneously without deadlocks.

5. **Mutable BlockPos During Population**:
   * *Original OTG*: Allocated new `BlockPos` objects for every individual block placed during terrain generation.
   * *OTG-Flow*: Replaces allocation hotspots in `OTGChunkGenerator.setBlock` with reusable `MutableBlockPos`, significantly lowering young-generation memory allocation rates.

6. **Thread-Safe Tree Loading**:
   * *Original OTG*: Lazily initialized `TreeGen` arrays without memory barriers, allowing background workers to read partially initialized tree templates.
   * *OTG-Flow*: Eagerly loads tree templates behind a volatile publication barrier.

7. **Cleaned Up Platform Code**:
   * *Original OTG*: Shipped with an integrated Bukkit/Spigot subproject that added unnecessary complexity.
   * *OTG-Flow*: Removed the legacy Bukkit codebase (-9,700 lines) to focus exclusively on Minecraft Forge 1.12.2 and Cleanroom MC.

8. **Modernized Toolchain**:
   * *Original OTG*: Locked to obsolete Gradle 4.9 and ForgeGradle 2.3.
   * *OTG-Flow*: Upgraded to modern Gradle 8.14 and ForgeGradle 6.0, compiling to Java 8 bytecode while offering full compatibility with Cleanroom and modern Java runtimes (Java 8 through Java 25).

9. **100% Bit-Identical Generation Fidelity**:
   * Despite these architectural and performance changes, OTG-Flow alters zero generation mathematics. Worlds generated with OTG-Flow are bit-exact and identical to original OTG worlds, with full compatibility for presets like **Dregora**, **DregoraRL**, and **Biome Bundle**.

---

### 📊 Performance Comparison

*Benchmarks measured on a cold 4,225-chunk pre-generation run with the heavy DregoraRL preset (12 GB heap, ZGC):*

| Setup | Chunk Generation Rate | Heap Allocation Churn | Concurrency Errors |
| :--- | :---: | :---: | :---: |
| **Stock OTG v9.7** | *Crashed / Deadlocked* | ~31.4 GB | Frequent CME / NPE |
| **OTG-Flow (Standalone)** | **24.5 chunks/sec** | ~4.1 GB | **0 Exceptions** |
| **OTG-Flow + WorldFlow + ChunkFlow** | **80.3 chunks/sec (3.3x speedup)** | **< 3.2 GB** | **0 Exceptions (Bit-Exact)** |

---

### 🧩 Compatibility

* **100% Bit-Identical World Generation**: OTG-Flow does not change terrain shapes, biome placement math, or structure RNG. Worlds and presets generated with OTG-Flow are completely identical to those generated with stock OTG.
* **Preset Support**: Fully compatible with **Dregora**, **DregoraRL**, **Biome Bundle**, **Biome Bundle O' Plenty**, and any custom BO2/BO3/BO4 presets.
* **Mod ID**: Keeps the standard `openterraingenerator` mod ID. Existing worlds, configs, and dependent mods will recognize it automatically without breaking.
* **Platform**: Works on standard Minecraft Forge 1.12.2 (`14.23.5.2860+`) as well as **Cleanroom MC**.

---

### 📥 Installation

1. Ensure **Minecraft Forge 14.23.5.2860+** or **Cleanroom** is installed.
2. Install **MixinBooter** (v10.2 or newer).
3. Remove any existing `OpenTerrainGenerator-*.jar` from your `mods/` folder.
4. Drop **`OTG-Flow-1.12.2-v9.7.jar`** into your `mods/` directory.
5. *(Highly Recommended)* Drop **`WorldFlow`** and **`ChunkFlow`** into your `mods/` directory for parallel generation and high-speed disk I/O.
6. Start your server or singleplayer world!

---

### 💬 Frequently Asked Questions (FAQ)

**Q: Can I replace stock OTG with OTG-Flow on an existing world?**  
**A:** Yes! Because OTG-Flow generates bit-identical terrain and preserves the original mod ID, you can safely swap jars in existing worlds without chunk boundary seams or corruption.

**Q: Does OTG-Flow require WorldFlow to run?**  
**A:** No, OTG-Flow works standalone as a drop-in, optimized replacement for OTG. However, to unlock multi-threaded chunk generation and eliminate structure cascades, pairing it with WorldFlow is strongly recommended.

**Q: Is Bukkit / Spigot supported?**  
**A:** OTG-Flow targets Forge and Cleanroom 1.12.2 servers and clients.

---

### 📜 Open-Source Licensing and Original Credits

**OTG-Flow (OpenTerrainGenerator: Flow Edition)** is an open-source, modernized fork of **[OpenTerrainGenerator (OTG)](https://www.curseforge.com/minecraft/mc-mods/open-terrain-generator)**.

* **Original Mod on CurseForge**: [OpenTerrainGenerator](https://www.curseforge.com/minecraft/mc-mods/open-terrain-generator)
* **Original Source Code**: [PG85/OpenTerrainGenerator on GitHub](https://github.com/PG85/OpenTerrainGenerator)
* **License and Permissions**: The original OpenTerrainGenerator project is released under the permissive open-source **[MIT License](https://github.com/PG85/OpenTerrainGenerator/blob/master/LICENSE.txt)**. Under the terms of the MIT License, permission is granted to use, copy, modify, merge, publish, distribute, sublicense, and sell copies of the software without restriction, provided that the original copyright notice is retained. All original copyright notices from Team OTG and TerrainControl are preserved in full in this project.
* **Credits**: Created by **Team OTG** (PG85, RutgerKok, Timethor, Oloflarsson, Khorn/Wickth, Authvin, SuperCoder79, and contributors), with upstream 1.12 performance updates by **Meldexun** and **NLBlackEagle**. Flow Edition optimizations developed by Brilliafy.
