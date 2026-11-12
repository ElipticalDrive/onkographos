# Backward Projection

Also called: Backward Mapping, Backward texture mapping, Forward texture mapping, image-order algorithms

## Overview
Backward Projection (also known as Backward Mapping or image-order rendering) is a volume rendering technique that iterates over all pixels of the output image to determine the contribution of the volume data to each pixel [S2, unofficial]. It solves the problem of mapping 3D volumetric data to a 2D image plane by reversing the projection direction: instead of sampling the volume and projecting to the image (forward mapping), it traces rays from the camera through the image plane into the volume [S2, unofficial]. It is distinguished from its siblings, such as [Front-to-Back Layer Traversal](1212-front-to-back-layer-traversal.md) and [Ray Marching with Octant Masking](2488-ray-marching-with-octant-masking.md), by its explicit iteration over the output image grid, which ensures every pixel is addressed exactly once and simplifies handling of non-uniform sampling or sparse data structures [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$ (scalar or vector values), a camera model (origin $\mathbf{o}$, direction $\mathbf{d}$ per pixel), and rendering parameters (opacity $\alpha$, transfer functions).
*   **Output:** A 2D image buffer $I$ of size $H \times W$.

**Data Structures:**
*   **Image Buffer:** A 2D array of size $H \times W \times C$ (where $C$ is channels, e.g., 3 for RGB).
*   **Volume Grid:** A uniform or structured grid with voxel dimensions $N_x \times N_y \times N_z$ and physical spacing $v_x, v_y, v_z$.

**Algorithm Steps:**
1.  **Initialize Image Buffer:** Set all pixel values in $I$ to zero (or background color) [S2, unofficial].
2.  **Iterate Pixels:** For each pixel $(i, j)$ in the image grid ($0 \le i < H, 0 \le j < W$):
    a.  **Ray Generation:** Compute the ray origin $\mathbf{o}_{ij}$ and direction $\mathbf{d}_{ij}$ for the current pixel based on the camera model [S2, unofficial].
    b.  **Ray-Volume Intersection:** Determine the entry and exit points ($t_{entry}, t_{exit}$) of the ray with the volume bounding box [S2, unofficial]. If no intersection, skip to next pixel.
    c.  **Traverse Volume:** Use a ray-grid traversal method (e.g., [DDA Traversal](3061-digital-differential-analyzer-dda-traversal.md)) to step through each voxel intersected by the ray segment $[t_{entry}, t_{exit}]$ [S2, unofficial].
    d.  **Sample and Accumulate:** For each voxel $k$ along the ray:
        i.  Compute the sample position $\mathbf{p}_k = \mathbf{o}_{ij} + t_k \mathbf{d}_{ij}$.
        ii. Sample the volume value $v_k = V(\mathbf{p}_k)$ using interpolation (e.g., trilinear) [S2, unofficial].
        iii. Apply transfer function to get color $c_k$ and opacity $\alpha_k$.
        iv. Accumulate into the pixel buffer using front-to-back compositing:
        $$
        I_{ij} = I_{ij} + c_k \cdot \alpha_k \cdot \prod_{m=0}^{k-1} (1 - \alpha_m)
        $$
        where the product term represents the accumulated transparency from previous samples [S2, unofficial].
3.  **Finalize:** Apply any post-processing (e.g., gamma correction) to $I$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_projection(volume, camera, image_height, image_width):
    # Initialize output image buffer
    image_buffer = zeros((image_height, image_width, 3))
    
    # Precompute volume bounds and spacing
    vol_bounds = volume.get_bounds()
    vol_spacing = volume.get_spacing()
    
    for j in range(image_height):
        for i in range(image_width):
            # 1. Generate ray for pixel (i, j)
            ray_origin, ray_direction = camera.get_ray(i, j)
            
            # 2. Find intersection with volume bounding box
            t_min, t_max = ray_box_intersection(ray_origin, ray_direction, vol_bounds)
            
            # If ray misses volume, skip
            if t_max < t_min:
                continue
                
            # 3. Traverse voxels along the ray
            # Using a standard DDA traversal as defined in parent article
            voxels = dda_traversal(ray_origin, ray_direction, vol_spacing, vol_bounds)
            
            accumulated_opacity = 0.0
            pixel_color = zeros(3)
            
            for voxel_index, t_entry, t_exit in voxels:
                # Sample volume at current ray position
                # Midpoint of the voxel interval for sampling
                t_sample = (t_entry + t_exit) / 2.0
                sample_pos = ray_origin + t_sample * ray_direction
                
                # Get voxel value and apply transfer function
                voxel_value = volume.sample(sample_pos) # Trilinear interpolation
                color, opacity = transfer_function(voxel_value)
                
                # Front-to-back compositing
                # Contribution of this voxel
                contribution = color * opacity * (1.0 - accumulated_opacity)
                pixel_color += contribution
                
                # Update accumulated opacity
                accumulated_opacity += opacity * (1.0 - accumulated_opacity)
                
                # Early exit if fully opaque
                if accumulated_opacity >= 1.0:
                    break
            
            image_buffer[j, i] = pixel_color
            
    return image_buffer
```

**Parameters:**
*   **Interpolation Method:** Trilinear is standard for uniform grids [S2, unofficial].
*   **Transfer Function:** Maps scalar values to color and opacity [S2, unofficial].
*   **Step Size:** Determined by voxel size in DDA traversal [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Ray Misses Volume:** Must be handled efficiently to avoid unnecessary computation [S2, unofficial].
*   **Precision Errors:** Floating-point errors in ray-box intersection or voxel stepping can cause gaps or overlaps; using robust geometric predicates is recommended [G].
*   **Aliasing:** Fixed-step sampling may cause aliasing; supersampling or adaptive step sizing can mitigate this [S2, unofficial].

## Subtopics

### [GPU Backprojection](5843-gpu-backprojection.md)
GPU Backprojection is a rendering pass that visualizes detected scale-space blobs by generating small slices for each blob, computing weighted contributions, and blending them to form the final image, followed by normalization. It is distinct from standard ray-casting as it operates on a sparse abstraction of the volume (scale maxima) rather than the full voxel grid, decoupling the expensive scale-space computation from the frequent backprojection step. It builds on [Backward Projection](1491-backward-projection.md) [G].

### [Ray Segment Projection to Image](7631-ray-segment-projection-to-image.md)
This method computes the bounding box of a ray segment within the normalized image plane $[0, 1] \times [0, 1]$ for backward projection. It determines the valid range of ray parameters ($t_{min}, t_{max}$) and the corresponding 2D image coordinates ($xy_{min}, xy_{max}$) where the ray intersects the image frame. It sets itself apart from sibling methods by explicitly handling near/far clipping boundaries and validity masking to ensure only rays overlapping the image frame contribute to the projection, avoiding unnecessary computation for rays missing the frame entirely.

### [Cone Beam Forward Projection with Trilinear Interpolation](8247-cone-beam-forward-projection-with-trilinear-interpolation.md)
Computes the projection of a 3D volume onto a 2D detector array for cone-beam geometry by integrating voxel values along explicit source-to-detector segments. It differs from its parent [Backward Projection](1491-backward-projection.md) by performing the forward integral using trilinear interpolation on a bricked volume layout, and from [GPU Backprojection](5843-gpu-backprojection.md) by operating on the forward ray-casting direction rather than distributing values back to the volume.

### [Circular Integral Convolution](8507-circular-integral-convolution.md)
Circular Integral Convolution generates a blurred image from a scan by integrating ray values across all angles for each point, effectively filtering the true image with a $1/r$-weighted kernel. It serves as a pre-processing step for deconvolution-based reconstruction rather than a full reconstruction method itself.

### Visibility Thresholding for Scale Backprojection
A method to prevent scale information from being backprojected to voxels that do not correspond to the target feature (e.g., vessels), improving classification accuracy by using a visibility parameter [S4]. A voxel is determined to be not visible from the Laplacian of Gaussian (LoG) extrema if, at some point along the ray connecting the two positions, the intensity is below the threshold $\tau$. Alternatively, thresholding can be based on gradient magnitude [S4]. The parameter $\tau$ (tau) is a visibility parameter used to control the sensitivity of the visibility test, based on scalar value or gradient magnitude [S4]. This is a fast approximation; more accurate results can be obtained with a region growing approach added as a post-process based on the initial size-based classification [S4].

## Operating characteristics
*   **Time Complexity:** $O(P \cdot V_{avg})$, where $P$ is the number of pixels and $V_{avg}$ is the average number of voxels intersected per ray [S2, unofficial].
*   **Memory Complexity:** $O(H \cdot W \cdot C)$ for the image buffer, plus $O(N_x \cdot N_y \cdot N_z)$ for the volume data [S2, unofficial].
*   **Performance:** Efficient for sparse volumes or when using acceleration structures like [Octree and Tree Traversal](4134-octree-and-tree-traversal.md) [S2, unofficial]. Performance is highly dependent on the efficiency of the ray-grid traversal [S2, unofficial].
*   **Strengths:** Simple implementation, easy to parallelize per pixel, handles non-uniform sampling well [S2, unofficial].
*   **Limitations:** Can be slow for dense volumes with many voxels per ray; requires careful handling of ray-box intersections [S2, unofficial].
*   **When to Use:** Ideal for interactive applications where frame rate is critical and volumes are sparse or use acceleration structures [S2, unofficial].
*   **When Not to Use:** Not suitable for very dense volumes without acceleration structures due to high computational cost [S2, unofficial].

## Background
Backward Projection, also known as Backward Mapping, was introduced as part of early volume rendering techniques to efficiently map 3D data to 2D images [S1, unofficial]. It is often contrasted with Forward Mapping, which projects volume samples to the image plane [S1, unofficial]. The method has been widely adopted in hardware-accelerated volume rendering due to its simplicity and suitability for parallel processing [S2, unofficial]. It is a foundational technique in medical imaging and scientific visualization [S1, unofficial].

## Sources
[S1] Image-based volume rendering, https://openalex.org/W2514442513, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S3] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S4] PDF Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
