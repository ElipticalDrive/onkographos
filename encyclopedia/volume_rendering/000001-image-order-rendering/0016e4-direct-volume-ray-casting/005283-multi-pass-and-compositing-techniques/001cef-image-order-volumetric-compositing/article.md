# Image-Order Volumetric Compositing

Also called: Image-Order Compositing, Levoy's Image-Order Compositing, Levoy's Image-Order Volume Rendering

## Overview
Image-Order Volumetric Compositing is a volume rendering framework proposed by Marc Levoy, originally implemented on the PIXEL-PLANES 5 hardware [S1, unofficial]. It solves the problem of generating high-quality composite projections of volumetric data by decoupling the definition of optical properties (color and opacity) from the ray-casting integration step [S2, unofficial]. Unlike methods that compute shading and compositing simultaneously in a single pass, this approach pre-processes the scalar data into separate color and opacity arrays, allowing for efficient empty space skipping and occlusion culling to achieve real-time or near real-time performance [S1, unofficial].

## How it works
The method operates in two distinct phases: preprocessing to generate optical property arrays, and image-order ray casting to composite the final image.

**Inputs and Outputs**
*   **Input:** A scalar data volume $S$, defined on a 3D grid.
*   **Output:** A 2D image where each pixel contains a single composite color value.

**Data Structures**
The algorithm generates two new 3D arrays from the input scalar array $S$:
1.  **Color Array ($S_c$):** Stores the color value at each grid location.
2.  **Opacity Array ($S_s$):** Stores the opacity (alpha) value at each grid location.

These arrays are defined over the same grid as $S$. The values are determined by interpolation functions $f_c(x,y,z)$ for color and $f_s(x,y,z)$ for opacity, which are derived from $S_c$ and $S_s$ respectively [S2, unofficial].

**Preprocessing Steps**
1.  **Gradient Computation:** Compute the gradient vector $\nabla S$ at each data sample. This is done by partially differentiating the interpolation function $f(x,y,z)$ with respect to $x$, $y$, and $z$ [S2, unofficial].
    *   *Pitfall:* If the interpolation function is not first-derivative continuous, aliasing artifacts occur due to discontinuous normal vectors [S2, unofficial]. A smoother set of gradient vectors can be obtained using central differencing [S2, unofficial].
2.  **Color Array Generation ($S_c$):** Perform a shading operation (e.g., Phong illumination) at each data sample in $S$. The normal vector for shading is the unit gradient vector derived in Step 1. The resulting color values populate $S_c$ [S2, unofficial].
3.  **Opacity Array Generation ($S_s$):** Perform surface classification on $S$ to determine opacity.
    *   For a single iso-surface at constant value $v$ with target opacity $v_{ought}$:
        $$ S_s(x,y,z) = \begin{cases} v_{ought} & \text{if } S(x,y,z) = v \\ 0 & \text{otherwise} \end{cases} $$
    *   To reduce aliasing artifacts, $S_s(x,y,z)$ is set to a value close to $v_{ought}$ if $S(x,y,z)$ is close to $v$.
    *   To maintain a constant transition region thickness, the opacity falls off at a rate inversely proportional to the magnitude of the local gradient vector $|\nabla S|$ [S2, unofficial].
    *   For multiple iso-surfaces, classification mappings are applied separately, and the resulting opacities are combined [S2, unofficial].

