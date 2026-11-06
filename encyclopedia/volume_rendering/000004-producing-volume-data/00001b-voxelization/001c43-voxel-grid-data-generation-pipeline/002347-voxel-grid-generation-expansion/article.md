# Voxel Grid Generation & Expansion

Also called: Occupancy Grid Expansion, Smoke Voxelization

## Overview
This method generates a 3D occupancy grid representing a volumetric smoke cloud by iteratively expanding a spherical radius from a central origin point over time. It solves the problem of creating a discrete voxel representation for a continuous, animated volume without using particle systems or billboard geometry. It differs from sibling methods like the DCX Pipeline by using a simple CPU-driven sphere expansion logic rather than complex mesh-based voxelization, and from Scene Data Loading by actively generating new data rather than resizing existing assets. The resulting grid is uploaded to a GPU `ComputeBuffer` for subsequent ray marching.

## How it works
**Inputs:**
- `boundsExtent`: `Vector3`, half-size of the volume bounds. Default: `(3, 3, 3)` [S1, unofficial].
- `voxelSize`: `float`, grid resolution. Default: `0.5` [S1, unofficial].
- `smokeRadius`: `float`, maximum expansion radius. Default: `4` [S1, unofficial].
- `smokeExpansionTime`: `float`, time required to reach maximum radius [S1, unofficial].
- `smokeOrigin`: `Vector3`, center point of volume expansion [S1, unofficial].
- `smokeTimer`: `float`, current elapsed time since expansion start [S1, unofficial].
- `easingCurve`: Function mapping time to normalized expansion factor [S1, unofficial].

**Outputs:**
- `ComputeBuffer` named `_SmokeVoxels`, typed as `StructuredBuffer<int>` in the shader. Size is `totalVoxels * 4` bytes. Linear memory order [S1, unofficial].

**Data Structures:**
- `smokeData`: Array of integers of size `totalVoxels`. Initialized to 0. A value of 1 indicates an occupied voxel; 0 indicates empty [S1, unofficial].

**Steps:**
1.  **Calculate Grid Dimensions:**
    Compute the number of voxels along each axis using ceiling division:
    $$
    \text{voxelsX} = \lceil \frac{\text{boundsSize}.x}{\text{voxelSize}} \rceil
    $$
    $$
    \text{voxelsY} = \lceil \frac{\text{boundsSize}.y}{\text{voxelSize}} \rceil
    $$
    $$
    \text{voxelsZ} = \lceil \frac{\text{boundsSize}.z}{\text{voxelSize}} \rceil
    $$
    $$
    \text{totalVoxels} = \text{voxelsX} \times \text{voxelsY} \times \text{voxelsZ}
    $$
    [S1, unofficial]

2.  **Initialize Buffer:**
    Allocate a `ComputeBuffer` of size `totalVoxels` with integer type (4 bytes each). Fill with 0 [S1, unofficial].

3.  **Clear Buffer (Periodic):**
    On every update cycle (configured to every 30 frames in the source), clear the `smokeData` array to 0 [S1, unofficial].

4.  **Iterate Voxels:**
    Loop through all voxel coordinates $x, y, z$ where:
    $$
    x \in [0, \text{voxelsX}), \quad y \in [0, \text{voxelsY}), \quad z \in [0, \text{voxelsZ})
    $$
    [S1, unofficial]

5.  **Compute Voxel World Position:**
    Calculate the world position of the voxel center. Note the coordinate system assumption: the bounds extent is subtracted from the raw grid position, implying the origin is centered or offset by half-bounds [S1, unofficial]:
    $$
    \text{voxelPos} = \text{float3}(x \cdot \text{voxelSize}, y \cdot \text{voxelSize}, z \cdot \text{voxelSize}) - \text{float3}(\text{boundsExtent}.x, 0, \text{boundsExtent}.z)
    $$
    [S1, unofficial]

6.  **Compute Distance to Origin:**
    $$
    \text{distance} = \text{length}(\text{voxelPos} - \text{smokeOrigin})
    $$
    [S1, unofficial]

7.  **Compute Current Expansion Radius:**
    Determine the current radius based on time and easing:
    $$
    \text{normalValue} = \text{inlerp}(0, \text{smokeExpansionTime}, \text{smokeTimer})
    $$
    $$
    \text{currentSmokeRadius} = \text{easingCurve.Evaluate}(\text{normalValue}) \cdot \text{smokeRadius}
    $$
    [S1, unofficial]

8.  **Set Occupancy:**
    If the voxel is within the current expansion radius, mark it as occupied:
    $$
    \text{index} = x + y \cdot \text{voxelsX} + z \cdot \text{voxelsX} \cdot \text{voxelsY}
    $$
    $$
    \text{if } \text{distance} \le \text{currentSmokeRadius}: \quad \text{smokeData}[\text{index}] = 1
    $$
    [S1, unofficial]

