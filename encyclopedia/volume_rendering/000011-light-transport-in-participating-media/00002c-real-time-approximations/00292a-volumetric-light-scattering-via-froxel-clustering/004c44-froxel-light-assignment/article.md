# Froxel Light Assignment

Also called: Froxelization, Light-to-Froxel Mapping

## Overview
Froxel Light Assignment is the core data-structure preparation step in volumetric light scattering via froxel clustering. It maps scene lights to a 3D grid of frustum voxels (froxels) by determining which froxels each light affects, counting these associations, and computing compact offsets to store variable-length light lists in a global buffer. It differs from [Froxel Grid Construction](13185-froxel-grid-construction.md) by focusing on the spatial assignment logic and buffer indexing rather than the geometric partitioning of the camera frustum.

## How it works
**Inputs:**
- List of lights visible in the current view frustum, filtered by sphere intersection with the view frustum [S1, unofficial].
- Froxel grid dimensions: $G_X, G_Y, G_Z$ (default 16x16x16) [S1, unofficial].
- Maximum lights per froxel: $N_{max}$ (default 32) [S1, unofficial].
- Camera parameters: View Matrix, Projection Matrix, Near/Far clipping planes in cm [S1, unofficial].

**Outputs:**
- `FroxelLists`: A structured buffer containing:
  - `Offsets`: `uint` array of size $G_X \times G_Y \times G_Z$, storing the starting index in `LightIndices` for each froxel [S1, unofficial].
  - `Counts`: `uint` array of size $G_X \times G_Y \times G_Z$, storing the number of lights assigned to each froxel [S1, unofficial].
  - `TotalIndices`: `uint`, the total number of light assignments [S1, unofficial].
  - `LightIndices`: `uint` array of size $G_X \times G_Y \times G_Z \times N_{max}$ (or dynamically sized based on total assignments), storing the indices of lights assigned to each froxel [S1, unofficial].
  - `FroxelCount`: `uint32`, total number of froxels [S1, unofficial].

**Data Structures:**
- `FFroxelUniformParameters`: Uniform buffer containing grid size ($G_X, G_Y, G_Z, G_X \times G_Y \times G_Z$), viewport/tile sizes, inverse tile pixels, View/Projection matrices, Near/Far clipping distances, number of lights, and max lights per froxel [S1, unofficial].
- `FFroxelLightData`: Struct representing light data (position, radius, etc.) used for frustum intersection tests [S1, unofficial].

**Steps:**

1. **Light Filtering:**
   - Iterate through all candidate lights.
   - For each light, check if its bounding sphere intersects the camera view frustum.
   - Store only intersecting lights in a temporary list `LightsInView` [S1, unofficial].
   - Create a Structured Buffer (`LightsSB`) from `LightsInView` for GPU access [S1, unofficial].

2. **Light Counting (Per-Froxel):**
   - Initialize a UAV (`FroxelLightCountsUAV`) for `FroxelLists.Counts` to zero [S1, unofficial].
   - Execute a compute shader (`FFroxelCountCS`) with group count $\lceil N_{lights} / 64 \rceil$ [S1, unofficial].
   - For each light thread:
     - Determine which froxels the light affects. This involves transforming the light's position and radius into froxel space and checking overlap with froxel bounds [S1, unofficial].
     - Use atomic operations to increment the count for each affected froxel in `FroxelLightCountsUAV` [S1, unofficial].

3. **Offset Computation (Blelloch Scan):**
   - Compute offsets into the global light index list using a 3-pass Blelloch scan approach [S1, unofficial].
   - **Pass A:**
     - Clear `ZSliceTotalsUAV` and `FroxelLightOffsetsUAV` to 0 [S1, unofficial].
     - Execute `FFroxelOffsetPassACS` with group count $\lceil Z_{slice\_size} / 256 \rceil$, where $Z_{slice\_size} = G_X \times G_Y$ [S1, unofficial].
     - This pass likely performs a parallel prefix sum within each Z-slice to compute local offsets and sums [S1, unofficial].
   - **Pass B:**
     - Clear `ZSliceOffsetsUAV` to 0 [S1, unofficial].
     - Execute `FFroxelOffsetPassBCS` with group count $\lceil G_Z / 1024 \rceil$ [S1, unofficial].
     - This pass performs a parallel prefix sum across the Z-slice totals to compute the base offset for each Z-slice [S1, unofficial].
   - **Pass C:**
     - Execute `FFroxelOffsetPassCCS` with group count $\lceil Z_{slice\_size} / 1024 \rceil$ [S1, unofficial].
     - This pass adds the Z-slice base offsets to the local froxel offsets to get absolute indices [S1, unofficial].
   - Store results in `FroxelLightOffsetsUAV` [S1, unofficial].

4. **Light Assignment (Reduce and Scatter):**
   - Execute a reduce pass to compute the total number of light assignments (`TotalIndices`) [S1, unofficial].
   - Execute a compute shader to assign light indices to froxels:
     - For each light, determine the froxels it affects.
     - Use atomic operations to write the light index into the `LightIndices` buffer at the position indicated by the froxel's current offset and count [S1, unofficial].
     - Increment the froxel's count atomically [S1, unofficial].

