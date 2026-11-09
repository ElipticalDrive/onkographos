# Backward Projection

Also called: Backward Mapping, Backward and forward texture mapping algorithms, Backward texture mapping, Forward texture mapping, image-order algorithms

## Overview
Backward Projection is an image-order rendering technique that iterates over all pixels of the output image to determine the contributions of the volume rendering integral to the current pixel [S2, unofficial]. It is the inverse of forward projection (object-order), where the algorithm iterates over volume elements and projects them onto the image plane. This method is distinct from its siblings in the Ray-grid traversal family because it drives the traversal from the image space, ensuring every output pixel is computed exactly once, which simplifies blending and opacity handling [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D volume dataset $V$, camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$ per pixel), and transfer functions for color and opacity.
- **Output:** A 2D image buffer $I$ with dimensions $H \times W$.

**Data Structures:**
- **Image Buffer:** A 2D array of size $H \times W \times C$ (where $C$ is channels, e.g., 3 for RGB + 1 for Alpha).
- **Volume Data:** A 3D array $V(x, y, z)$ with dimensions $N_x \times N_y \times N_z$.
- **Ray Parameters:** For each pixel $(i, j)$, a ray is defined by origin $\mathbf{o}_{i,j}$ and direction $\mathbf{d}_{i,j}$.

