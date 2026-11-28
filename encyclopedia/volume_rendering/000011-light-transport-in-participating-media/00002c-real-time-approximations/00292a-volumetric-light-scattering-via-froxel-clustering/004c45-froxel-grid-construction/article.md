# Froxel Grid Construction

Also called: Clustered Lighting, Froxel Grid Builder, Froxel Lighting

## Overview
Froxel Grid Construction partitions the 3D view frustum into a regular 3D grid of voxels (froxels) to spatially organize lights for efficient real-time volumetric light scattering and clustered shading [S1, unofficial]. It differs from standard clustered lighting by explicitly targeting participating media, where light contributions are evaluated at froxel midpoints rather than screen-space pixels, and by excluding the sun and non-volumetric lights from the clustering process [G].

## How it works
**Inputs and Outputs**
*   **Input:** A list of active scene lights (point and spot) with world-space positions and radii, the current view matrix, projection matrix, view frustum bounds, and grid configuration parameters [S1, unofficial].
*   **Output:** A structured buffer of filtered lights (`LightsSB`), a froxel grid structure containing light counts and indices (`FFroxelLists`), and a shared uniform buffer (`FroxelSharedUB`) containing grid dimensions and view parameters [S1, unofficial].

**Data Structures**
The system uses a Structure of Arrays (SoA) layout for GPU efficiency [S1, unofficial].

