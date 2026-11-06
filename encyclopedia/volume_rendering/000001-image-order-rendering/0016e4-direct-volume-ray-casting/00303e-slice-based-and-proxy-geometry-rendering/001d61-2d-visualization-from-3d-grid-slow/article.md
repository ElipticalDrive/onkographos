# 2D Visualization from 3D Grid (Slow)

Also called: 2D Slice Compositing, Direct Volume Rendering (DVR), Slice Projection, Slice-Based Volume Rendering (Back-to-Front), Volume Visualization, matRad_rayTracing Function

## Overview
This method generates a 2D projection of a 3D scalar volume by processing the data as a series of 2D slices, applying a transfer function to each slice, and compositing them into a 2D image using the over operator [S4, unofficial]. It solves the problem of visualizing medical, scientific, or natural volumetric data (such as fluids, smoke, or fire) without extracting explicit surface geometry [S3, unofficial]. Unlike its sibling [Hybrid Ray Tracer](2308-hybrid-ray-tracer.md) which may use geometry acceleration, or [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) which leverages hardware texture units, this approach explicitly iterates through slices or rays in a software-defined or less-optimized manner, often referred to as "slow" due to the lack of hardware-accelerated ray marching or pre-integration [S1][S4, unofficial]. It builds on the parent [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md) by relying on the volume rendering integral discretized into a summation over sample points along a ray [S3, S5, unofficial].

## How it works
The method operates on a scalar volume $s: \mathbb{R}^3 \rightarrow \mathbb{R}$, discretely sampled on a regular 3D grid where each voxel contains constant data values (scalars like density/opacity) [S3, unofficial]. The rendering process involves casting rays from a camera, sampling the volume along these rays, mapping values to color and opacity via a transfer function, and compositing the results front-to-back [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial].

### Inputs and Outputs
*   **Input**: A 3D scalar volume (CT cube or similar) [S1][S3, unofficial]. Camera parameters including position, orientation, and grid dimensions [S4, unofficial]. A transfer function mapping scalar values to RGBA [S4, unofficial].
*   **Output**: A 2D image array with shape matching the camera grid dimensions (e.g., height $\times$ width $\times$ 3 for RGB) [S4, unofficial].

### Data Structures
*   **Volume Data**: A 3D array of scalars. In some implementations, this may be pre-processed into radiological depth cubes [S1].
*   **Camera Grid**: A structure defining the rays, often represented as a grid of camera positions and directions [S4, unofficial].
*   **Image Buffer**: A 2D array initialized to zeros, storing the accumulated color for each pixel [S4, unofficial].

### Steps
1.  **Ray Initialization**: For each pixel in the output image, define a ray $r(t) = O + tD$ originating from the camera eye point [S1][S4, unofficial]. If using a slice-based iteration, the "rays" are effectively the projection of the camera grid onto the volume slices [S4, unofficial].
2.  **Ray Tracing/Slicing**:
    *   *Ray-based*: Compute the intersection of the ray with the volume bounds to determine $t_{start}$ and $t_{end}$ [G].
    *   *Slice-based*: Iterate through the volume slices defined by the `camera_grid` [S4, unofficial]. The order of iteration determines the compositing direction (front-to-back or back-to-front) [S4, pitfall, unofficial].
3.  **Sampling**:
    *   Sample the volume at the current ray position or slice location.
    *   In ray-based approaches, use trilinear interpolation to sample the scalar value $d_i$ at position $pos$ [S1][S5, unofficial].
    *   In slice-based approaches (e.g., matRad), compute radiological depth values or sample directly from the CT cube [S1].
    *   Apply a fixed random offset to the ray start position to prevent banding artifacts [S18, unofficial].
