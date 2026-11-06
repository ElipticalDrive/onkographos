# Forward Mapping

Also called: Direct projection volume rendering, object-order algorithms, splatting

## Overview
Forward mapping is a volume rendering technique that projects each voxel directly onto the image plane, relying on the traversal order of the data to ensure proper hidden surface removal [S1, unofficial]. Unlike backward mapping (image-order) methods that cast rays from the eye, forward mapping iterates over the volume data and determines the contribution of each reconstruction kernel to the screen pixels [S2, unofficial]. It is distinguished by its object-order nature, which allows for efficient hardware acceleration via accumulation buffers but requires careful handling of traversal order to maintain correct blending [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volumetric dataset (typically a cuberille or dense grid), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$, projection matrix), and optical properties (extinction $g$, emission $c$).
- Output: A 2D image (screen pixels).

**Data Structures:**
- **Cuberille:** The subset of voxels belonging to an object [S1, unofficial].
- **Image Plane:** An accumulation buffer storing the integrated radiance $I(\hat{x})$ for each pixel $\hat{x}$.

**Steps:**
1. **Determine Traversal Order:** For any viewing direction, identify the foremost corner voxel. Establish a traversal order by stepping through rows, columns, and slices such that the foremost voxel is visited last. This ensures that when contributions are accumulated, later (closer) voxels correctly overwrite or blend with earlier (farther) ones without explicit sorting [S1, unofficial].
2. **Project Voxel to Image:** For each voxel $k$ in the traversal order, compute its projection onto the image plane.
3. **Splatting:** Convolve the voxel with a reconstruction kernel $r_k(\hat{x}, \xi)$ and accumulate its contribution to the affected image pixels. The contribution is weighted by the voxel's optical properties.

**Formulas:**
The intensity $I(\hat{x})$ at a screen point $\hat{x}$ is computed by summing the contributions of all voxels:
$$
I(\hat{x}) = \sum_k g_k \int \int c_\lambda(\hat{x}, \xi) r_k(\hat{x}, \xi) \prod_j e^{-\int g_j(\hat{x}, \mu) d\mu} d\xi
$$
Simplified for linear integration and small extinction (using $e^{-x} \approx 1-x$), the contribution of voxel $k$ with extinction $g_k$ is added to the image [S2, unofficial].

**Parameters:**
- Kernel size: Typically small (e.g., 3x3 or 5x5 pixels) to limit splat footprint.
- Extinction $g$: Modeled per voxel.

**Edge Cases:**
- **Traversal Order:** Incorrect traversal order leads to incorrect hidden surface removal. The order must strictly follow the viewing direction [S1, unofficial].
- **Kernel Support:** Voxels near the edge of the image may have partial splats; boundary conditions must be handled.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_mapping(volume, camera, image_width, image_height):
    # Initialize accumulation buffer
    image = zeros((image_height, image_width, 3))
    
    # Determine traversal order based on camera direction
    # This ensures foremost voxel is visited last
    traversal_order = determine_traversal_order(volume, camera.direction)
    
    for voxel in traversal_order:
        # Project voxel center to image plane
        screen_pos = project_to_image(voxel.center, camera)
        
        # Determine kernel footprint on image
        footprint = get_kernel_footprint(screen_pos, voxel.kernel)
        
        # Splat voxel contribution to image
        for pixel in footprint:
            weight = voxel.kernel.evaluate(pixel - screen_pos)
            # Accumulate contribution
            # Note: Exact blending depends on optical model (e.g., over operator)
            # Simplified additive accumulation for linear case
            image[pixel.y, pixel.x] += voxel.color * weight * voxel.extinction
            
    return image
```

## Subtopics

### Splatting
The traditional forward mapping algorithm introduced by Westover [S2, unofficial]. It uses a reconstruction kernel to spread voxel contributions to neighboring pixels. Choose when hardware accumulation buffers are available.

### Direct Projection
A general term for projecting voxels to the image plane. Choose when object-order traversal is preferred for cache coherence in structured grids.

## Operating characteristics
- **Time Complexity:** $O(V \cdot K)$, where $V$ is the number of voxels and $K$ is the kernel footprint size. Independent of image resolution for the projection step, but accumulation is $O(P)$ where $P$ is pixels.
- **Memory Complexity:** $O(P)$ for the image buffer.
- **Strengths:** Efficient for hardware acceleration (accumulation buffers); good cache locality for structured data; no need for ray traversal per pixel.
- **Limitations:** Requires careful traversal order; kernel blurring can reduce sharpness; difficult to handle complex lighting or shadows compared to ray casting.
- **When to use:** Real-time rendering on older hardware; structured volumes where traversal order is predictable.
- **When not to use:** Unstructured grids; complex lighting; when high-quality anti-aliasing via ray-based sampling is required.

## Background
Forward mapping was introduced as an alternative to backward mapping (ray casting) to leverage hardware accumulation buffers. Westover's splatting algorithm is the seminal work [S2, unofficial]. It relies on the spatial presorting of volumetric data to avoid explicit sorting [S1, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
