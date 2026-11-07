# Debug Voxel Visualization

Also called: Threshold-based Sphere Spawning, DisplayVolume, Voxel Sphere Debug

## Overview
Debug Voxel Visualization renders occupied voxels as geometric primitives (typically spheres or cubes) to inspect the spatial distribution and density of scalar field data. It solves the problem of visualizing sparse or dense voxel grids by converting discrete volume data into visible geometry, allowing developers to verify data loading, normalization, and occupancy. It differs from ray-marched volume rendering by using explicit geometry submission to the graphics pipeline, prioritizing immediate visual feedback over photorealistic integration [G].

## How it works
The method converts a 3D scalar grid into a set of geometric primitives based on value thresholds and occupancy.

**Inputs and Outputs**
- **Input**: A 3D scalar volume grid (float32, normalized to [0, 1]), metadata (dimensions, bounds), and visualization parameters.
- **Output**: A list of geometric primitives (spheres or cubes) submitted to the graphics pipeline.

**Data Structures**
- `VolumeMetadata`: Contains dimensions $N = (N_x, N_y, N_z)$, world bounds $P_{min}$ and $P_{max}$, and normalization parameters.
- `VoxelPrimitive`: Represents a single debug object.
  - `position`: World-space center $(x, y, z)$.
  - `radius`: Sphere radius in world units.
  - `color`: RGBA color derived from voxel intensity or channel data.

**Steps**
1. **Load and Validate Metadata**: Load volume metadata (e.g., from JSON) and validate that dimension length is 3 [S1, unofficial].
2. **Normalize Intensity**: Normalize the voxel grid values to [0, 1] using percentile clipping (1st and 99th percentiles) to handle sparse data [S1, unofficial].
   $$
   v_{norm} = \text{clip}\left(\frac{v - p_1}{p_99 - p_1}, 0, 1\right)
   $$
   where $p_1$ and $p_99$ are the 1st and 99th percentiles of non-zero values.
3. **Iterate and Filter**: Iterate through all voxel indices $(i, j, k)$.
   - If $v_{norm}(i, j, k) < \text{intensityThreshold}$, skip the voxel [S1, unofficial].
   - If the number of spawned primitives exceeds `maxSpheres`, stop [S1, unofficial].
4. **Transform to World Space**: Calculate the world position of the voxel center. For unit voxels scaled by $V_{size}$:
   $$
   P_{world} = (i, j, k) \times V_{size}
   $$
   For non-unit grids with bounds $P_{min}$ and $P_{max}$:
   $$
   P_{world} = P_{min} + \frac{(i, j, k)}{N} \times (P_{max} - P_{min})
   $$
5. **Generate Primitive**: Create a sphere (or cube) at $P_{world}$ with radius $R = \text{sphereRadius} \times V_{size}$ [S1, unofficial].
6. **Submit to Pipeline**: Transform primitives using the Model-View-Projection (MVP) matrix and rasterize [G].

**Parameters**
- `intensityThreshold`: Float in [0.0, 1.0]. Voxels below this value are skipped. Default: 0.0 [S1, unofficial].
- `sphereRadius`: Float in mm units before scaling. Default: 0.5f [S1, unofficial].
- `maxSpheres`: Integer safety cap for spawned debug spheres. Default: 5000 [S1, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: Use float32 for voxel values to maintain memory efficiency, but ensure normalization does not underflow for sparse data [S1, unofficial].
- **Boundaries**: Ensure $P_{min}$ and $P_{max}$ are correctly mapped to avoid voxel misalignment [G].
- **Degenerate Input**: If $p_99 - p_1 < 1e-12$, set all normalized values to 0 or 1 depending on occupancy [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def debug_voxel_visualization(volume_grid, metadata, intensity_threshold=0.0, sphere_radius_mm=0.5, max_spheres=5000):
    """
    Generates debug spheres for occupied voxels.
    volume_grid: 3D numpy array (float32)
    metadata: dict with 'dims', 'p_min', 'p_max'
    """
    dims = metadata['dims']
    p_min = metadata['p_min']
    p_max = metadata['p_max']
    
    # Normalize intensity
    p1, p99 = np.percentile(volume_grid[volume_grid > 0], [1, 99]) if np.any(volume_grid > 0) else (0, 1)
    if p99 - p1 < 1e-12:
        norm_grid = np.zeros_like(volume_grid, dtype=np.float32)
    else:
        norm_grid = np.clip((volume_grid - p1) / (p99 - p1), 0, 1).astype(np.float32)
        
    voxel_size = (p_max - p_min) / dims
    primitives = []
    count = 0
    
    for i in range(dims[0]):
        for j in range(dims[1]):
            for k in range(dims[2]):
                if count >= max_spheres:
                    break
                if norm_grid[i, j, k] < intensity_threshold:
                    continue
                    
                # World position
                pos = p_min + np.array([i, j, k]) * voxel_size
                radius = sphere_radius_mm * voxel_size[0] # Assume isotropic for simplicity
                
                primitives.append({
                    'position': pos,
                    'radius': radius,
                    'color': [norm_grid[i, j, k], 0, 0, 1] # Red tinted by intensity
                })
                count += 1
    return primitives
```

## Subtopics

### Voxel Edges
A visual effect that draws outlines of the voxel shapes on the grid with a specified width. It is used to emphasize the grid structure rather than volume occupancy. The parameter `width` controls the line thickness of the edges [S2, unofficial].

### Central Slice Visualization
A technique for inspecting dense voxel grids by plotting central slices along X, Y, and Z axes. It optionally overlays light occupied voxels with red hierarchy overlays from an octree structure. Steps include loading the voxel grid, building an octree (`build_voxel_octree`), and plotting central slices (`plot_central_slices`) with optional hierarchy depth limits [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the total number of voxels, as each voxel is checked once.
- **Memory Complexity**: $O(M)$ where $M$ is the number of spawned primitives (spheres/cubes), capped by `maxSpheres`.
- **Performance**: Fast for sparse data; degrades rapidly for dense data due to the large number of primitives submitted to the GPU.
- **Strengths**: Immediate visual feedback, easy to debug data loading and normalization.
- **Limitations**: Not suitable for large dense volumes due to primitive count limits; does not show internal structure without slicing.

## Background
Debug Voxel Visualization is a standard technique in volume data inspection tools, often implemented in SDKs for voxel-based rendering and editing. It relies on basic graphics pipeline features (geometry instancing or individual primitive submission) and is commonly found in tools like MagicaVoxel for debugging voxel grids and in custom volume rendering SDKs for data validation [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S2] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S3] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
