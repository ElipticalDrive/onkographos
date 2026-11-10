# Voxel Grid Generation & Expansion

Also called: Occupancy Grid Expansion, Smoke Voxelization

## Overview
This method generates a 3D occupancy grid representing a volumetric smoke cloud by iteratively expanding a spherical radius from a central origin point over time. It solves the problem of creating a discrete voxel representation of a continuous smoke volume for GPU-based ray marching, distinguishing itself from particle-based splatting by using a simple radial distance check against an easing curve to fill the grid [S1, unofficial]. Unlike siblings that load pre-existing scene data or generate synthetic medical CTs, this method procedurally constructs the volume geometry from scratch based on temporal expansion parameters [S1, unofficial].

## How it works
**Inputs:**
- `boundsExtent`: `Vector3` defining the half-size of the volume bounds (Default: `(3, 3, 3)`).
- `voxelSize`: `float` defining the side length of a single voxel (Default: `0.5`).
- `smokeRadius`: `float` defining the maximum expansion radius (Default: `4.0`).
- `smokeExpansionTime`: `float` defining the duration to reach `smokeRadius`.
- `smokeOrigin`: `Vector3` defining the center point of the volume expansion.
- `smokeTimer`: `float` current elapsed time since expansion start.
- `easingCurve`: A function or curve object that maps a normalized time value `[0, 1]` to a scalar multiplier.

**Outputs:**
- `smokeData`: A 1D array of integers (`int`) representing the occupancy grid.
- `ComputeBuffer`: A GPU buffer (`StructuredBuffer<int>`) containing the grid data.

**Data Structures:**
- **Voxel Grid Layout**: Linear 1D array of size `totalVoxels`.
- **Indexing**: Linear index `i` for voxel at `(x, y, z)` is calculated as:
  $$ i = x + y \cdot \text{voxelsX} + z \cdot \text{voxelsX} \cdot \text{voxelsY} $$
- **Buffer Type**: `int` (4 bytes per element), initialized to `0` (empty) or `1` (occupied).

