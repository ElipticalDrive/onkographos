# Froxel Grid Construction

Also called: Clustered Lighting, Froxel Grid Builder, Froxel Lighting

## Overview
Froxel Grid Construction partitions the 3D view frustum into a regular 3D grid of voxels (froxels) to spatially organize lights for efficient real-time volumetric light scattering and clustered shading. It differs from standard clustered lighting by explicitly targeting participating media, where light contributions are evaluated at froxel midpoints rather than screen-space pixels, and by excluding the sun and non-volumetric lights from the clustering process.

## How it works
**Inputs and Outputs**
*   **Input:** A list of active scene lights (point and spot) with world-space positions and radii, the current view matrix, projection matrix, view frustum bounds, and grid configuration parameters.
*   **Output:** A structured buffer of filtered lights (`LightsSB`), a froxel grid structure containing light counts and indices (`FFroxelLists`), and a shared uniform buffer (`FroxelSharedUB`) containing grid dimensions and view parameters.

**Data Structures**
The system uses a Structure of Arrays (SoA) layout for GPU efficiency.

1.  **`FFroxelUniformParameters`** (Shared Uniform Buffer)
    *   `GridSize`: `FIntVector4` (X, Y, Z dimensions, and light count or padding).
    *   `NearCm`: `float` (Near clipping plane in centimeters).
    *   `FarCm`: `float` (Far clipping plane in centimeters).
    *   `NumLights`: `int32` (Total number of valid lights).
    *   `MaxLightsPerFroxel`: `int32` (Capacity of each froxel's light list).
    *   `ViewportSize`: `FIntPoint` (Resolution in pixels).
    *   `ViewRectMinPx`: `FIntPoint` (Minimum X, Y of the view rect).
    *   `TileSizePx`: `FIntPoint` (Width and height of a froxel in pixels, derived from viewport/grid).
    *   `InvTilePx`: `float2` (Inverse of `TileSizePx`).
    *   `ViewMatrix`: `float4x4`.
    *   `ProjectionMatrix`: `float4x4`.

2.  **`FFroxelLists`** (Structured Buffers)
    *   `Counts`: `RWStructuredBuffer<uint>` of size `GridX * GridY * GridZ`. Stores the number of lights assigned to each froxel.
    *   `Offsets`: `RWStructuredBuffer<uint>` of size `GridX * GridY * GridZ`. Stores the starting index in `LightIndices` for each froxel.
    *   `TotalIndices`: `RWStructuredBuffer<uint>` of size 1. Stores the total number of light indices written.
    *   `LightIndices`: `RWStructuredBuffer<uint>` of size `GridX * GridY * GridZ * MaxLightsPerFroxel`. Stores the indices into the `LightsSB` buffer for each light assigned to a froxel.

3.  **`LightsSB`** (Structured Buffer)
    *   Stride: `sizeof(FFroxelLightData)`.
    *   Contains the filtered lights. The exact layout of `FFroxelLightData` is not fully specified in the provided notes, but it must contain at least position and radius.

**Numbered Steps**

1.  **Capture Lights (Game Thread):** Collect all relevant lights from the scene into a temporary array `FrameLights` of type `TArray<FFroxelLightData>`.
2.  **Transfer to Render Thread:** Lock `LightBufferMutex` and move `FrameLights` to `FrameLights_GT`. On the render thread, lock `LightBufferMutex` and move `FrameLights_GT` to `FrameLights_RT`, setting `NumLights_RT`.
3.  **Frustum Culling:** Iterate through `FrameLights_RT`. For each light, check if its bounding sphere intersects the view frustum:
    ```python
    if ViewFrustum.IntersectSphere(Light.PositionWS, Light.Radius):
        Add Light to FilteredLights
    ```
4.  **Create Light Buffer:** Create a GPU structured buffer `LightsSB` with stride `sizeof(FFroxelLightData)` and populate it with `FilteredLights`.
5.  **Initialize Froxel Lists:** Call `CreateFroxelLists` to allocate the `Counts`, `Offsets`, `TotalIndices`, and `LightIndices` buffers. Initialize all counts and offsets to zero.
6.  **Create Shared Uniform Buffer:** Call `CreateSharedUB` to populate `FFroxelUniformParameters` with the view matrix, projection matrix, grid dimensions (`CVarFroxelGridX/Y/Z`), far clip (`CVarFroxelFarClipCm`), max lights per froxel (`CVarFroxelMaxLightsPerFroxel`), and viewport/tile sizes.
7.  **Count Lights per Froxel:** Dispatch the `CountLightPerFroxel` compute shader. This shader maps each light to its corresponding froxel index and atomically increments the `Counts` buffer.
    *   *Note:* The specific mapping logic (linear vs logarithmic Z distribution) is controlled by `CVarFroxelZMode`.
8.  **Compute Offsets (Prefix Sum):** Dispatch a series of compute shaders (`OffsetPassA`, `OffsetPassB`, `OffsetPassC`) to perform a parallel prefix sum (scan) on the `Counts` buffer to generate the `Offsets` buffer.
    *   `OffsetPassA`: Computes Z-slice totals.
    *   `OffsetPassB`: Computes Z-slice offsets.
    *   `OffsetPassC`: Computes final froxel offsets.
9.  **Assign Lights:** Dispatch the `AssignLights` compute shader. This shader iterates through the lights, determines the froxel index, uses the `Offsets` buffer to find the write position, and atomically writes the light index into `LightIndices`.

**Parameters**
*   `CVarFroxelGridX`: Grid dimension X. Default: 16 (inferred from code snippet `16` in `TAutoConsoleVariable<int32> CVarFroxelGridX(..., 16, ...)`).
*   `CVarFroxelGridY`: Grid dimension Y. Default: 16 (inferred from code snippet `16` in `TAutoConsoleVariable<int32> CVarFroxelGridY(..., 16, ...)`).
*   `CVarFroxelGridZ`: Grid dimension Z. Default: 1.
*   `CVarFroxelMaxLightsPerFroxel`: Maximum lights allowed per froxel. Default: 32.
*   `CVarFroxelFarClipCm`: Far clipping plane in cm. Default: 100000.
*   `CVarFroxelZMode`: Z distribution mode (0: linear, 1: logarithmic). Default: 1.

**Edge Cases and Pitfalls**
*   **Precision:** The use of `float` for view/projection matrices and `int32` for indices requires careful handling of large scenes to avoid index overflow or precision loss in froxel mapping.
*   **Boundaries:** Lights exactly on the frustum boundary must be handled consistently by `IntersectSphere` to avoid flickering or double-counting.
*   **Degenerate Input:** If `NumLights` is 0, the grid construction should skip compute dispatches to avoid undefined behavior in atomic operations.
*   **Tile Size:** `TileSizePx` is clamped to a minimum of 1 to prevent division by zero or invalid shader dispatch sizes.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1-2: Data Transfer (Conceptual)
FrameLights_GT = LockAndTransfer(FrameLights)
FrameLights_RT = LockAndTransfer(FrameLights_GT)
NumLights_RT = Length(FrameLights_RT)

# Step 3: Frustum Culling
FilteredLights = []
for Light in FrameLights_RT:
    if ViewFrustum.IntersectSphere(Light.PositionWS, Light.Radius):
        FilteredLights.append(Light)

# Step 4: Create Light Buffer
LightsSB = CreateStructuredBuffer(Stride=sizeof(FFroxelLightData), Data=FilteredLights)

# Step 5: Initialize Froxel Lists
GX = CVarFroxelGridX.GetValue()
GY = CVarFroxelGridY.GetValue()
GZ = CVarFroxelGridZ.GetValue()
MaxLights = CVarFroxelMaxLightsPerFroxel.GetValue()
FroxelCount = GX * GY * GZ

Counts = CreateRWBuffer<uint>(FroxelCount, Init=0)
Offsets = CreateRWBuffer<uint>(FroxelCount, Init=0)
TotalIndices = CreateRWBuffer<uint>(1, Init=0)
LightIndices = CreateRWBuffer<uint>(FroxelCount * MaxLights, Init=0)

# Step 6: Create Shared Uniform Buffer
Params = FFroxelUniformParameters()
Params.GridSize = (GX, GY, GZ, 0)
Params.NearCm = View.NearClippingDistance
Params.FarCm = Max(1000, CVarFroxelFarClipCm.GetValue())
Params.NumLights = Length(FilteredLights)
Params.MaxLightsPerFroxel = MaxLights
Params.ViewportSize = View.UnscaledViewRect.Size()
Params.ViewRectMinPx = View.UnscaledViewRect.Min

TileSizeF = ViewportSize / (GX, GY)
Params.TileSizePx = (Floor(TileSizeF.X), Floor(TileSizeF.Y))
Params.InvTilePx = (1.0 / Params.TileSizePx.X, 1.0 / Params.TileSizePx.Y)
Params.ViewMatrix = View.ViewMatrix
Params.ProjectionMatrix = View.ProjectionMatrix

FroxelSharedUB = CreateUniformBuffer(Params)

# Step 7: Count Lights per Froxel
Dispatch(CountLightPerFroxelCS, GridSize=(GX, GY, GZ))
# Shader Logic (Conceptual):
# uint froxelIdx = GetFroxelIndex(LightPos, ViewMatrix, ProjectionMatrix, GridSize, ZMode);
# InterlockedAdd(Counts[froxelIdx], 1);

# Step 8: Compute Offsets (Prefix Sum)
# This is typically done via a multi-pass parallel scan.
# Pass A: Sum Z-slices
Dispatch(OffsetPassACS, GridSize=(GX, GY, GZ))
# Pass B: Scan Z-slice totals
Dispatch(OffsetPassBCS, GridSize=(GX, GY, GZ))
# Pass C: Final offsets
Dispatch(OffsetPassCCS, GridSize=(GX, GY, GZ))

# Step 9: Assign Lights
Dispatch(AssignLightsCS, GridSize=(GX, GY, GZ))
# Shader Logic (Conceptual):
# for each light index i:
#     froxelIdx = GetFroxelIndex(LightsSB[i].Pos, ...)
#     offset = AtomicAdd(Offsets[froxelIdx], 1) # Or use pre-computed offsets with atomic add to counts
#     LightIndices[froxelIdx * MaxLights + offset] = i
```

## Subtopics

### Froxel Light Assignment
Determines which lights contribute to which froxels using the constructed grid. It sets apart from grid construction by focusing on the actual assignment logic and atomic writes, whereas grid construction focuses on the spatial partitioning and buffer initialization. Choose it when you need to map specific lights to spatial regions. [Froxel Light Assignment](13184-froxel-light-assignment.md)

## Operating characteristics
*   **Time Complexity:** $O(N + F)$, where $N$ is the number of lights and $F$ is the number of froxels. The culling and counting steps are linear in $N$, and the prefix sum and assignment steps are linear in $F$ and $N$ respectively.
*   **Memory Complexity:** $O(F \cdot M + N)$, where $M$ is `MaxLightsPerFroxel`. The buffers scale with the grid size and light capacity.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with grid resolution. Higher grid dimensions increase memory usage and prefix sum cost but improve light culling efficiency.
*   **Strengths:** Efficient spatial culling of lights for volumetric rendering; reduces overdraw by skipping lights that do not intersect the view frustum or froxels.
*   **Limitations:** Requires careful tuning of grid dimensions and max lights per froxel to balance memory and performance. Logarithmic Z distribution may be necessary for scenes with large depth ranges.
*   **When to use:** Real-time volumetric light scattering with many local lights.
*   **When not to use:** Scenes with very few lights or where the sun is the primary light source (as the sun is excluded).

## Background
The method originates from the "UE5 Froxel Lighting Plugin" by Armin Tajik, an unofficial community plugin for Unreal Engine 5. It implements a real-time clustered lighting and volumetrics system, adapting clustered lighting techniques to the specific needs of volumetric fog rendering in UE5. The plugin integrates with UE5's Render Graph API and uses C++20 and HLSL.

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
