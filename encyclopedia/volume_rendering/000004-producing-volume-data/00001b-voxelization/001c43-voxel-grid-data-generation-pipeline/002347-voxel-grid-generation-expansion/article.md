# Voxel Grid Generation & Expansion

## Overview
This method generates a 3D occupancy grid representing a volumetric smoke cloud by expanding a spherical radius from a central origin point over time [S1, unofficial]. It solves the problem of creating a discrete, GPU-friendly voxel representation for ray-marched volumetric rendering without using particle systems or billboard geometry [S1, unofficial]. It is distinguished from standard point-splatting by evaluating a continuous sphere falloff against a discrete grid, and from the [DCX Pipeline](3770-dcx-pipeline.md) by its procedural, time-driven expansion rather than static file parsing.

## How it works
**Inputs:**
- `boundsExtent` (Vector3): Half-size of the volume bounds. Default: $(3, 3, 3)$ [S1, unofficial].
- `voxelSize` (float): Grid resolution. Default: $0.5$ [S1, unofficial].
- `smokeRadius` (float): Maximum expansion radius. Default: $4$ [S1, unofficial].
- `smokeExpansionTime` (float): Time required to reach maximum radius [S1, unofficial].
- `smokeOrigin` (Vector3): Center point of volume expansion [S1, unofficial].
- `smokeTimer` (float): Current elapsed time for expansion [S1, unofficial].
- `easingCurve`: A function mapping $[0, 1]$ to $[0, 1]$ for non-linear expansion [S1, unofficial].

**Outputs:**
- `ComputeBuffer` named `_SmokeVoxels`: A `StructuredBuffer<int>` of size `totalVoxels`, containing $1$ for occupied voxels and $0$ for empty ones [S1, unofficial].

**Data Structures:**
- The buffer uses linear memory, indexed by $x + y \cdot \text{voxelsX} + z \cdot \text{voxelsX} \cdot \text{voxelsY}$ [S1, unofficial].
- Each element is a 4-byte integer [S1, unofficial].

**Steps:**
1. Calculate grid dimensions:
   $$ \text{voxelsX} = \lceil \text{boundsExtent}.x / \text{voxelSize} \rceil $$
   $$ \text{voxelsY} = \lceil \text{boundsExtent}.y / \text{voxelSize} \rceil $$
   $$ \text{voxelsZ} = \lceil \text{boundsExtent}.z / \text{voxelSize} \rceil $$
   [S1, unofficial]

2. Initialize a buffer of size $\text{voxelsX} \times \text{voxelsY} \times \text{voxelsZ}$ with integers set to $0$ [S1, unofficial].

3. On update (every 30 frames), clear the buffer to $0$ [S1, unofficial].

4. Iterate through all voxel indices $(x, y, z)$ where $0 \le x < \text{voxelsX}$, $0 \le y < \text{voxelsY}$, $0 \le z < \text{voxelsZ}$ [S1, unofficial].

5. Compute the world position of the voxel center:
   $$ \text{voxelPos} = (x \cdot \text{voxelSize}, y \cdot \text{voxelSize}, z \cdot \text{voxelSize}) - (\text{boundsExtent}.x, 0, \text{boundsExtent}.z) $$
   [S1, unofficial]

6. Compute the Euclidean distance from the voxel to the smoke origin:
   $$ \text{distance} = \| \text{voxelPos} - \text{smokeOrigin} \| $$
   [S1, unofficial]

7. Compute the current expansion radius:
   $$ \text{normalValue} = \text{smokeTimer} / \text{smokeExpansionTime} $$
   $$ \text{currentSmokeRadius} = \text{easingCurve}(\text{normalValue}) \cdot \text{smokeRadius} $$
   [S1, unofficial]

8. If $\text{distance} \le \text{currentSmokeRadius}$, set the buffer value at index $x + y \cdot \text{voxelsX} + z \cdot \text{voxelsX} \cdot \text{voxelsY}$ to $1$ [S1, unofficial].

9. Upload the data to the GPU buffer via `smokeBuffer.SetData(smokeData)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_grid(bounds_extent, voxel_size, smoke_radius, smoke_expansion_time, smoke_origin, smoke_timer, easing_curve):
    # 1. Calculate dimensions
    voxels_x = math.ceil(bounds_extent.x / voxel_size)
    voxels_y = math.ceil(bounds_extent.y / voxel_size)
    voxels_z = math.ceil(bounds_extent.z / voxel_size)
    total_voxels = voxels_x * voxels_y * voxels_z
    
    # 2 & 3. Initialize and clear buffer
    smoke_data = [0] * total_voxels
    
    # 7. Compute current radius
    # Handle division by zero if expansion time is zero
    if smoke_expansion_time == 0:
        normal_value = 0.0
    else:
        normal_value = smoke_timer / smoke_expansion_time
        
    current_smoke_radius = easing_curve(normal_value) * smoke_radius
    
    # 4, 5, 6, 8. Iterate, compute position/distance, and fill
    for x in range(voxels_x):
        for y in range(voxels_y):
            for z in range(voxels_z):
                # Evidence specifies asymmetric offset: (boundsExtent.x, 0, boundsExtent.z)
                voxel_pos_x = x * voxel_size - bounds_extent.x
                voxel_pos_y = y * voxel_size
                voxel_pos_z = z * voxel_size - bounds_extent.z
                
                dx = voxel_pos_x - smoke_origin.x
                dy = voxel_pos_y - smoke_origin.y
                dz = voxel_pos_z - smoke_origin.z
                distance = math.sqrt(dx*dx + dy*dy + dz*dz)
                
                if distance <= current_smoke_radius:
                    index = x + y * voxels_x + z * voxels_x * voxels_y
                    smoke_data[index] = 1
                    
    # 9. Upload to GPU (simulated)
    # smokeBuffer.SetData(smoke_data)
    return smoke_data
```