1.  **`FFroxelUniformParameters`** (Shared Uniform Buffer)
    *   `GridSize`: `FIntVector4` (X, Y, Z dimensions, and padding) [S1, unofficial].
    *   `NearCm`: `float` (Near clipping plane in centimeters) [S1, unofficial].
    *   `FarCm`: `float` (Far clipping plane in centimeters) [S1, unofficial].
    *   `NumLights`: `int32` (Total number of valid lights) [S1, unofficial].
    *   `MaxLightsPerFroxel`: `int32` (Capacity of each froxel's light list) [S1, unofficial].
    *   `ViewportSize`: `FIntPoint` (Resolution in pixels) [S1, unofficial].
    *   `ViewRectMinPx`: `FIntPoint` (Minimum X, Y of the view rect) [S1, unofficial].
    *   `TileSizePx`: `FIntPoint` (Width and height of a froxel in pixels, derived from viewport/grid) [S1, unofficial].
    *   `InvTilePx`: `float2` (Inverse of `TileSizePx`) [S1, unofficial].
    *   `ViewMatrix`: `float4x4` [S1, unofficial].
    *   `ProjectionMatrix`: `float4x4` [S1, unofficial].

2.  **`FFroxelLists`** (Structured Buffers)
    *   `Counts`: `RWStructuredBuffer<uint>` of size `GridX * GridY * GridZ`. Stores the number of lights assigned to each froxel [S1, unofficial].
    *   `Offsets`: `RWStructuredBuffer<uint>` of size `GridX * GridY * GridZ`. Stores the starting index in `LightIndices` for each froxel [S1, unofficial].
    *   `TotalIndices`: `RWStructuredBuffer<uint>` of size 1. Stores the total number of light indices written [S1, unofficial].
    *   `LightIndices`: `RWStructuredBuffer<uint>` of size `GridX * GridY * GridZ * MaxLightsPerFroxel`. Stores the indices into the `LightsSB` buffer for each light assigned to a froxel [S1, unofficial].

3.  **`LightsSB`** (Structured Buffer)
    *   Stride: `sizeof(FFroxelLightData)` [S1, unofficial].
    *   Contains the filtered lights. The exact layout of `FFroxelLightData` is not fully specified in the provided notes, but it must contain at least position and radius [S1, unofficial].

**Numbered Steps**

1.  **Capture Lights (Game Thread):** Collect all relevant lights from the scene into a temporary array `FrameLights` of type `TArray<FFroxelLightData>` [S1, unofficial].
2.  **Transfer to Render Thread:** Lock `LightBufferMutex` and move `FrameLights` to `FrameLights_GT` [S1, unofficial].
3.  **Render Thread Transfer:** On the render thread, lock `LightBufferMutex` and move `FrameLights_GT` to `FrameLights_RT`, setting `NumLights_RT` [S1, unofficial].
4.  **Frustum Culling:** Iterate through `FrameLights_RT`. For each light, check if its bounding sphere intersects the view frustum: `InView.ViewFrustum.IntersectSphere(FVector(L.PositionWS), L.Radius)` [S1, unofficial].
5.  **Create Light Buffer:** Create a GPU structured buffer `LightsSB` with stride `sizeof(FFroxelLightData)` and populate it with the filtered lights [S1, unofficial].
6.  **Initialize Froxel Lists:** Call `CreateFroxelLists` to allocate the `Counts`, `Offsets`, `TotalIndices`, and `LightIndices` buffers [S1, unofficial]. Initialize all counts and offsets to zero [G].
7.  **Create Shared Uniform Buffer:** Call `CreateSharedUB` to populate `FFroxelUniformParameters` with the view matrix, projection matrix, grid dimensions (`CVarFroxelGridX/Y/Z`), far clip (`CVarFroxelFarClipCm`), max lights per froxel (`CVarFroxelMaxLightsPerFroxel`), and viewport/tile sizes [S1, unofficial].
8.  **Count Lights per Froxel:** Dispatch the `CountLightPerFroxel` compute shader to count lights per froxel [S1, unofficial].

**Parameters**
*   `CVarFroxelGridX`: Grid dimension X. Default: 16 [S1, unofficial].
*   `CVarFroxelGridY`: Grid dimension Y. Default: 16 [S1, unofficial].
*   `CVarFroxelGridZ`: Grid dimension Z. Default: 1 [S1, unofficial].
*   `CVarFroxelMaxLightsPerFroxel`: Maximum lights allowed per froxel. Default: 32 [S1, unofficial].
*   `CVarFroxelFarClipCm`: Far clipping plane in cm. Default: 100000 [S1, unofficial].
*   `CVarFroxelZMode`: Z distribution mode (0: linear, 1: logarithmic). Default: 1 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** The use of `float` for view/projection matrices and `int32` for indices requires careful handling of large scenes to avoid index overflow or precision loss in froxel mapping [G].
*   **Boundaries:** Lights exactly on the frustum boundary must be handled consistently by `IntersectSphere` to avoid flickering or double-counting [G].
*   **Degenerate Input:** If `NumLights` is 0, the grid construction should skip compute dispatches to avoid undefined behavior in atomic operations [G].
*   **Tile Size:** `TileSizePx` is clamped to a minimum of 1 to prevent division by zero or invalid shader dispatch sizes [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1-3: Data Transfer
FrameLights_GT = []
with LightBufferMutex:
    FrameLights_GT = list(FrameLights)
FrameLights_RT = []
with LightBufferMutex:
    FrameLights_RT = list(FrameLights_GT)
NumLights_RT = len(FrameLights_RT)

# Step 4: Frustum Culling
FilteredLights = []
for Light in FrameLights_RT:
    if ViewFrustum.IntersectSphere(Light.PositionWS, Light.Radius):
        FilteredLights.append(Light)

# Step 5: Create Light Buffer
LightsSB = StructuredBuffer(data=FilteredLights, stride=sizeof(FFroxelLightData))

# Step 6: Initialize Froxel Lists
GX = CVarFroxelGridX.GetValue()
GY = CVarFroxelGridY.GetValue()
GZ = CVarFroxelGridZ.GetValue()
MaxLights = CVarFroxelMaxLightsPerFroxel.GetValue()
FroxelCount = GX * GY * GZ

Counts = RWBuffer(size=FroxelCount, init=0)
Offsets = RWBuffer(size=FroxelCount, init=0)
TotalIndices = RWBuffer(size=1, init=0)
LightIndices = RWBuffer(size=FroxelCount * MaxLights, init=0)

# Step 7: Create Shared Uniform Buffer
Params = UniformBuffer()
Params.GridSize = (GX, GY, GZ, 0)
Params.NearCm = View.NearClippingDistance
Params.FarCm = max(1000, CVarFroxelFarClipCm.GetValue())
Params.NumLights = len(FilteredLights)
Params.MaxLightsPerFroxel = max(1, MaxLights)
Params.ViewportSize = View.UnscaledViewRect.Size()
Params.ViewRectMinPx = View.UnscaledViewRect.Min

TileSizeF_X = ViewportSize.X / GX
TileSizeF_Y = ViewportSize.Y / GY
Params.TileSizePx = (floor(max(TileSizeF_X, 1.0)), floor(max(TileSizeF_Y, 1.0)))
Params.InvTilePx = (1.0 / Params.TileSizePx.X, 1.0 / Params.TileSizePx.Y)
Params.ViewMatrix = View.ViewMatrix
Params.ProjectionMatrix = View.ProjectionMatrix

FroxelSharedUB = UniformBuffer(params=Params)

# Step 8: Count Lights per Froxel
Dispatch(CountLightPerFroxelCS, thread_groups=(GX, GY, GZ))
# The sources do not give the exact shader implementation for light-to-froxel mapping or atomic counting.
```

## Subtopics

### Froxel Light Assignment
Determines which lights contribute to which froxels using the constructed grid. It sets apart from grid construction by focusing on the actual assignment logic and atomic writes, whereas grid construction focuses on the spatial partitioning and buffer initialization. Choose it when you need to map specific lights to spatial regions. [Froxel Light Assignment](13184-froxel-light-assignment.md)

## Operating characteristics
*   **Time Complexity:** $O(N + F)$, where $N$ is the number of lights and $F$ is the number of froxels [G].
*   **Memory Complexity:** $O(F \cdot M + N)$, where $M$ is `MaxLightsPerFroxel` [G].
*   **Measured Performance:** Not given in the sources [G].
*   **Scaling:** Scales with grid resolution [G].
*   **Strengths:** Efficient spatial culling of lights for volumetric rendering [G].
*   **Limitations:** Requires careful tuning of grid dimensions and max lights per froxel to balance memory and performance [G].
*   **When to use:** Real-time volumetric light scattering with many local lights [G].
*   **When not to use:** Scenes with very few lights or where the sun is the primary light source [G].

## Background
The method originates from the "UE5 Froxel Lighting Plugin" by Armin Tajik, an unofficial community plugin for Unreal Engine 5 [S1, unofficial]. It implements a real-time clustered lighting and volumetrics system, adapting clustered lighting techniques to the specific needs of volumetric fog rendering in UE5 [S1, unofficial]. The plugin integrates with UE5's Render Graph API and uses C++20 and HLSL [S1, unofficial].

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
