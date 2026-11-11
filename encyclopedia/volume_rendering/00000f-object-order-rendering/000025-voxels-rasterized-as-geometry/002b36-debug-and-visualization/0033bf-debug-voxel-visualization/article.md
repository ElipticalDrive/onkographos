# Debug Voxel Visualization

Also called: DisplayVolume, Voxel Sphere Debug

## Overview
Debug Voxel Visualization converts scalar field data into geometric primitives (spheres) for spatial inspection, allowing developers to verify voxel occupancy and intensity distribution [S1, unofficial]. It differs from sibling methods like [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md) by using volumetric primitives rather than empty bounding boxes, providing a denser visual representation of the data's internal structure.

## How it works
**Inputs:**
- A 3D scalar field (voxel grid) containing intensity values.
- `VolumeMetadata`: A JSON structure containing dimensions and metadata, validated to ensure exactly 3 dimensions [S1, unofficial].

**Outputs:**
- A set of sphere primitives submitted to the graphics pipeline [G].

**Data Structures:**
- `VolumeMetadata`: JSON object with fields `dimensions` (array of 3 integers) and other metadata.
- `Sphere`: Defined by position $(x, y, z)$ and radius $r$.

**Parameters:**
- `intensityThreshold`: Float in $[0.0, 1.0]$. Voxels with intensity below this value are skipped [S1, unofficial].
- `sphereRadius`: Float in mm units before scaling. Default: $0.5$ [S1, unofficial].
- `maxSpheres`: Integer safety cap for spawned debug spheres. Default: $5000$ [S1, unofficial].

**Steps:**
1. **Load and Validate:** Load `VolumeMetadata` from the source (e.g., `StreamingAssets`). Validate that the dimension array length is exactly 3 [S1, unofficial].
2. **Normalize Intensity:** Normalize the voxel grid to $[0, 1]$ using percentile-based scaling to handle sparse or non-uniform data [S1, unofficial]:
   - Compute $p_1$ (1st percentile) and $p_{99}$ (99th percentile) of the voxel values.
   - If $p_{99} - p_1 < 10^{-12}$, set all normalized values to $0$.
   - Otherwise, compute $v_{norm} = \text{clip}\left(\frac{v - p_1}{p_{99} - p_1}, 0, 1\right)$ for each voxel $v$.
   - For sparse channels, only consider voxels where $v > 0$ for percentile calculation [S1, unofficial].
3. **Spawn Spheres:** Iterate through the normalized voxel grid:
   - If $v_{norm} < \text{intensityThreshold}$, skip the voxel.
   - If the number of spawned spheres reaches `maxSpheres`, stop.
   - Otherwise, spawn a sphere at the voxel's center with `sphereRadius`.
4. **Render:** Submit the generated spheres to the graphics pipeline [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def debug_voxel_visualization(grid: np.ndarray, metadata: dict, intensity_threshold: float = 0.0, sphere_radius: float = 0.5, max_spheres: int = 5000) -> list:
    """
    grid: 3D numpy array of float32 voxel intensities
    metadata: dict with 'dimensions' key (list of 3 ints)
    Returns: list of sphere objects (dict with 'pos' and 'radius')
    """
    # Validate metadata
    assert len(metadata['dimensions']) == 3, "Metadata must have exactly 3 dimensions"
    
    # Normalize intensity
    p1 = np.percentile(grid, 1)
    p99 = np.percentile(grid, 99)
    
    if p99 - p1 < 1e-12:
        grid_norm = np.zeros_like(grid, dtype=np.float32)
    else:
        grid_norm = (grid - p1) / (p99 - p1)
        grid_norm = np.clip(grid_norm, 0, 1).astype(np.float32)
        
    spheres = []
    dims = metadata['dimensions']
    
    for x in range(dims[0]):
        for y in range(dims[1]):
            for z in range(dims[2]):
                if len(spheres) >= max_spheres:
                    break
                
                if grid_norm[x, y, z] >= intensity_threshold:
                    # Assuming voxel centers are at integer coordinates
                    pos = (x, y, z)
                    spheres.append({'pos': pos, 'radius': sphere_radius})
        if len(spheres) >= max_spheres:
            break
    return spheres
```

**Edge Cases and Pitfalls:**
- **Precision:** If $p_{99} \approx p_1$, the normalization denominator becomes zero, resulting in all zeros [S1, unofficial].
- **Boundaries:** The algorithm assumes voxel centers are at integer coordinates. Scaling must be applied consistently between `sphereRadius` (in mm) and the grid coordinates [S1, unofficial].
- **Degenerate Input:** Empty grids or grids with all zero values result in no spheres being spawned [S1, unofficial].

## Subtopics

### Voxel Edges
A visual effect that draws outlines of the voxel shapes on the grid with a specified width [S2, unofficial]. It uses the `voxel_edges <width>` parameter to define the outline thickness, providing a wireframe-like inspection of the voxel structure without filling the volume [S2, unofficial].

### Central Slice Visualization
A technique for inspecting dense voxel grids by plotting central slices along the X, Y, and Z axes, optionally overlaying hierarchy information [S3, unofficial]. It involves building an octree from the voxel grid (`build_voxel_octree`) and then plotting the central slices (`plot_central_slices`), which can highlight occupied voxels with red hierarchy overlays from the octree [S3, unofficial].

### Wireframe Block Highlighter
Draws a wireframe outline around a target voxel block using 12 LineRenderer components [S4, unofficial]. It calculates block bounds with padding (`min -= padding`, `max += padding`), computes 8 corners, and assigns line endpoints using a predefined Edges array of 12 corner index pairs (e.g., `{0,1}, {1,2}, ...`) [S4, unofficial]. Parameters include a line width of $0.025$ and an outline padding of $0.01$ [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the total number of voxels, as each voxel is checked once. The normalization step is also $O(N)$.
- **Memory Complexity:** $O(N)$ for storing the normalized grid and the list of spheres. The `maxSpheres` cap limits the output size but not the input processing.
- **Performance:** Performance is limited by the number of spheres spawned. The default `maxSpheres` of 5000 ensures a manageable draw call count for debug purposes [S1, unofficial].
- **Strengths:** Provides a direct visual mapping of intensity values to spatial occupancy, useful for debugging scalar field data [S1, unofficial].
- **Limitations:** Not suitable for large-scale rendering due to the high number of primitives. The threshold-based approach may hide low-intensity features if the threshold is set too high [S1, unofficial].

## Background
Debug Voxel Visualization is part of the VRDF SDK, an unofficial toolset for volume rendering and debugging [S1, unofficial]. It was developed to provide developers with a simple way to visualize voxel data during the development process, leveraging standard graphics pipeline primitives [S1, unofficial].

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S2] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S3] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
[S4] philip2012/voxel-forge, https://github.com/philip2012/voxel-forge, fetched 2026-10-27 (unofficial)
