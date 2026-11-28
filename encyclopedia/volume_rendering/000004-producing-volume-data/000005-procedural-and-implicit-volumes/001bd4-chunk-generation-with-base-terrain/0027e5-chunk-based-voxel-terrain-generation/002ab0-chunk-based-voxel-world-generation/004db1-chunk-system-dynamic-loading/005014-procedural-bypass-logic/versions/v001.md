# Procedural Bypass Logic

Also called: Procedural Bypass, Regeneration from Seed

## Overview
Procedural Bypass Logic is a data management strategy for voxel worlds where pure procedural chunks are regenerated from a deterministic seed on demand rather than being persisted to disk, thereby minimizing I/O overhead [S1, unofficial]. It distinguishes itself from persistent storage models by discarding internal voxel data and calculating it only when visibility requires it (e.g., excavation), storing only user modifications to maintain state [S2, unofficial].

## How it works
**Inputs:**
- A deterministic random seed for procedural generation [S1, S2, unofficial].
- 2D map images (temperature, humidity, elevation) for surface definition [S2, unofficial].
- A list of modifications (added/removed voxels) for persistent changes [S2, unofficial].

**Outputs:**
- A voxel chunk containing visible surface and excavated internal blocks [S2, unofficial].

**Data Structures:**
- **Static Maps:** Store voxels in `(Z, Y, X)` order to align with NumPy C-order contiguity and optimize flat serialization [S1, unofficial].
- **Halo Border:** Chunks are generated with a `+1` block border (effectively `+2` in total width/height/depth relative to the core) to ensure geometry continuity across chunk boundaries [S1, unofficial].
- **Modification Store:** A sparse structure recording only removed or added voxels relative to the procedural baseline [S2, unofficial].

**Steps:**
1. **Initialization:** Load `VoxelRegistry` definitions once at startup to prevent redundant disk access [S1, unofficial].
2. **Chunk Request:** When a chunk is needed, check if it exists in the modification store.
3. **Procedural Generation (Surface):**
   - Use the deterministic seed to generate noise [S2, unofficial].
   - Sample 2D images (temperature, humidity, elevation) [S2, unofficial].
   - Mix noise with map data to determine surface voxel types (e.g., water, sand, grass, snow) [S2, unofficial].
   - Use elevation maps to determine surface height features (mountains, etc.) [S2, unofficial].
4. **Procedural Generation (Internal):**
   - For internal blocks, apply a procedural approach based on distance from the earth-center [S2, unofficial].
   - *Note:* Internal voxels are discarded by default and calculated only when visible (e.g., when a player digs a hole) [S2, unofficial].
5. **Apply Modifications:** Overlay any stored modifications (removed/added voxels) onto the procedurally generated chunk [S2, unofficial].
6. **Serialization:** If saving is required (e.g., for the static map cache), transpose `(X, Y, Z)` to `(Z, Y, X)` and apply horizontal flips offline to avoid real-time 3D array transpositions [S1, unofficial].

**Pitfalls:**
- **Storage Growth:** If players modify the planet extensively, the size of the modification data may eventually equal the size of the original planet data, negating the storage benefits [S2, unofficial].
- **Boundary Continuity:** Failure to implement the halo trick (`+1` border) may result in geometry discontinuities at chunk edges [S1, unofficial].

## Subtopics

### Modification Storage for Procedural Worlds
A data management strategy where the world is generated on the fly using deterministic seeds and noise, storing only modifications to persist changes [S2, unofficial]. It is chosen when I/O overhead is a primary constraint and the world is largely static or only sparsely modified [S2, unofficial]. It differs from full persistence by relying on procedural regeneration for unmodified areas [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for lookup of unmodified chunks (regeneration is fast but non-zero); $O(N)$ for generating modified chunks where $N$ is the chunk volume.
- **Memory Complexity:** Proportional to the number of modifications, not the total world volume [S2, unofficial].
- **Performance:** Minimizes disk I/O by avoiding saving pure procedural data [S1, unofficial].
- **Strengths:** Efficient storage for large, mostly static worlds; fast loading of unmodified chunks [S1, S2, unofficial].
- **Limitations:** Storage cost grows with user modification; requires sophisticated culling/LOD to avoid calculating invisible internal voxels [S2, unofficial].
- **When to use:** Infinite or very large worlds with low modification density [S2, unofficial].
- **When not to use:** Worlds with high modification density where modification data exceeds procedural data size [S2, unofficial].

## Background
Originating from discussions on procedural planet rendering in voxel-based games, the concept emphasizes generating data on the fly using deterministic seeds to avoid storing uncertain or vast amounts of procedural data [S2, unofficial]. It is used in demos and implementations aiming to simulate large-scale environments (e.g., Earth-sized maps) with minimal storage footprint [S1, S2, unofficial].

## Sources
[S1] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
[S2] What kind of hardware would be required to render an Earth sized minecraft like map?, https://gamedev.stackexchange.com/questions/21951, fetched 2026-10-28 (unofficial)
