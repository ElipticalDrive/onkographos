# Alpha Compositing Ray Casting

Also called: Alpha Compositing with White Background, Front-to-back compositing, Front-to-back ray casting, Volume Compositing, Volumetric Compositing, White Background Compositing

## Overview
Alpha Compositing Ray Casting projects a semi-transparent 3D scalar field onto a 2D image plane by integrating optical properties along viewing rays, eliminating the need for binary classification of the data [S3, unofficial]. It differs from Maximum Intensity Projection (MIP) by preserving internal texture and visibility through opacity-weighted accumulation rather than selecting only the highest intensity value [S4, unofficial].

## How it works
The method discretizes the volume rendering integral into a summation over sample points along a ray, performing front-to-back compositing [S2, S5, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `volume_texture`: 3D array or texture storing scalar voxel values [S4, S5, unofficial].
    *   `camera`: Defines ray origins and directions for each pixel [S1, S4, unofficial].
    *   `transfer_function`: Maps scalar values to RGB color; opacity is often derived directly from the sampled scalar value rather than a separate alpha channel [S4, S5, unofficial]. Implemented as a 2D texture in WebGL contexts [S4, unofficial].
    *   `step_size`: Fixed distance between samples, or derived from volume dimensions [S4, S5, unofficial].
    *   `threshold`: Opacity threshold for early ray termination (typically 0.95–0.99) [S2, S4, unofficial].
*   **Outputs**:
    *   `image`: 2D array of RGB values representing the final rendered frame [S2, unofficial].

### Data Structures
*   **Volume Texture**: 3D array of scalar values [S4, S5, unofficial]. Sampled using trilinear interpolation or nearest neighbor [S4, S5, unofficial].
*   **Transfer Function**: 2D texture where the x-coordinate is the scalar value and y-coordinate is fixed (e.g., 0.5) [S4, unofficial]. Output is RGB.
*   **Accumulators**:
    *   `accumulated_color` ($C'$): `vec3`, initialized to $(0, 0, 0)$ [G].
    *   `accumulated_opacity` ($\alpha'$): `float`, initialized to $0.0$ [G].

### Steps
1.  **Ray Initialization**: For each pixel, compute ray origin $\bar{o}$ and direction $\bar{d}$ [G].
2.  **Bounding Box Intersection**: Intersect the ray with the volume bounds $[0,1]^3$ to find entry $t_0$ and exit $t_1$ [S5, unofficial]. If $t_0 > t_1$, discard the pixel. If the eye is inside the volume, clamp $t_0$ to $0.0$ [S5, unofficial].
3.  **Step Size Calculation**: Compute step size $dt$ based on volume dimensions $V_{dim}$ and ray direction $\bar{d}$:
    $$ dt = \min \left( \frac{1}{V_{dim,x} |d_x|}, \frac{1}{V_{dim,y} |d_y|}, \frac{1}{V_{dim,z} |d_z|} \right) $$
    [S5, unofficial].
4.  **Ray Marching Loop**: Iterate from $t = t_0$ to $t < t_1$ with increment $dt$:
    a.  **Sample**: Evaluate scalar value $s$ at position $\bar{p} = \bar{o} + t\bar{d}$ [S5, unofficial].
    b.  **Transfer Function**: Map $s$ to color $\bar{c}_i$; opacity $\alpha_i$ is typically the sampled scalar value itself [S4, S5, unofficial].
    c.  **Compositing**: Update accumulators:
        $$ C'_i = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i \bar{c}_i $$
        $$ \alpha'_i = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i $$
        [S2, S5, unofficial].
    d.  **Early Termination**: If $\alpha' \ge \text{threshold}$ (typically 0.95 [S4, unofficial] or 0.99 [S2, unofficial]), break the loop.
5.  **Gamma Correction**: Apply gamma correction (typically 2.2) to the final accumulated color:
    $$ \bar{C}_{final} = \bar{C}'_{last}^{1/2.2} $$
    [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alpha_compositing_raycast(volume_texture, camera, transfer_function, volume_dims, step_size=None, threshold=0.99):
    """
    Renders a volume using front-to-back alpha compositing.
    
    Args:
        volume_texture: 3D array or texture object supporting trilinear sampling.
        camera: Object with method cast_ray(x, y) returning (origin, direction).
        transfer_function: Function or 2D texture mapping scalar value -> (color, opacity).
        volume_dims: Tuple (dim_x, dim_y, dim_z) of the volume grid.
        step_size: Fixed marching distance. If None, derived from volume_dims.
        threshold: Opacity threshold for early ray termination.
        
    Returns:
        image: 2D list of RGB values.
    """
    import math
    H, W = camera.height, camera.width
    image = [[0.0, 0.0, 0.0] for _ in range(H)]
    
    for y in range(H):
        for x in range(W):
            ray_origin, ray_direction = camera.cast_ray(x, y)
            # Normalize ray direction
            length = math.sqrt(sum(d*d for d in ray_direction))
            ray_direction = [d/length for d in ray_direction]
            
            accumulated_color = [0.0, 0.0, 0.0]
            accumulated_opacity = 0.0
            
            if step_size is None:
                dir_abs = [max(abs(d), 1e-6) for d in ray_direction]
                dt_vec = [1.0 / (volume_dims[i] * dir_abs[i]) for i in range(3)]
                step_size = min(dt_vec)
                
            inv_dir = [1.0 / d if abs(d) > 1e-6 else 1e6 for d in ray_direction]
            box_min = [0.0, 0.0, 0.0]
            box_max = [1.0, 1.0, 1.0]
            tmin_tmp = [(box_min[i] - ray_origin[i]) * inv_dir[i] for i in range(3)]
            tmax_tmp = [(box_max[i] - ray_origin[i]) * inv_dir[i] for i in range(3)]
            tmin = [min(tmin_tmp[i], tmax_tmp[i]) for i in range(3)]
            tmax = [max(tmin_tmp[i], tmax_tmp[i]) for i in range(3)]
            t0 = max(tmin)
            t1 = min(tmax)
            
            if t0 > t1:
                continue
            t0 = max(t0, 0.0)
            
            t = t0
            
            while t < t1 and accumulated_opacity < threshold:
                pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
                scalar_value = volume_texture.sample_trilinear(pos)
                color, opacity = transfer_function(scalar_value)
                
                accumulated_color[0] += (1.0 - accumulated_opacity) * opacity * color[0]
                accumulated_color[1] += (1.0 - accumulated_opacity) * opacity * color[1]
                accumulated_color[2] += (1.0 - accumulated_opacity) * opacity * color[2]
                accumulated_opacity += (1.0 - accumulated_opacity) * opacity
                
                t += step_size
                
            image[y][x] = [c ** (1.0/2.2) for c in accumulated_color]
            
    return image
```

### Parameters and Edge Cases
*   **Threshold**: Typically 0.95 [S4, unofficial] or 0.99 [S2, unofficial]. Higher thresholds improve accuracy but increase computation [G].
*   **Numerical Stability**: In some variants, a small epsilon ($\epsilon = 10^{-10}$) is added to the product term to prevent instability [S6, unofficial].
*   **Zero Direction Components**: If a ray direction component is zero, the step size calculation must clamp it to a small positive value to avoid division by zero [G].
*   **Transfer Function Alpha**: Some implementations ignore the transfer function's alpha channel and use the sampled scalar value directly as opacity [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of samples per ray. $M$ depends on volume depth and step size [G].
*   **Memory Complexity**: $O(V)$ for the volume texture, where $V$ is the number of voxels [G].
*   **Performance**: Real-time rendering is achievable with hardware acceleration (e.g., WebGL fragment shaders) [S4, unofficial]. Software implementations may be slow without optimization [S1, unofficial].
*   **Strengths**: Preserves internal structure and texture better than MIP [S4, unofficial]. No binary classification required, reducing false positives/negatives [S3, unofficial].
*   **Limitations**: Computationally expensive for large volumes or high resolutions. Sensitive to transfer function design.

## Background
The technique was popularized by Drebin et al. [S3, unofficial], who introduced volumetric compositing to eliminate binary classification artifacts. Early software implementations were slow, taking several seconds per frame, but later hardware frameworks enabled real-time rendering [S3, unofficial]. It is widely used in medical imaging and scientific visualization [G].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S3] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S4] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S5] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S6] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S7] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S8] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S9] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