9.  **Upload to GPU:**
    Transfer the `smokeData` array to the GPU buffer:
    $$
    \text{smokeBuffer.SetData}(\text{smokeData})
    $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_grid(bounds_extent, voxel_size, smoke_radius, smoke_expansion_time, smoke_origin, smoke_timer, easing_curve):
    # 1. Calculate dimensions
    bounds_size = bounds_extent * 2.0  # Assuming bounds_extent is half-size
    voxels_x = ceil(bounds_size.x / voxel_size)
    voxels_y = ceil(bounds_size.y / voxel_size)
    voxels_z = ceil(bounds_size.z / voxel_size)
    total_voxels = voxels_x * voxels_y * voxels_z

    # 2. Initialize data array
    smoke_data = [0] * total_voxels

    # 3. Compute current radius
    # inlerp is linear interpolation
    normal_value = smoke_timer / smoke_expansion_time if smoke_expansion_time > 0 else 1.0
    # Clamp normal_value to [0, 1]
    normal_value = max(0.0, min(1.0, normal_value))
    current_smoke_radius = easing_curve.evaluate(normal_value) * smoke_radius

    # 4. Iterate and fill
    for x in range(voxels_x):
        for y in range(voxels_y):
            for z in range(voxels_z):
                # 5. Compute voxel world position
                # Note: Source subtracts bounds_extent.x and z, but 0 for y.
                # This implies a specific coordinate alignment.
                voxel_pos_x = x * voxel_size - bounds_extent.x
                voxel_pos_y = y * voxel_size - 0.0
                voxel_pos_z = z * voxel_size - bounds_extent.z
                voxel_pos = (voxel_pos_x, voxel_pos_y, voxel_pos_z)

                # 6. Distance
                distance = sqrt(voxel_pos.x**2 + voxel_pos.y**2 + voxel_pos.z**2) # Assuming smoke_origin is (0,0,0) relative to this calc or handled by offset
                # Correction: Source says length(voxelPos - smokeOrigin). 
                # If smokeOrigin is not (0,0,0), adjust:
                dist_vec = (voxel_pos_x - smoke_origin.x, voxel_pos_y - smoke_origin.y, voxel_pos_z - smoke_origin.z)
                distance = sqrt(dist_vec.x**2 + dist_vec.y**2 + dist_vec.z**2)

                # 8. Set occupancy
                if distance <= current_smoke_radius:
                    index = x + y * voxels_x + z * voxels_x * voxels_y
                    smoke_data[index] = 1

    return smoke_data, total_voxels
```
[G] The source code implies `smokeOrigin` is used in the distance calculation. If `smokeOrigin` is not the world origin, the `voxelPos` calculation must account for it. The pseudocode above assumes `smokeOrigin` is subtracted from `voxelPos`. The source snippet `voxelPos = ... - float3(boundsExtent.x, 0, boundsExtent.z)` suggests the grid is centered or offset relative to the bounds, and `smokeOrigin` is likely defined in that same space.

**Edge Cases and Pitfalls:**
- **Stutter:** Buffer updates run on the CPU every 30 frames, causing potential stutter or delay in volume expansion [S1, unofficial].
- **Coordinate System:** The position calculation `voxelPos = float3(x * voxelSize, y * voxelSize, z * voxelSize) - float3(boundsExtent.x, 0, boundsExtent.z)` is asymmetric (subtracts `boundsExtent` from X and Z, but not Y). This may be a bug or specific to the scene setup [S1, unofficial].
- **Blocky Edges:** No upsampling filter is applied to the occupancy grid, resulting in blocky edges under motion [S1, unofficial].
- **Variable Shadowing:** In the rendering shader, `float length` shadows the HLSL intrinsic `length()`, causing compilation or logic errors [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the total number of voxels. The CPU iterates through every voxel to check distance.
- **Memory Complexity:** $O(N)$ integers. For a $6 \times 6 \times 6$ grid with voxel size 0.5, $N = 12 \times 12 \times 12 = 1728$ voxels.
- **Performance:** The method is lightweight for small volumes but scales poorly with high resolution due to CPU iteration. The 30-frame update interval reduces CPU load but introduces latency in expansion [S1, unofficial].
- **Strengths:** Simple to implement; no external geometry required; GPU-friendly data structure for ray marching.
- **Limitations:** Low temporal resolution (updates every 30 frames); static spherical expansion shape (cannot represent complex smoke plumes); blocky visual artifacts [S1, unofficial].
- **When to use:** Simple, abstract smoke effects where performance is critical and visual fidelity of the expansion shape is secondary.
- **When not to use:** Realistic smoke simulation requiring complex shapes or smooth temporal updates.

## Background
This method originates from an unofficial Unity project by Schokolasbotter, built in 2024. It demonstrates a pure GPU-rendered volumetric smoke technique using a voxel grid driven by a compute shader for ray marching and Worley noise for silhouette breaking. It is presented as a tutorial or proof-of-concept rather than a production-grade engine feature [S1, unofficial].

## Sources
[S1] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
