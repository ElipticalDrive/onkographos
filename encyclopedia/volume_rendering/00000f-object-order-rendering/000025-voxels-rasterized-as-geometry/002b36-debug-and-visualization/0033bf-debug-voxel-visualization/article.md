# Debug Voxel Visualization

Also called: Threshold-based Sphere Spawning, DisplayVolume, Voxel Sphere Debug

## Overview
Debug Voxel Visualization renders occupied voxels as geometric primitives to inspect the spatial distribution and density of scalar field data [G]. It solves the problem of visualizing sparse or dense voxel grids by converting discrete volume data into visible geometry, allowing developers to verify data loading, normalization, and occupancy [G]. It differs from ray-marched volume rendering by using explicit geometry submission to the graphics pipeline, prioritizing immediate visual feedback over photorealistic integration [G].

## How it works
The method converts a 3D scalar grid into a set of geometric primitives based on value thresholds and occupancy [G].

**Inputs and Outputs**
- **Input**: A 3D scalar volume grid, metadata (dimensions, bounds), and visualization parameters [S1, unofficial].
- **Output**: A list of geometric primitives submitted to the graphics pipeline [G].

**Data Structures**
- `VolumeMetadata`: Contains dimensions $N = (N_x, N_y, N_z)$, world bounds $P_{min}$ and $P_{max}$, and normalization parameters [S1, unofficial].
- `VoxelPrimitive`: Represents a single debug object with fields for `position`, `radius`, and `color` [G].

**Steps**
1. **Load and Validate Metadata**: Load volume metadata and validate that dimension length equals 3 [S1, unofficial].
2. **Normalize Intensity**: Normalize the voxel grid values to $[0, 1]$ using percentile clipping (1st and 99th percentiles) to handle sparse data [S1, unofficial].
   $$
   v_{norm} = \text{clip}\left(\frac{v - p_1}{p_{99} - p_1}, 0, 1\right)
   $$
   where $p_1$ and $p_{99}$ are the 1st and 99th percentiles of non-zero values [S1, unofficial].
3. **Iterate and Filter**: Iterate through all voxel indices $(i, j, k)$ [G]. If the normalized value at $(i, j, k)$ is below `intensityThreshold`, skip the voxel [S1, unofficial]. If the number of spawned primitives exceeds `maxSpheres`, stop [S1, unofficial].
4. **Transform to World Space**: Calculate the world position of the voxel center [G]. For unit voxels scaled by $V_{size}$:
   $$
   P_{world} = (i, j, k) \times V_{size}
   $$
   For non-unit grids with bounds $P_{min}$ and $P_{max}$:
   $$
   P_{world} = P_{min} + \frac{(i, j, k)}{N} \times (P_{max} - P_{min})
   $$
   [G]
5. **Generate Primitive**: Create a sphere at $P_{world}$ with a radius specified in millimeters that is scaled to world units [S1, unofficial].
6. **Submit to Pipeline**: Transform primitives using the Model-View-Projection matrix and rasterize them [G].

**Parameters**
- `intensityThreshold`: Float in $[0.0, 1.0]$. Voxels below this value are skipped during debug sphere spawning. Default: 0.0 [S1, unofficial].
- `sphereRadius`: Float in mm units before scaling. Default: 0.5f [S1, unofficial].
- `maxSpheres`: Integer safety cap for spawned debug spheres. Default: 5000 [S1, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: Use float32 for voxel values [S1, unofficial].
- **Boundaries**: Ensure $P_{min}$ and $P_{max}$ are correctly mapped to avoid voxel misalignment [G].
- **Degenerate Input**: If $p_{99} - p_1 < 10^{-12}$, set occupied voxels to 1.0 and empty voxels to 0.0 [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def debug_voxel_visualization(volume_grid, metadata, intensity_threshold=0.0, sphere_radius_mm=0.5, max_spheres=5000):
    dims = metadata['dims']
    p_min = metadata['p_min']
    p_max = metadata['p_max']
    
    # Extract non-zero values for percentile calculation
    non_zero = sorted([v for v in volume_grid.flatten() if v > 0])
    if non_zero:
        p1 = non_zero[len(non_zero) // 100]
        p99 = non_zero[int(len(non_zero) * 0.99)]
    else:
        p1, p99 = 0.0, 1.0
        
    norm_grid = []
    for v in volume_grid.flatten():
        if p99 - p1 < 1e-12:
            norm_val = 1.0 if v > 0 else 0.0
        else:
            norm_val = max(0.0, min(1.0, (v - p1) / (p99 - p1)))
        norm_grid.append(norm_val)
        
    nx, ny, nz = dims
    grid_3d = [[[norm_grid[i * ny * nz + j * nz + k] for k in range(nz)] for j in range(ny)] for i in range(nx)]
    
    voxel_size = [(p_max[d] - p_min[d]) / dims[d] for d in range(3)]
    primitives = []
    count = 0
    
    for i in range(nx):
        for j in range(ny):
            for k in range(nz):
                if count >= max_spheres:
                    break
                if grid_3d[i][j][k] < intensity_threshold:
                    continue
                    
                pos = [p_min[d] + i * voxel_size[d] for d in range(3)]
                radius = sphere_radius_mm * voxel_size[0]
                
                primitives.append({
                    'position': pos,
                    'radius': radius,
                    'color': [grid_3d[i][j][k], 0.0, 0.0, 1.0]
                })
                count += 1
    return primitives
```

## Subtopics

### Voxel Edges
A visual effect that draws outlines of the voxel shapes on the grid with a specified width [S2, unofficial]. It is used to emphasize the grid structure rather than volume occupancy [G]. The parameter `width` controls the line thickness of the edges [S2, unofficial].

### Central Slice Visualization
A technique for inspecting dense voxel grids by plotting central slices along X, Y, and Z axes [S3, unofficial]. It optionally overlays light occupied voxels with red hierarchy overlays from an octree structure [S3, unofficial]. Steps include loading the voxel grid, building an octree, and plotting central slices with optional hierarchy depth limits [S3, unofficial].

### Wireframe Block Highlighter
Draws a wireframe outline around a target voxel block using 12 LineRenderer components [S4, unofficial]. Steps include calculating block bounds, applying padding, computing 8 corners, and assigning line endpoints using an Edges array of 12 corner index pairs [S4, unofficial]. Parameters include line width (0.025f), outline padding (0.01f), and outline color (black) [S4, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the total number of voxels, as each voxel is checked once [G].
- **Memory Complexity**: $O(M)$ where $M$ is the number of spawned primitives, capped by `maxSpheres` [S1, unofficial].
- **Performance**: Fast for sparse data; degrades rapidly for dense data due to the large number of primitives submitted to the GPU [G].
- **Strengths**: Immediate visual feedback, easy to debug data loading and normalization [G].
- **Limitations**: Not suitable for large dense volumes due to primitive count limits; does not show internal structure without slicing [G].

## Background
Debug Voxel Visualization is used in volume data inspection tools and custom volume rendering SDKs for data validation [S1, unofficial]. It relies on basic graphics pipeline features for geometry submission and rasterization [G].

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S2] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S3] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
[S4] philip212/voxel-forge, https://github.com/philip212/voxel-forge, fetched 2026-10-27 (unofficial)