**Steps:**
1.  **Initialize Image Buffer:** Set all pixels in $I$ to zero (or background color).
2.  **Iterate Pixels:** For each pixel $(i, j)$ in the image grid:
    a.  **Construct Ray:** Determine the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ corresponding to pixel $(i, j)$ based on camera geometry.
    b.  **Ray-Grid Traversal:** Use a ray-grid traversal method (e.g., DDA) to iterate through the voxels intersected by the ray [S2, unofficial][S3, unofficial]. The traversal determines the entry and exit points of the ray within each voxel.
    c.  **Sample Volume:** For each voxel visited, sample the volume data $V$ at the intersection point using interpolation (e.g., trilinear).
    d.  **Apply Transfer Function:** Map the sampled value to color $c$ and opacity $\alpha$.
    e.  **Composite:** Accumulate the contribution into the image buffer using front-to-back compositing:
        $$ I_{i,j} = I_{i,j} + c \cdot \alpha \cdot \prod_{k=1}^{l-1} (1 - \alpha_k) $$
        where $l$ is the current sample index along the ray [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_projection(volume, camera, image_height, image_width):
    # Initialize output image buffer
    image_buffer = zeros((image_height, image_width, 4))
    
    # Iterate over all pixels
    for j in range(image_height):
        for i in range(image_width):
            # Construct ray for pixel (i, j)
            ray_origin, ray_direction = camera.get_ray(i, j)
            
            # Initialize compositing variables
            accumulated_color = [0.0, 0.0, 0.0, 0.0]
            accumulated_opacity = 0.0
            
            # Ray-grid traversal (e.g., DDA)
            # This yields a sequence of (voxel_index, t_entry, t_exit)
            for voxel_idx, t_entry, t_exit in dda_traversal(ray_origin, ray_direction, volume.grid):
                
                # Calculate sample position along the ray
                t_sample = (t_entry + t_exit) / 2.0
                sample_pos = ray_origin + t_sample * ray_direction
                
                # Sample volume data (e.g., trilinear interpolation)
                value = volume.sample(sample_pos)
                
                # Apply transfer function to get color and opacity
                color, opacity = transfer_function(value)
                
                # Front-to-back compositing
                # Weight of current sample
                weight = opacity * (1.0 - accumulated_opacity)
                
                # Update accumulated color and opacity
                accumulated_color += color * weight
                accumulated_opacity += weight
                
                # Early ray termination if fully opaque
                if accumulated_opacity >= 1.0:
                    break
            
            # Store result in image buffer
            image_buffer[j, i] = accumulated_color
            
    return image_buffer
```

**Parameters:**
- **Interpolation Method:** Trilinear is common for accuracy; nearest-neighbor for speed [S2, unofficial].
- **Transfer Function:** Maps scalar values to color and opacity.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in ray traversal can cause gaps or overlaps in the image if not handled carefully [S2, unofficial].
- **Boundaries:** Rays missing the volume entirely should be handled efficiently to avoid unnecessary computation.
- **Degenerate Input:** Rays parallel to grid axes require special handling for step directions and boundary distances [S2, unofficial].

## Subtopics

### [GPU Backprojection](5843-gpu-backprojection.md)
GPU Backprojection is a rendering pass that visualizes detected scale-space blobs by generating small slices for each blob, computing weighted contributions, and blending them to form the final image, followed by normalization. It is distinct from standard ray-casting as it operates on a sparse abstraction of the volume (scale maxima) rather than the full voxel grid, decoupling the expensive scale-space computation from the frequent backprojection step. It builds on [Backward Projection](1491-backward-projection.md).

### [Ray Segment Projection to Image](7631-ray-segment-projection-to-image.md)
This method computes the bounding box of a ray segment within the normalized image plane $[0, 1] \times [0, 1]$ for backward projection. It determines the valid range of ray parameters ($t_{min}, t_{max}$) and the corresponding 2D image coordinates ($xy_{min}, xy_{max}$) where the ray intersects the image frame. It sets itself apart from sibling methods by explicitly handling near/far clipping boundaries and validity masking to ensure only rays overlapping the image frame contribute to the projection, avoiding unnecessary computation for rays missing the frame entirely.

### [Cone Beam Forward Projection with Trilinear Interpolation](8247-cone-beam-forward-projection-with-trilinear-interpolation.md)
Computes the projection of a 3D volume onto a 2D detector array for cone beam geometry by ray-casting through the volume. It uses trilinear interpolation to sample the volume along rays defined by source and detector positions. It simulates cone-beam projections by reading volumes in a specific bricked layout with a 1-voxel zero border.

### [Circular Integral Convolution](8507-circular-integral-convolution.md)
Circular Integral Convolution generates a blurred image from a scan by integrating ray values across all angles for each point, effectively filtering the true image with a $1/r$-weighted kernel. It serves as a pre-processing step for deconvolution-based reconstruction rather than a full reconstruction method itself. It differs from [GPU Backprojection](5843-gpu-backprojection.md) by explicitly constructing the convolution filter weights for subsequent deconvolution, and from [Ray Segment Projection to Image](7631-ray-segment-projection-to-image.md) by operating in the back-projection domain to create a spatially blurred result rather than projecting segments directly to image space.

### Visibility Thresholding for Scale Backprojection
A method to prevent scale information from being backprojected to voxels that do not correspond to the target feature (e.g., vessels), improving classification accuracy by using a visibility parameter [S4]. A voxel is determined to be not visible from the Laplacian of Gaussian (LoG) extrema if, at some point along the ray connecting the two positions, the intensity is below the threshold $\tau$ [S4]. Alternatively, thresholding can be based on gradient magnitude [S4]. The parameter $\tau$ (tau) is a visibility parameter used to control the sensitivity of the visibility test, based on scalar value or gradient magnitude [S4]. This is a fast approximation; more accurate results can be obtained with a region growing approach added as a post-process based on the initial size-based classification [S4].

## Operating characteristics
- **Time Complexity:** $O(P \cdot S)$, where $P$ is the number of pixels and $S$ is the average number of samples per ray.
- **Memory Complexity:** $O(H \cdot W \cdot C)$ for the image buffer, plus $O(N_x \cdot N_y \cdot N_z)$ for the volume data.
- **Performance:** Efficient for high-resolution images as it avoids over-sampling. However, it can be slower than forward projection for sparse volumes because it processes every pixel, even those that might be empty.
- **Strengths:** Simple blending logic, easy to implement with hardware acceleration, handles opacity well.
- **Limitations:** Can be computationally expensive for large images with deep volumes. Not ideal for sparse volumes where most rays do not intersect the data.

## Background
Backward projection, also known as backward mapping, was introduced as part of early volume rendering techniques. It is often contrasted with forward projection (texture mapping). The method was popularized in the context of image-order rendering, where the focus is on computing the color of each pixel by casting rays through the volume [S2, unofficial]. It is a fundamental technique in medical imaging and scientific visualization.

## Sources
[S1] Image-based volume rendering, https://openalex.org/W2514442513, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S3] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S4] PDF Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
