# Forward Mapping

Also called: Direct projection volume rendering, object-order algorithms, splatting

## Overview
Forward mapping is a direct volume ray casting technique that projects each voxel directly onto the image plane, relying on the traversal order of the data to ensure proper hidden surface removal [S1, unofficial]. Unlike image-order methods that cast rays from pixels into the volume, forward mapping iterates over the volume data and determines the contribution of each reconstruction kernel to the screen pixels [S2, unofficial]. It is distinguished by its object-order nature, where the data drives the rendering process rather than the image plane.

## How it works
**Inputs:** Volume data (typically a cuberille), viewing parameters (camera position, direction), transfer function.
**Outputs:** 2D image (color and opacity per pixel).

**Data Structures:**
The volume is stored as a 3D grid. A "cuberille" is the subset of voxels belonging to an object [S1, unofficial]. Each voxel contains a scalar value and potentially other attributes.

**Steps:**
1.  **Determine Traversal Order:** For any viewing direction, determine the foremost corner voxel. Establish a traversal order by stepping through rows, columns, and slices such that the foremost voxel is visited last. This requires determining the traversal direction along each of the major axes [S1, unofficial].
2.  **Iterate and Project:** Iterate over the volume data in the established order. For each voxel, determine its contribution to the image plane [S2, unofficial].
3.  **Kernel Convolution (Splatting):** The traditional forward mapping algorithm is splatting. It convolves every voxel in object space with a reconstruction kernel and accumulates their contributions on the image plane [S2, unofficial].
4.  **Accumulation:** Contributions are accumulated on the image plane. Because the traversal order ensures the foremost voxel is visited last, standard alpha compositing (or additive accumulation for specific kernels) can be used without explicit sorting of fragments per pixel.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize image buffer
image = zeros((width, height, 4))

# Determine traversal directions based on view vector
# dx, dy, dz are step signs (-1 or 1) for x, y, z axes
# such that the foremost voxel is visited last
dx, dy, dz = determine_traversal_direction(view_vector)

# Iterate over volume data in traversal order
for z in range(volume_depth, -1, -dz) if dz > 0 else range(0, volume_depth + 1, dz):
    for y in range(volume_height, -1, -dy) if dy > 0 else range(0, volume_height + 1, dy):
        for x in range(volume_width, -1, -dx) if dx > 0 else range(0, volume_width + 1, dx):
            
            # Get voxel value
            val = volume[x, y, z]
            
            # Map to optical properties (color C, opacity alpha)
            # Using transfer function T
            C, alpha = transfer_function(val)
            
            # Project voxel center to image plane
            # This step involves transforming voxel coordinates to screen space
            # and determining the footprint of the reconstruction kernel
            screen_coords = project_to_screen(x, y, z, view_matrix)
            
            # Splat: Convolve voxel with reconstruction kernel
            # Accumulate contribution to image pixels covered by kernel
            accumulate_splat(screen_coords, C, alpha, image)
```

**Parameters:**
-   Kernel size: Determines the footprint of the splat.
-   Transfer function: Maps scalar values to color and opacity.

**Edge Cases:**
-   Precision issues in projection calculations.
-   Boundary conditions for voxels near the edge of the volume.

## Subtopics

### [Splatting](1576-splatting.md)
The traditional forward mapping algorithm that convolves every voxel with a reconstruction kernel. It is chosen for its simplicity and hardware acceleration potential.

### [Cuberille](38-cuberille.md)
The subset of voxels in the data volume that belong to an object. It is used to optimize forward mapping by only processing relevant voxels.

## Operating characteristics
-   **Time Complexity:** $O(V)$, where $V$ is the number of voxels. Each voxel is processed once.
-   **Memory Complexity:** $O(I)$, where $I$ is the number of image pixels, for the accumulation buffer.
-   **Strengths:** Simple implementation, efficient for sparse data (using cuberilles), naturally handles hidden surface removal via traversal order.
-   **Limitations:** Can suffer from overdraw and blending artifacts if not carefully implemented. Performance depends on the number of voxels rather than the number of pixels.
-   **When to use:** When the volume is sparse or when hardware acceleration for splatting is available.
-   **When not to use:** When high-quality anti-aliasing is required without additional filtering, or when the volume is dense and most voxels contribute to the final image.

## Background
Forward mapping was introduced as an alternative to backward mapping (ray casting) methods. The term "splatting" was introduced by Westover in 1991 [S2, unofficial]. The concept relies on the intrinsic spatial presorting of volumetric data sets [S1, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