4.  **Transfer Function Application**:
    *   Map the sampled scalar value $d_i$ to color $C_i$ and opacity $\alpha_i$.
    *   The transfer function may involve preprocessing the input data. For example, in [S4, unofficial], the logarithm of the slice data is used: $x = \log(\text{dataslice})$.
    *   A specific Gaussian mixture transfer function is defined in [S4, unofficial] for input $x$:
        $$
        \begin{aligned}
        r &= 1.0 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(x + 3.0)^2}{0.5}} \\
        g &= 1.0 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 1.0 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(x + 3.0)^2}{0.5}} \\
        b &= 0.1 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 1.0 \cdot e^{-\frac{(x + 3.0)^2}{0.5}} \\
        a &= 0.6 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 0.01 \cdot e^{-\frac{(x + 3.0)^2}{0.5}}
        \end{aligned}
        $$
    *   Clip color and alpha values to the range $[0.0, 1.0]$ [S2, unofficial].
5.  **Compositing**:
    *   Update the accumulated color $C'$ and opacity $\alpha'$ using front-to-back compositing [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial]:
        $$
        \begin{aligned}
        C'_i &= C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i \\
        \alpha'_i &= \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
        \end{aligned}
        $$
    *   Initialize $C'_0 = 0$ and $\alpha'_0 = 0$ [S9, unofficial].
    *   In slice-based implementations, this is performed per-slice: `image[:,:,0] = a*r + (1-a)*image[:,:,0]` for each RGB channel [S4, unofficial].
6.  **Early Termination**:
    *   If $\alpha' \ge 1$ or a threshold (typically 0.99) [S2, S18, unofficial], terminate the ray processing for that pixel [S2, S3, S4, unofficial].
7.  **Post-Processing**:
    *   Clip the final image values to $[0.0, 1.0]$ [S4, unofficial].
    *   Apply gamma correction (typically $\gamma = 2.2$) to the final accumulated color [S9, unofficial].
    *   Convert to uint8 if necessary [S2, unofficial].

### Pseudocode
The following pseudocode combines the ray-based approach (standard DVR) with the specific transfer function and gamma correction details found in the sources. It assumes a standard ray-casting loop.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def vec3(x, y, z):
    return [x, y, z]

def pow_vec3(v, exponent):
    return [v[0]**exponent, v[1]**exponent, v[2]**exponent]

def gaussian_transfer_function(scalar_value):
    """
    Applies the Gaussian mixture transfer function from [S4, unofficial].
    Input scalar_value is expected to be pre-processed (e.g., log) if required by the specific implementation.
    Here we assume the input 'x' in the formula is the scalar_value passed.
    """
    x = scalar_value
    # Gaussian components from [S4, unofficial]
    # Note: The source [S4, unofficial] applies np.log() to the data before calling this function.
    # To match the source exactly, the caller should pass log(data).
    
    # Calculate Gaussian terms
    term1_r = 1.0 * math.exp(-((x - 9.0)**2) / 1.0)
    term1_g = 1.0 * math.exp(-((x - 9.0)**2) / 1.0)
    term1_b = 0.1 * math.exp(-((x - 9.0)**2) / 1.0)
    term1_a = 0.6 * math.exp(-((x - 9.0)**2) / 1.0)
    
    term2_r = 0.1 * math.exp(-((x - 3.0)**2) / 0.1)
    term2_g = 1.0 * math.exp(-((x - 3.0)**2) / 0.1)
    term2_b = 0.1 * math.exp(-((x - 3.0)**2) / 0.1)
    term2_a = 0.1 * math.exp(-((x - 3.0)**2) / 0.1)
    
    term3_r = 0.1 * math.exp(-((x + 3.0)**2) / 0.5)
    term3_g = 0.1 * math.exp(-((x + 3.0)**2) / 0.5)
    term3_b = 1.0 * math.exp(-((x + 3.0)**2) / 0.5)
    term3_a = 0.01 * math.exp(-((x + 3.0)**2) / 0.5)
    
    r = term1_r + term2_r + term3_r
    g = term1_g + term2_g + term3_g
    b = term1_b + term2_b + term3_b
    a = term1_a + term2_a + term3_a
    
    # Clip to [0.0, 1.0] as per [S2, unofficial]
    r = max(0.0, min(1.0, r))
    g = max(0.0, min(1.0, g))
    b = max(0.0, min(1.0, b))
    a = max(0.0, min(1.0, a))
    
    return [r, g, b], a

