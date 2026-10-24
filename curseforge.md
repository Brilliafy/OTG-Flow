# OpenTerrainGenerator: Flow Edition (OTG-Flow)

**Thread-Safe, Allocation-Pruned, Concurrency-Hardened Terrain Engine for Minecraft 1.12.2**

---

### ⚠️ IMPORTANT: Recommended Companions (The "Flow" Worldgen Trinity)
**OTG-Flow** is optimized specifically for parallel, multi-threaded chunk generation. To experience smooth, freeze-free terrain generation without server stalls or cascading chunk generation, **we strongly recommend installing OTG-Flow alongside its companion Flow mods**:

1. 🌊 **[WorldFlow](https://www.curseforge.com/minecraft/mc-mods/worldflow)**:
   - **Parallel Generation Wavefront**: Generates chunks in parallel across all available CPU cores.
   - **Dregora / OTG Structure Cascade Eliminator**: Intercepts cross-chunk block placements from OTG structures and stages them, permanently eliminating cascading chunk generation freezes.
   - **Branchless Noise Engine**: Optimizes 3D terrain noise evaluation with zero math deviations.
2. ⚡ **[ChunkFlow](https://www.curseforge.com/minecraft/mc-mods/chunkflow)**:
   - **Lock-Free Positional Region I/O**: Eliminates Forge’s 1-thread async I/O clamp and vanilla’s synchronized `.mca` file monitor locks.
   - **Velocity-Projected Chunk Streaming**: Delivers chunks ahead of players flying at high speeds (Elytra, jetpacks, mounts), eliminating void fall.
   - **Pooled Native Decompression**: Cuts GC pauses by recycling direct-memory decompression buffers.

---

## 🌟 What is OTG-Flow?

**OpenTerrainGenerator (OTG)** is the legendary custom terrain engine behind iconic Minecraft 1.12.2 world presets such as **Dregora**, **DregoraRL**, and **Biome Bundle**. 

However, original OTG releases (v9.0–v9.7) were written in an era where Minecraft servers ran strictly on a single thread. When modern modpacks and servers attempt to pre-generate chunks or generate terrain asynchronously across multiple worker threads, original OTG suffers from:
- **Thread race conditions** in biome calculation, leading to corrupted biome borders and random server crashes (`ArrayIndexOutOfBoundsException`, `NullPointerException`).
- **Global cache contention** in its shared FIFO biome cache.
- **Extreme memory churn**—instantiating millions of coordinate objects (**over 27 GB of heap garbage in 25 seconds** during structure scans).
- **BO4 structure plotter locks** causing long server hangs and thread starvation.

**OTG-Flow (OpenTerrainGenerator: Flow Edition)** solves these limitations through full thread confinement, zero-allocation ring caches, striped region locks, and modern build optimizations.

---

## ⚡ Key Enhancements Over Original OTG

- 🛡️ **Thread-Confined Biome Generation**: World seed and chunk seed scratch variables in `Layer` and `LayerBiomeInBiome` are isolated per-thread using `ThreadLocal`. Multiple worker threads can now calculate biomes concurrently without data races or lock contention.
- ⚡ **Lock-Free Per-Thread Biome Cache**: Replaces the legacy shared biome map with a dedicated, thread-isolated 1024-chunk cache per worker thread.
- 🚀 **-27.3 GB Heap Garbage Elimination**: Implements a high-speed bitwise canonical cache for `ChunkCoordinate.fromBlockCoords`, eliminating coordinate object churn from the top-15 GC hotspots during BO3/BO4 structure plotting.
- 🏰 **Striped Region Locks for BO4 Structures**: Replaces unsynchronized map access and hardcoded `Thread.sleep(20)` pauses with a 64-stripe region hash lock, allowing parallel structure plotting across different world regions.
- 🏎️ **Mutable BlockPos in Terrain Population**: Eliminates young-generation garbage during block placement in `OTGChunkGenerator.setBlock`.
- 🌲 **Thread-Safe Tree Initialization**: Eagerly loads `TreeGen` arrays with volatile publication barriers, preventing worker threads from reading partially initialized foliage templates.
- 📦 **Modernized Engine & Fastutil Primitives**: Incorporates Meldexun's upstream performance overhaul, caching `BlockFunctions`, packing coordinate integers, and upgrading to fast primitive collections.
- ☕ **Cleanroom & Java 8–25 Ready**: Built with modern ForgeGradle 6.0 and Gradle 8.14, targeting Java 8 bytecode while running seamlessly on Cleanroom and modern Java runtimes (Java 17 / 21 / 25).

---

## 📊 Performance Comparison

*Benchmarks measured on a cold 4,225-chunk pre-generation run with the heavy DregoraRL preset (12 GB heap, ZGC):*

| Setup | Chunk Generation Rate | Heap Allocation Churn | Concurrency Errors |
| :--- | :---: | :---: | :---: |
| **Stock OTG v9.7** | *Crashed / Deadlocked* | ~31.4 GB | Frequent CME / NPE |
| **OTG-Flow (Standalone)** | **24.5 chunks/sec** | ~4.1 GB | **0 Exceptions** |
| **OTG-Flow + WorldFlow + ChunkFlow** | **80.3 chunks/sec (3.3x speedup)** | **< 3.2 GB** | **0 Exceptions (Bit-Exact)** |

---

## 🧩 Compatibility

- **100% Bit-Identical World Generation**: OTG-Flow does **not** change terrain shapes, biome placement math, or structure RNG. Worlds and presets generated with OTG-Flow are completely identical to those generated with stock OTG.
- **Preset Support**: Fully compatible with **Dregora**, **DregoraRL**, **Biome Bundle**, **Biome Bundle O' Plenty**, and any custom BO2/BO3/BO4 presets.
- **Mod ID**: Keeps the standard `openterraingenerator` mod ID. Existing worlds, configs, and dependent mods will recognize it automatically without breaking.
- **Platform**: Works on standard Minecraft Forge 1.12.2 (`14.23.5.2860+`) as well as **Cleanroom MC**.

---

## 📥 Installation

1. Ensure **Minecraft Forge 14.23.5.2860+** or **Cleanroom** is installed.
2. Install **MixinBooter** (v10.2 or newer).
3. Remove any existing `OpenTerrainGenerator-*.jar` from your `mods/` folder.
4. Drop **`OTG-Flow-1.12.2-v9.7.jar`** into your `mods/` directory.
5. *(Highly Recommended)* Drop **`WorldFlow`** and **`ChunkFlow`** into your `mods/` directory for parallel generation and high-speed disk I/O.
6. Start your server or singleplayer world!

---

## 💬 Frequently Asked Questions (FAQ)

**Q: Can I replace stock OTG with OTG-Flow on an existing world?**  
**A:** Yes! Because OTG-Flow generates bit-identical terrain and preserves the original mod ID, you can safely swap jars in existing worlds without chunk boundary seams or corruption.

**Q: Does OTG-Flow require WorldFlow to run?**  
**A:** No, OTG-Flow works standalone as a drop-in, optimized replacement for OTG. However, to unlock multi-threaded chunk generation and eliminate structure cascades, pairing it with WorldFlow is strongly recommended.

**Q: Is Bukkit / Spigot / Mohist supported?**  
**A:** OTG-Flow targets Forge and Cleanroom 1.12.2 servers and clients.

---

## 📜 Credits

- Original **OpenTerrainGenerator** by **Team OTG** (PG85, RutgerKok, Timethor, Oloflarsson, Khorn/Wickth, Authvin, SuperCoder79, and contributors).
- **OTG+ Performance Modernization**: Meldexun (PR #1020) and NLBlackEagle.
- **Flow Edition Architecture & Thread Confinement**: Developed by Brilliafy for the Flow Optimization Project.
- Released under the open-source **MIT License**.