**Steps:**
1. **Calculate Grid Dimensions**:
   Compute the number of voxels along each axis based on the bounds and voxel size:
   $$ \text{voxelsX} = \lceil \frac{\text{boundsExtent}.x \cdot 2}{\text{voxelSize}} \rceil $$
   $$ \text{voxelsY} = \lceil \frac{\text{boundsExtent}.y \cdot 2}{\text{voxelSize}} \rceil $$
   $$ \text{voxelsZ} = \lceil \frac{\text{boundsExtent}.z \cdot 2}{\text{voxelSize}} \rceil $$
   $$ \text{totalVoxels} = \text{voxelsX} \cdot \text{voxelsY} \cdot \text{voxelsZ} $$
   *Note: The source notes imply `boundsSize` is derived from `boundsExtent` (likely `2 * boundsExtent` or the extent itself represents the full size depending on implementation context, but standard bounding box logic uses half-extents. The source code snippet `float3(x * voxelSize, ...)` suggests `x` is an index. The formula `ceil(boundsSize.x / voxelSize)` is used in notes. We assume `boundsSize` corresponds to the full dimensions of the box defined by `boundsExtent`.*

2. **Initialize Buffer**:
   Allocate a `ComputeBuffer` of size `totalVoxels * 4` bytes. Initialize the backing array `smokeData` with zeros.

3. **Update Loop (Every 30 Frames)**:
   To mitigate CPU stutter, the grid is not updated every frame. Instead, it is cleared and recomputed every 30 frames [S1, unofficial].
   - Clear `smokeData` to all zeros.
   - Calculate normalized time progress:
     $$ \text{normalValue} = \text{inlerp}(0, \text{smokeExpansionTime}, \text{smokeTimer}) $$
     *Note: `inlerp` clamps the value to `[0, 1]`.*
   - Calculate current expansion radius:
     $$ \text{currentSmokeRadius} = \text{easingCurve.Evaluate}(\text{normalValue}) \cdot \text{smokeRadius} $$
   - Iterate through all voxels:
     For `x` from `0` to `voxelsX - 1`:
       For `y` from `0` to `voxelsY - 1`:
         For `z` from `0` to `voxelsZ - 1`:
           1. Compute voxel world position relative to the bounds center. The source uses:
              $$ \text{voxelPos} = \text{float3}(x \cdot \text{voxelSize}, y \cdot \text{voxelSize}, z \cdot \text{voxelSize}) - \text{float3}(\text{boundsExtent}.x, 0, \text{boundsExtent}.z) $$
              *Note: This specific offset formula appears asymmetric in the source (`boundsExtent.x` and `boundsExtent.z` are subtracted, `y` is not). This may be a specific implementation detail or bug in the reference code, but is reproduced as stated.*
           2. Compute distance to origin:
              $$ \text{distance} = \text{length}(\text{voxelPos} - \text{smokeOrigin}) $$
           3. Check occupancy:
              If $\text{distance} \le \text{currentSmokeRadius}$, set:
              $$ \text{smokeData}[x + y \cdot \text{voxelsX} + z \cdot \text{voxelsX} \cdot \text{voxelsY}] = 1 $$
              Else, set to `0`.

4. **Upload to GPU**:
   Transfer the `smokeData` array to the GPU `ComputeBuffer` using `smokeBuffer.SetData(smokeData)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_grid(bounds_extent, voxel_size, smoke_radius, smoke_expansion_time, smoke_origin, smoke_timer, easing_curve):
    # 1. Calculate dimensions
    # Assuming bounds_extent is half-size, full size is 2 * bounds_extent
    bounds_size_x = 2 * bounds_extent.x
    bounds_size_y = 2 * bounds_extent.y
    bounds_size_z = 2 * bounds_extent.z
    
    voxels_x = math.ceil(bounds_size_x / voxel_size)
    voxels_y = math.ceil(bounds_size_y / voxel_size)
    voxels_z = math.ceil(bounds_size_z / voxel_size)
    total_voxels = voxels_x * voxels_y * voxels_z
    
    # 2. Initialize data array
    smoke_data = [0] * total_voxels
    
    # 3. Calculate current radius
    # inlerp clamps value between 0 and 1
    normal_value = min(1.0, max(0.0, smoke_timer / smoke_expansion_time))
    current_smoke_radius = easing_curve.evaluate(normal_value) * smoke_radius
    
    # 4. Iterate and fill
    for x in range(voxels_x):
        for y in range(voxels_y):
            for z in range(voxels_z):
                # Compute voxel position
                # Note: Source formula is asymmetric: subtracts bounds_extent.x and .z, but not y
                voxel_pos_x = x * voxel_size - bounds_extent.x
                voxel_pos_y = y * voxel_size # Source does not subtract bounds_extent.y
                voxel_pos_z = z * voxel_size - bounds_extent.z
                
                voxel_pos = Vector3(voxel_pos_x, voxel_pos_y, voxel_pos_z)
                
                # Distance to origin
                distance = math.sqrt(
                    (voxel_pos.x - smoke_origin.x)**2 +
                    (voxel_pos.y - smoke_origin.y)**2 +
                    (voxel_pos.z - smoke_origin.z)**2
                )
                
                # Set occupancy
                index = x + y * voxels_x + z * voxels_x * voxels_y
                if distance <= current_smoke_radius:
                    smoke_data[index] = 1
                else:
                    smoke_data[index] = 0
                    
    return smoke_data, (voxels_x, voxels_y, voxels_z)
```

## Subtopics

### [Point Wisp Voxelization](10591-point-wisp-voxelization.md)
A procedural method for generating volumetric smoke data by distributing points around a "guide particle" using fractal noise and baking them into a voxel grid. It distinguishes itself by using correlated random walks and radial displacement to form cohesive "wisps" rather than isolated blobs, suitable for organic smoke shapes compared to the simple spherical expansion of this method.

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the total number of voxels in the grid. The iteration covers the entire bounding box volume regardless of actual smoke density.
- **Memory Complexity**: $O(N)$ integers. For a $6 \times 6 \times 6$ grid (default bounds 3,3,3 with voxel size 0.5), $N = 12 \cdot 12 \cdot 12 = 1728$ voxels, requiring $1728 \cdot 4$ bytes $\approx 6.75$ KB.
- **Performance**: The method runs on the CPU. Updates are throttled to every 30 frames to reduce overhead [S1, unofficial]. This causes potential stutter or delay in volume expansion responsiveness.
- **Strengths**: Simple to implement; low memory footprint for small volumes; provides a clean occupancy mask for GPU ray marching.
- **Limitations**: The spherical expansion creates a "smoke ball" rather than a complex volume shape [S1, unofficial]. The asymmetric bounding box offset in the source implementation may cause misalignment. No upsampling filter is applied during rendering, leading to blocky edges under motion.

## Background
This method originates from a 2024 Unity-based implementation by Schokolasbotter, designed to render volumetric smoke entirely on the GPU using a voxel grid driven by a two-stage ray march [S1, unofficial]. It was developed as an alternative to particle systems and billboard geometry, aiming for analytical per-pixel evaluation. The approach is part of a pipeline that includes noise generation and ray marching shaders.

## Sources
[S1] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