def trilinear_interpolate(volume, pos, dims):
    """
    Standard trilinear interpolation [G].
    pos is in [0, 1] texture coordinates.
    dims is the size of the volume grid.
    """
    vx = pos[0] * (dims[0] - 1)
    vy = pos[1] * (dims[1] - 1)
    vz = pos[2] * (dims[2] - 1)
    
    x0, y0, z0 = int(math.floor(vx)), int(math.floor(vy)), int(math.floor(vz))
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    
    # Clamp indices
    x0 = max(0, min(x0, dims[0]-1))
    x1 = max(0, min(x1, dims[0]-1))
    y0 = max(0, min(y0, dims[1]-1))
    y1 = max(0, min(y1, dims[1]-1))
    z0 = max(0, min(z0, dims[2]-1))
    z1 = max(0, min(z1, dims[2]-1))
    
    wx = vx - x0
    wy = vy - y0
    wz = vz - z0
    
    # Interpolate
    c00 = volume[x0][y0][z0] * (1-wx) + volume[x1][y0][z0] * wx
    c01 = volume[x0][y0][z1] * (1-wx) + volume[x1][y0][z1] * wx
    c10 = volume[x0][y1][z0] * (1-wx) + volume[x1][y1][z0] * wx
    c11 = volume[x0][y1][z1] * (1-wx) + volume[x1][y1][z1] * wx
    
    r0 = c00 * (1-wy) + c10 * wy
    r1 = c01 * (1-wy) + c11 * wy
    
    return r0 * (1-wz) + r1 * wz

def compute_ray_exit(origin, direction, bounds):
    """
    Compute the exit distance t_end for a ray from an axis-aligned bounding box [G].
    bounds is [[min_x, min_y, min_z], [max_x, max_y, max_z]].
    """
    t0 = float('-inf')
    t1 = float('inf')
    
    for i in range(3):
        if direction[i] != 0:
            t_min = (bounds[0][i] - origin[i]) / direction[i]
            t_max = (bounds[1][i] - origin[i]) / direction[i]
            if t_min > t_max:
                t_min, t_max = t_max, t_min
            t0 = max(t0, t_min)
            t1 = min(t1, t_max)
        else:
            if origin[i] < bounds[0][i] or origin[i] > bounds[1][i]:
                return float('inf') # Ray misses box
    
    return max(t0, t1)

def render_volume(volume_data, camera, volume_bounds, volume_dims, step_size=0.001, threshold=0.99):
    """
    Renders a volume using fixed-step raycasting with front-to-back compositing.
    Implements the logic from [S2, S3, S4, S9, S18, unofficial].
    """
    h, w = camera.screen_height, camera.screen_width
    image = [[vec3(0.0, 0.0, 0.0) for _ in range(w)] for _ in range(h)]
    
    for y in range(h):
        for x in range(w):
            ray_origin, ray_direction = camera.cast_ray(x, y)
            accumulated_color = vec3(0.0, 0.0, 0.0)
            accumulated_opacity = 0.0
            
            # Ray initialization: fixed random offset to prevent banding [S18, unofficial]
            t = random.uniform(0.0, step_size)
            
            # Compute ray exit distance from volume bounds [G]
            t_end = compute_ray_exit(ray_origin, ray_direction, volume_bounds)
            
            # If ray doesn't intersect, skip
            if t_end == float('inf') or t_end <= 0:
                continue

            while accumulated_opacity < threshold and t < t_end:
                # Compute current position in world space
                pos_world = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
                
                # Convert to texture coordinates [0, 1] assuming volume bounds map to [0,1]
                # Note: In practice, volume_bounds might not be [0,1]. This is a simplification [G].
                pos_tex = [
                    (pos_world[0] - volume_bounds[0][0]) / (volume_bounds[1][0] - volume_bounds[0][0]),
                    (pos_world[1] - volume_bounds[0][1]) / (volume_bounds[1][1] - volume_bounds[0][1]),
                    (pos_world[2] - volume_bounds[0][2]) / (volume_bounds[1][2] - volume_bounds[0][2])
                ]
                
                # Sample volume at current position using trilinear interpolation [S1][S5, unofficial]
                scalar_value = trilinear_interpolate(volume_data, pos_tex, volume_dims)
                
                # Apply transfer function.
                # Source [S4, unofficial] applies np.log() before transfer function.
                # We assume the transfer function handles the mapping.
                # If the specific Gaussian TF from [S4, unofficial] is used, the input should be log(scalar_value).
                # Here we pass scalar_value directly, but note the discrepancy.
                # To be safe and generic, we call a generic TF.
                color, opacity = gaussian_transfer_function(math.log(scalar_value + 1e-8)) # +1e-8 to avoid log(0)
                
                # Front-to-back compositing [S2, S5, unofficial]
                # C' = C' + (1 - alpha') * alpha * C
                # alpha' = alpha' + (1 - alpha') * alpha
                
                factor = (1.0 - accumulated_opacity) * opacity
                
                accumulated_color[0] += factor * color[0]
                accumulated_color[1] += factor * color[1]
                accumulated_color[2] += factor * color[2]
                
                accumulated_opacity += factor
                
                t += step_size
                
            # Gamma correction [S9, unofficial]
            image[y][x] = pow_vec3(accumulated_color, 1.0/2.2)
            
    return image
