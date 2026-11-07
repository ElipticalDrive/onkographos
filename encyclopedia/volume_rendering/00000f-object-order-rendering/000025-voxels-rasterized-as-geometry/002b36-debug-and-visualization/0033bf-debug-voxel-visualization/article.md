# Debug Voxel Visualization

Also called: DisplayVolume, Voxel Sphere Debug

## Overview
Debug Voxel Visualization renders auxiliary geometry (typically spheres) to provide visual feedback about the location and density of voxel data without contributing to the final volumetric appearance [S1, unofficial]. It solves the problem of inspecting sparse or dense voxel grids by converting discrete volume data into recognizable 3D primitives. It differs from [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md) and [Bounding Box Drawing](2733-bounding-box-drawing.md) by representing individual occupied voxels rather than just the volume's extent [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D voxel grid (density/intensity values), metadata (dimensions, spacing), and visualization parameters.
*   Output: A set of rendered sphere primitives positioned at voxel centers.

**Data Structures:**
*   `VolumeMetadata`: Parsed from a JSON file. Contains `dimensions` (array of 3 integers) and `spacing` (float). Validation requires `len(dimensions) == 3` [S1, unofficial].
*   `VoxelGrid`: 3D array of floats (or normalized 0.0–1.0).

**Parameters:**
*   `intensityThreshold`: Float in [0.0, 1.0]. Voxels with intensity below this value are skipped [S1, unofficial].
*   `sphereRadius`: Float in mm units (before world-space scaling). Default: 0.5f [S1, unofficial].
*   `maxSpheres`: Integer safety cap for spawned debug spheres. Default: 5000 [S1, unofficial].

**Steps:**
1.  **Load Metadata:** Parse the JSON metadata file into a `VolumeMetadata` structure. Validate that the dimension array has length 3 [S1, unofficial].
2.  **Normalize Intensity:** If the input grid is not normalized, apply percentile normalization [S1, unofficial]:
    *   Calculate $p_1$ (1st percentile) and $p_{99}$ (99th percentile) of the voxel values.
    *   If $p_{99} - p_1 < 10^{-12}$, set all values to 0.0.
    *   Otherwise, compute $v_{norm} = \text{clip}(\frac{v - p_1}{p_{99} - p_1}, 0, 1)$.
    *   For sparse channels, only consider non-zero values for percentile calculation [S1, unofficial].
3.  **Iterate Voxels:** Loop through all voxel indices $(i, j, k)$ within the grid dimensions.
4.  **Threshold Check:** If $v_{norm}[i, j, k] < \text{intensityThreshold}$, skip this voxel [S1, unofficial].
5.  **Sphere Spawning:**
    *   Check if the current number of spawned spheres $< \text{maxSpheres}$. If not, stop spawning [S1, unofficial].
    *   Calculate world position: $P = (i \cdot \text{spacing}_x, j \cdot \text{spacing}_y, k \cdot \text{spacing}_z) + \text{offset}$ [G].
    *   Generate a sphere primitive at $P$ with radius `sphereRadius` [S1, unofficial].
6.  **Render:** Draw the generated spheres using standard OpenGL rasterization with depth testing and blending [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def debug_voxel_visualization(grid, metadata, intensity_threshold=0.0, sphere_radius=0.5, max_spheres=5000):
    # 1. Validate metadata
    if len(metadata.dimensions) != 3:
        raise ValueError("Metadata dimensions must be length 3")
    
    # 2. Normalize grid if necessary (simplified for demo)
    # In practice, use normalize_intensity_percentile or normalize_channel_sparse from [S1, unofficial]
    normalized_grid = grid # Assume pre-normalized 0.0-1.0 for this snippet
    
    spawned_spheres = []
    
    # 3. Iterate
    for i in range(metadata.dimensions[0]):
        for j in range(metadata.dimensions[1]):
            for k in range(metadata.dimensions[2]):
                # 4. Threshold check
                if normalized_grid[i, j, k] < intensity_threshold:
                    continue
                
                # 5. Safety cap
                if len(spawned_spheres) >= max_spheres:
                    break
                
                # Calculate world position
                # spacing is assumed to be in metadata or a uniform scalar
                x = i * metadata.spacing[0]
                y = j * metadata.spacing[1]
                z = k * metadata.spacing[2]
                
                # Add sphere primitive
                spawned_spheres.append({
                    "position": (x, y, z),
                    "radius": sphere_radius
                })
        
        if len(spawned_spheres) >= max_spheres:
            break
        if len(spawned_spheres) >= max_spheres:
            break
            
    return spawned_spheres
```

**Edge Cases:**
*   **Precision:** Use float32 for grid values to match typical voxel storage [S1, unofficial].
*   **Boundaries:** Ensure sphere radius does not cause visual clipping if the camera is too close; debug views often use orthographic cameras [G].
*   **Degenerate Input:** If all voxel values are identical, normalization results in zero intensity, hiding all voxels unless threshold is 0.0 [S1, unofficial].

## Subtopics

### Voxel Edges
A visual effect that draws outlines of the voxel shapes on the grid with a specified width [S2, unofficial]. It sets apart from sphere spawning by rendering wireframe-like boundaries rather than solid volumes. Choose it when inspecting voxel connectivity or grid structure is required. Parameter: `width` (float) defining the edge thickness [S2, unofficial].

### Central Slice Visualization
Visualization technique for inspecting dense voxel grids by plotting central slices along X, Y, and Z axes, with optional hierarchy overlays [S3, unofficial]. It differs by providing 2D cross-sections instead of 3D geometry. Choose it for analyzing internal density distribution or octree hierarchy. Steps: Load voxel grid and build octree (`build_voxel_octree`), then plot central slices (`plot_central_slices`) which may overlay occupied voxels with red hierarchy overlays [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of voxels, as each voxel is checked against the threshold.
*   **Memory Complexity:** $O(M)$ where $M$ is the number of spawned spheres ($M \le \text{maxSpheres}$).
*   **Performance:** Rendering thousands of spheres can be expensive if not instanced. Performance depends on the number of active voxels relative to `maxSpheres`.
*   **Strengths:** Intuitive visual representation of sparse data; easy to implement with standard geometry shaders.
*   **Limitations:** High draw call count if not instanced; `maxSpheres` cap may hide data in large volumes.
*   **When to use:** Debugging voxel generation, inspecting sparse datasets, or visualizing thresholded regions.
*   **When not to use:** Final rendering or dense volume inspection where occlusion makes individual spheres indistinguishable.

## Background
Originated in voxel-based rendering SDKs and tools for debugging sparse volume data [S1, unofficial]. It is commonly used in medical imaging (MRI/CT) and 3D modeling tools (like MagicaVoxel) to inspect voxel grids [S1, S2, unofficial]. The method relies on standard graphics pipeline features for auxiliary geometry rendering [G].

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S2] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S3] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