**Parameters:**
- `r.Froxel.GridX`: Froxel grid dimension X. Default: 16 [S1, unofficial].
- `r.Froxel.GridY`: Froxel grid dimension Y. Default: 16 [S1, unofficial].
- `r.Froxel.GridZ`: Froxel grid dimension Z. Default: 16 [S1, unofficial].
- `r.Froxel.MaxLightsPerFroxel`: Maximum number of lights that can be assigned to a single froxel. Default: 32 [S1, unofficial].
- `r.Froxel.FarClipCm`: Froxel grid far clipping plane in cm. Default: 100000 [S1, unofficial].
- `r.Froxel.ZMode`: Froxel grid Z distribution mode. 0: linear, 1: logarithmic. Default: 1 [S1, unofficial].
- `r.Froxel.Visualize`: Visualize Froxel grid. 0: off, 1: heatmap, 2: hash pattern. Default: 0 [S1, unofficial].
- `r.Froxel.DebugOpacity`: Overlay opacity when compositing on SceneColor. Default: 0.5f [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Light Frustum Culling:** Lights outside the view frustum must be filtered out to avoid incorrect assignments [S1, unofficial].
- **Atomic Contention:** Multiple lights may assign to the same froxel, requiring atomic operations for count and index updates [S1, unofficial].
- **Buffer Overflow:** If a froxel exceeds `MaxLightsPerFroxel`, assignments may be dropped or cause buffer overflows if not handled [S1, unofficial].
- **Precision:** Transformations between world space and froxel space must maintain sufficient precision to avoid misclassification of lights near froxel boundaries [S1, unofficial].

## Subtopics

### Froxel Offset Computation
Computes the offsets into the global light index list for each froxel using a 3-pass Blelloch scan approach. Results are stored in `FroxelLightOffsetsUAV` [S1, unofficial]. Pass A clears `ZSliceTotalsUAV` and `FroxelLightOffsetsUAV` and executes `FFroxelOffsetPassACS` with group count $\lceil Z_{slice\_size} / 256 \rceil$, where $Z_{slice\_size} = G_X \times G_Y$ [S1, unofficial]. Pass B clears `ZSliceOffsetsUAV` and executes `FFroxelOffsetPassBCS` with group count $\lceil G_Z / 1024 \rceil$ [S1, unofficial]. Pass C executes `FFroxelOffsetPassCCS` with group count $\lceil Z_{slice\_size} / 1024 \rceil$ to add z-slice group sums to each froxel [S1, unofficial]. Data structures include `ZSliceTotals` (Structured buffer of size $G_Z$, type `uint32`) and `ZSliceOffsets` (Structured buffer of size $G_Z$, type `uint32`) [S1, unofficial].

### Froxel Light Counting
A compute shader pass that counts the number of lights affecting each froxel in the grid. The result is stored in the `FroxelLightCountsUAV` buffer [S1, unofficial]. It creates a UAV for `FroxelLists.Counts` and clears it to 0 [S1, unofficial]. It allocates parameters for `FFroxelCountCS`, binding `SharedUB`, `View` Uniform Buffer, `Lights` SRV, and `FroxelLightCounts` UAV [S1, unofficial]. It executes the compute shader with group count calculated as $\lceil NumLights_{RT} / 64 \rceil$ [S1, unofficial]. The parameter structure `FFroxelCountCS::FParameters` contains `FroxelSharedUB` (struct ref), `View` (struct ref), `Lights` (SRV: `StructuredBuffer<FFroxelLightData>`), and `FroxelLightCounts` (UAV: `RWStructuredBuffer<uint>`) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{lights} \times N_{froxels\_affected})$ for counting and assignment, where $N_{froxels\_affected}$ is the average number of froxels a light intersects. The Blelloch scan is $O(G_Z)$ for Pass B and $O(G_X \times G_Y)$ for Passes A and C.
- **Memory Complexity:** $O(G_X \times G_Y \times G_Z \times N_{max})$ for the `LightIndices` buffer, plus $O(G_X \times G_Y \times G_Z)$ for `Offsets` and `Counts` [S1, unofficial].
- **Performance:** Depends heavily on the number of lights and their spatial distribution. Dense light clusters may cause atomic contention.
- **Strengths:** Efficiently maps lights to volume regions, enabling localized lighting calculations.
- **Limitations:** Requires careful tuning of grid resolution and max lights per froxel to balance memory usage and accuracy.
- **When to use:** Real-time volumetric lighting with local lights.
- **When not to use:** Scenes with very few lights or where global illumination is handled differently.

## Background
Originated in Unreal Engine 5 as part of the Froxel Lighting system, implemented by Armin Tajik [S1, unofficial]. It is used in real-time rendering of participating media, specifically for local lights within fog volumes.

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