**Ray Casting and Compositing Steps**
1.  **Ray Definition:** Cast rays from each pixel through the volume.
2.  **Sampling:** Sample the $S_c$ and $S_s$ arrays at evenly spaced locations along each ray.
3.  **Interpolation:** Determine the color $C$ and opacity $\alpha$ at each sample location using the trilinear interpolation functions $f_c$ and $f_s$ [S2, unofficial].
4.  **Background:** Add a fully opaque background to the ray.
5.  **Compositing:** Composite the sampled values in back-to-front order to produce a single color for the pixel. The standard front-to-back or back-to-front compositing equation is applied:
    $$ C_{out} = C_{sample} \cdot \alpha_{sample} + C_{accumulated} \cdot (1 - \alpha_{sample}) $$
    $$ \alpha_{out} = \alpha_{sample} + \alpha_{accumulated} \cdot (1 - \alpha_{sample}) $$
    (Note: The specific compositing order is back-to-front as stated in the source [S2, unofficial], which implies accumulating from the farthest sample to the nearest).

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_order_volumetric_compositing(volume_S, resolution, num_samples_per_ray):
    """
    Implements Levoy's Image-Order Volumetric Compositing.
    
    Args:
        volume_S: 3D array of scalar data samples.
        resolution: (width, height) of the output image.
        num_samples_per_ray: Number of evenly spaced samples per ray.
        
    Returns:
        output_image: 2D array of RGB colors.
    """
    # Preprocessing Phase
    # 1. Compute Gradient Array
    # Compute partial derivatives of the interpolation function f(x,y,z)
    # using central differencing to ensure first-derivative continuity.
    gradient_array = compute_gradient(volume_S)
    
    # 2. Generate Color Array S_c
    # For each voxel, compute normal = normalize(gradient_array[voxel])
    # Apply Phong illumination model using normal and light direction
    S_c = apply_shading(volume_S, gradient_array)
    
    # 3. Generate Opacity Array S_s
    # Define classification function for iso-surface v with opacity v_ought
    v = 0.5  # Example iso-value
    v_ought = 1.0
    
    S_s = zeros_like(volume_S)
    for x, y, z in grid_indices(volume_S):
        val = volume_S[x, y, z]
        grad_mag = magnitude(gradient_array[x, y, z])
        
        # Transition region logic: opacity falls off inversely proportional to gradient magnitude
        # Simplified: if close to v, assign opacity based on distance and grad_mag
        if abs(val - v) < threshold:
            # Approximate constant transition thickness
            # Opacity is v_ought scaled by inverse gradient magnitude factor
            S_s[x, y, z] = v_ought * (1.0 / (1.0 + grad_mag)) 
        else:
            S_s[x, y, z] = 0.0
            
    # Ray Casting Phase
    output_image = zeros((resolution[0], resolution[1], 3))
    
    for pixel_x in range(resolution[0]):
        for pixel_y in range(resolution[1]):
            # Define ray origin and direction for this pixel
            ray_origin, ray_direction = get_ray(pixel_x, pixel_y)
            
            # Initialize accumulated color and alpha
            # Add fully opaque background
            acc_color = background_color # e.g., (0, 0, 0)
            acc_alpha = 1.0
            
            # Sample along ray in back-to-front order
            # Note: Back-to-front means starting from farthest point along ray
            for i in range(num_samples_per_ray):
                # Calculate sample position
                t_far = max_distance
                t_near = 0.0
                t = t_far - (i * (t_far - t_near) / num_samples_per_ray)
                sample_pos = ray_origin + ray_direction * t
                
                # Check if sample is within volume bounds
                if not is_inside_volume(sample_pos):
                    continue
                    
                # Interpolate color and opacity from precomputed arrays
                color_sample = trilinear_interpolate(S_c, sample_pos)
                alpha_sample = trilinear_interpolate(S_s, sample_pos)
                
                # Composite back-to-front
                # C_out = C_sample * alpha_sample + C_acc * (1 - alpha_sample)
                # alpha_out = alpha_sample + alpha_acc * (1 - alpha_sample)
                
                # However, standard front-to-back is more common for accumulation.
                # Source specifies back-to-front. 
                # If back-to-front, we accumulate from far to near.
                # The formula remains the same, but the order of application changes.
                # Let's assume standard compositing equation applied in back-to-front order.
                
                acc_color = color_sample * alpha_sample + acc_color * (1 - alpha_sample)
                acc_alpha = alpha_sample + acc_alpha * (1 - alpha_sample)
                
                # Optimization: Empty space skipping and occlusion culling
                # If acc_alpha is 1.0, break early (occlusion culling)
                if acc_alpha >= 1.0:
                    break
            
            output_image[pixel_x, pixel_y] = acc_color
            
    return output_image

def compute_gradient(volume):
    # Central differencing for first-derivative continuity
    # Returns 3D vector field
    pass

def apply_shading(volume_S, gradient_array):
    # Phong illumination model
    # Normal = normalize(gradient)
    # Color = ambient + diffuse + specular
    pass
```

**Parameters**
*   $S$: Array of data samples [S2, unofficial].
*   $S_c$: Array defining color at each grid location [S2, unofficial].
*   $S_s$: Array defining opacity at each grid location [S2, unofficial].
*   $f(x,y,z)$: Interpolation function for sample value [S2, unofficial].
*   $f_c(x,y,z)$: Interpolation function for color (transfer function) [S2, unofficial].
*   $f_s(x,y,z)$: Interpolation function for opacity (transfer function) [S2, unofficial].
*   $v$: Constant iso-surface value [S2, unofficial].
*   $v_{ought}$: Target opacity for the iso-surface [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Aliasing:** Discontinuous normal vectors due to non-first-derivative continuous interpolation functions cause aliasing artifacts [S2, unofficial].
*   **Transition Region:** If the opacity fall-off is not inversely proportional to the gradient magnitude, the transition region thickness will vary, leading to visual inconsistencies [S2, unofficial].
*   **Multiple Iso-surfaces:** Must be handled by separately applying classification mappings and combining opacities [S2, unofficial].

## Operating characteristics
*   **Performance:** The system achieved real-time or near real-time performance on the PIXEL-PLANES 5 hardware [S1, unofficial].
*   **Acceleration:** Uses empty space skipping and occlusion culling to accelerate rendering speed [S1, unofficial].
*   **Complexity:** Not given in the sources.
*   **Strengths:** Decouples shading and compositing, allowing for efficient hardware acceleration and preprocessing.
*   **Limitations:** Requires significant preprocessing to generate $S_c$ and $S_s$ arrays. Memory usage increases due to storing these additional arrays.

## Background
*   **Origin:** Proposed by Marc Levoy [S1, unofficial].
*   **Implementation:** Implemented on the PIXEL-PLANES 5 hardware, which consisted of 32 independently programmable 20-MFLOPS graphics processors, 16 independently programmable renderers with 1/4 million pixel processors, a 512x512 pixel color frame buffer, and a 640 Mb/sec ring network [S1, unofficial].
*   **History:** The algorithm was described in Levoy's work on image-order volumetric compositing [S2, unofficial]. It was one of the early frameworks to demonstrate real-time volume rendering capabilities using specialized hardware [S1, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