```

### Edge Cases and Pitfalls
*   **Compositing Order**: The order of iteration through slices or samples is critical. If the `camera_grid` or ray samples are not ordered from front to back relative to the viewer, the compositing will be incorrect (back-to-front instead of front-to-back) [S4, pitfall, unofficial].
*   **Banding Artifacts**: Without a random offset at the start of each ray, sample layers can appear as visible bands [S18, unofficial].
*   **Logarithm Input**: Some implementations (e.g., [S4, unofficial]) apply a logarithmic transformation to the scalar data before passing it to the transfer function. Failing to do so when using the specific Gaussian parameters from [S4, unofficial] will yield incorrect colors.
*   **Gradient Discontinuities**: If shading is used, normal discontinuities can cause visible rings. Using a two-voxel offset for gradient computation rather than one-voxel can mitigate this [S18, unofficial].

## Subtopics

### [Grid Visualization with Thresholding](1515-grid-visualization-with-thresholding.md)
Visualizes sparse/noisy data by thresholding voxels and plotting them as a 3D scatter plot. Chosen when data is sparse or noisy.

### [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md)
Accelerates DVR by pre-computing the rendering integral into a 3D texture on the GPU. Chosen for performance on 8-bit scalar domains.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of samples per ray. This is generally slower than hardware-accelerated methods like [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) or [View-Aligned Slicing](6530-view-aligned-slicing.md) because it lacks optimized memory access patterns and hardware compositing units [S1][S4, unofficial].
*   **Memory Complexity**: $O(V)$ for the volume data, where $V$ is the number of voxels. The image buffer is $O(P)$ where $P$ is the number of pixels.
*   **Strengths**: Simple to implement; flexible transfer functions; no need for complex data structures like octrees or grids (though it can use them) [S3, unofficial].
*   **Limitations**: Slow performance for high-resolution volumes; sensitive to sampling rate (aliasing); requires careful handling of compositing order [S4, pitfall, unofficial].
*   **When to Use**: Educational purposes, prototyping transfer functions, or when hardware acceleration is unavailable.
*   **When Not to Use**: Real-time applications or high-resolution medical imaging where performance is critical.

## Background
The method is rooted in the fundamental principles of Direct Volume Rendering (DVR) [S3, unofficial]. The specific implementation details, such as the Gaussian transfer function and slice-based compositing, are drawn from educational and open-source projects like [S4, unofficial] and [S2, unofficial]. The matRad implementation [S1] provides a specific use case for dose distribution visualization in radiotherapy, highlighting the method's application in medical physics. The technique is a foundational approach in scientific visualization, as described in standard courses [S3, unofficial].

## Sources
[S1] rayTracing — matRad 3.2.3 documentation, https://matrad.readthedocs.io/en/stable/api/raytracing.html, fetched 2026-10-25
[S2] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S4] SREESUBBARAYUDU/Parallel_Volume_Rendering, https://github.com/SREESUBBARAYUDU/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