**Edge Cases:**
- If `smokeExpansionTime` is $0$, division by zero occurs in normalization; handle by clamping `normal_value` to $0$ or $1$ [G].
- Buffer updates occur every 30 frames, causing potential stutter or delay in volume expansion [S1, unofficial].
- Smoke is rendered to a reduced-resolution target and composited back up without an upsampling filter, resulting in blocky edges under motion [S1, unofficial].

## Subtopics

### [Point Wisp Voxelization](10591-point-wisp-voxelization.md)
Point Wisp Voxelization is a procedural method for generating volumetric smoke data by distributing points around a guide particle using fractal noise and baking them into a voxel grid. It solves the problem of creating detailed, organic-looking smoke volumes from sparse particle data, distinguishing itself from standard point-splatting by using correlated random walks and radial displacement to form cohesive wisps rather than isolated blobs. Choose it when organic, noise-driven smoke structures are required over simple geometric expansions.

### [Virtual Smoke](17061-virtual-smoke.md)
Virtual Smoke is an interactive 3D flow visualization technique that simulates the injection of colored smoke from a user-selected seed point into a fixed vector field, where the smoke color is mapped to a local scalar value. It solves the problem of simultaneously visualizing flow direction (via vector fields) and scalar properties (e.g., temperature, concentration) in large 3D datasets. Unlike this method's time-driven sphere expansion, Virtual Smoke uses an event queue ordered by propagation time derived from velocity vectors to grow a control volume. Choose it for interactive flow and scalar field visualization.

### [Voxel CFDG Expansion](19345-voxel-cfdg-expansion.md)
Voxel CFDG Expansion generates a 3D voxel structure representing a smoke volume using a Context-Free Grammar (CFDG) style recursive branching algorithm with probability-based face selection. It differs from sibling methods by explicitly modeling hierarchical growth from a root voxel, whereas Point Wisp Voxelization relies on particle trajectories and Virtual Smoke simulates fluid dynamics. Choose it when procedural, grammar-based hierarchical smoke structures are needed.

### [3D-to-4D Voxel Expansion Algorithm](19990-3d-to-4d-voxel-expansion-algorithm.md)
The 3D-to-4D Voxel Expansion Algorithm converts a static 3D occupancy grid into a 4D object-centric representation by clustering voxels based on geometry, color, and dynamics. It solves the problem of separating mixed volumetric data into distinct object slots, assuming voxels belonging to the same object share similar motion and appearance features. Unlike Point Wisp Voxelization which relies on point cloud structures, or Virtual Smoke which models fluid dynamics explicitly, this method uses a graph-based connected components approach on pre-filtered voxel features to achieve unsupervised object-centric decomposition. Choose it for dynamic object tracking and decomposition in volumetric data.

### Block-Palette Map Generation
Block-Palette Map Generation is a tool to generate a static physical walk-through grid of all registered voxels, exporting a block-palette matrix [S2, unofficial]. It is used for inspecting textures, normals, and colors side-by-side [S2, unofficial]. Steps: Run `python map_maker.py` to export the block-palette matrix to `map/maps/default_map.npz` [S2, unofficial]. Choose it for debugging and visual inspection of voxelized maps.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the total number of voxels in the grid, as every voxel is checked against the radius [S1, unofficial].
- **Memory Complexity:** $O(N)$ integers for the occupancy buffer [S1, unofficial].
- **Performance:** The method runs on the CPU and uploads data to the GPU every 30 frames [S1, unofficial]. This low-frequency update causes potential stutter or delay in volume expansion [S1, unofficial].
- **Strengths:** Simple to implement; provides a discrete occupancy grid suitable for ray marching with empty-space skipping [S1, unofficial].
- **Limitations:** Renders a "smoke ball" (sphere falloff) rather than complex smoke shapes; lacks velocity advection; blocky edges due to lack of upsampling filter [S1, unofficial].
- **When to use:** When a simple, expanding volumetric cloud is needed for stylized or abstract effects in Unity [S1, unofficial].
- **When not to use:** When realistic smoke dynamics, complex shapes, or high-frequency updates are required [S1, unofficial].

## Background
This method originates from a 2024 Unity project by Schokolasbotter, which implements a volumetric smoke renderer entirely on the GPU using a voxel grid driven by a two-stage ray march [S1, unofficial]. It is part of a pipeline that includes animated Worley noise for silhouette breaking [S1, unofficial].

## Sources
[S1] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
[S2] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
