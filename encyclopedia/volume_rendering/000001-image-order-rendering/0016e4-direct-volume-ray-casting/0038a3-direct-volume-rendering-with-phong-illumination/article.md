# Direct Volume Rendering with Phong Illumination

Also called: Phong Illumination

## Overview
Direct Volume Rendering with Phong Illumination is a shading technique applied within the Direct Volume Ray Casting pipeline to produce photo-realistic images with strong depth perception and a plastic appearance [S1, unofficial]. It distinguishes itself from non-photorealistic methods (such as Tone or Contour Rendering) by calculating ambient, diffuse, and specular light intensities at each sample point based on the volume gradient, light direction, and view direction [S1, unofficial]. While it offers the highest realism among standard implemented DVR methods, it is computationally expensive due to the requirement of gradient precalculation and vector-based lighting calculations for every sample [S1, unofficial].

## How it works
The method extends the standard Direct Volume Ray Casting pipeline [19-direct-volume-ray-casting.md] by inserting a specific shading calculation between classification and compositing. The pipeline consists of three main steps: Classification, Shading (Phong Illumination), and Compositing [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scalar volume data $V$ (voxel grid).
*   Camera parameters: origin $\mathbf{o}$, direction $\mathbf{d}$, view direction $\mathbf{V}$.
*   Light parameters: normalized light direction $\mathbf{L}$, light color $C^p_\lambda$.
*   Transfer function parameters: mapping scalar values to base color $C^p_\lambda$ and opacity $\alpha$.
*   Phong parameters: ambient $k_a^\lambda$, diffuse $k_d^\lambda$, specular $k_s^\lambda$ coefficients, specular exponent $n$, depth cueing constants $k_1, k_2$.
*   Sampling Distance: defines the equidistant spacing between sample points along the ray.

**Outputs:**
*   Final pixel color $C_{final}$ for each ray.

### Data Structures
The implementation requires storing interpolated values for each sample point $x_i$ along the ray:
1.  **Scalar Value:** $V(x_i)$, obtained via trilinear interpolation of the volume grid.
2.  **Gradient Vector:** $\nabla V(x_i)$, obtained via gradient estimation (see Shading step).
3.  **Surface Normal:** $N(x_i)$, derived from the gradient.
4.  **Base Color:** $C^p_\lambda(x_i)$, derived from the transfer function applied to $V(x_i)$.
5.  **Opacity:** $\alpha(x_i)$, derived from the transfer function applied to $V(x_i)$.
6.  **Accumulated Ray State:** $C^{in}_\lambda(u_i)$ (accumulated color) and $\alpha^{in}(u_i)$ (accumulated opacity), initialized to 0.

### Numbered Steps

1.  **Ray Generation and Sampling:**
    For each pixel, generate a ray $r(t) = \mathbf{o} + t\mathbf{d}$. Intersect with the volume bounding box to find $t_{start}$ and $t_{end}$. Discretize the segment $[t_{start}, t_{end}]$ into $N$ sample points $x_i$ at equidistant intervals defined by the Sampling Distance [S1, unofficial].

2.  **Classification:**
    For each sample point $x_i$:
    *   Interpolate the scalar value $V(x_i)$ from the voxel grid using trilinear interpolation [S1, unofficial].
    *   Apply the transfer function to map $V(x_i)$ to base color $C^p_\lambda(x_i)$ and opacity $\alpha(x_i)$ [S1, unofficial].
    *   Note: The opacity $\alpha(x_i)$ is determined here and remains unchanged during the subsequent shading step [S1, unofficial].

3.  **Shading (Phong Illumination):**
    Calculate the final color $C_\lambda(x_i)$ for the sample point by modifying the base color $C^p_\lambda(x_i)$ using the Phong model with depth cueing [S1, unofficial].
    
    *   **Gradient Estimation:** Compute the surface normal $N(x_i)$. The gradient $\nabla V(x_i)$ is estimated from neighboring voxel values. Two methods are cited:
        *   *Central Differences:* Fast but lower quality. Approximates the first derivative in x, y, and z directions [S1, unofficial].
        *   *4D Linear Regression:* Higher quality with lower approximation error but computationally expensive [S1, unofficial].
        *   The normal is normalized: $N(x_i) = \frac{\nabla V(x_i)}{|\nabla V(x_i)|}$ [S1, unofficial].
    
    *   **Lighting Calculation:** Compute the Phong color using the following formula [S1, unofficial]:
        $$
        C_\lambda(x_i) = \frac{C^p_\lambda k_a^\lambda + \frac{k_d^\lambda (N(x_i) \cdot \mathbf{L}) + k_s^\lambda (N(x_i) \cdot \mathbf{H})^n}{k_1 + k_2 d(x_i)}}{d^\lambda}
        $$
        Where:
        *   $C_\lambda(x_i)$ is the final sample color component $\lambda \in \{r, g, b\}$.
        *   $C^p_\lambda$ is the directional light color component.
        *   $k_a^\lambda, k_d^\lambda, k_s^\lambda$ are the ambient, diffuse, and specular reflection coefficients.
        *   $n$ is the specular highlight exponent.
        *   $k_1, k_2$ are constants for linear depth cueing approximation.
        *   $d(x_i)$ is the perpendicular distance from the image plane to the sample point $x_i$.
        *   $N(x_i)$ is the normalized surface normal.
        *   $\mathbf{L}$ is the normalized light direction vector.
        *   $\mathbf{H}$ is the normalized highlight direction vector.
    
    *   **Highlight Direction:** The vector $\mathbf{H}$ is approximated as the normalized sum of the view direction $\mathbf{V}$ and light direction $\mathbf{L}$ [S1, unofficial]:
        $$
        \mathbf{H} = \frac{\mathbf{V} + \mathbf{L}}{|\mathbf{V} + \mathbf{L}|}
        $$
        Where $\mathbf{V}$ is the normalized view direction (from sample to camera) and $\mathbf{L}$ is the normalized light direction [S1, unofficial].

4.  **Compositing (Front-to-Back):**
    Blend the sample points along the ray in Front-to-Back (FTB) order (from $t_{start}$ to $t_{end}$) [S1, unofficial]. For each sample $i$, update the accumulated ray color $C^{out}_\lambda(u_i)$ and opacity $\alpha^{out}(u_i)$ using the over operator [S1, unofficial]:
    $$
    C^{out}_\lambda(u_i) = C^{in}_\lambda(u_i) + (1 - \alpha^{in}(u_i)) \cdot C_\lambda(x_i) \cdot \alpha(x_i)
    $$
    $$
    \alpha^{out}(u_i) = \alpha^{in}(u_i) + (1 - \alpha^{in}(u_i)) \cdot \alpha(x_i)
    $$
    Where:
    *   $C^{in}_\lambda(u_i)$ and $\alpha^{in}(u_i)$ are the color and opacity accumulated from previous samples (front side).
    *   $C_\lambda(x_i)$ is the shaded color of the current sample.
    *   $\alpha(x_i)$ is the opacity of the current sample.
    *   $C^{out}_\lambda(u_i)$ and $\alpha^{out}(u_i)$ are the updated values passed to the next sample.
    
    Initialize $C^{in}_\lambda(u_0) = 0$ and $\alpha^{in}(u_0) = 0$. The final pixel color is $C^{out}_\lambda(u_N)$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_phong_volume(volume, camera, light, transfer_function, params):
    """
    Renders a volume using Direct Volume Ray Casting with Phong Illumination.
    
    Args:
        volume: 3D array of scalar values.
        camera: Object with origin 'o', direction 'd', and view direction 'V'.
        light: Object with normalized direction 'L' and color 'Cp'.
        transfer_function: Function mapping scalar value to (base_color, opacity).
        params: Dictionary containing:
            - sampling_distance: float
            - k_a: float (ambient coefficient)
            - k_d: float (diffuse coefficient)
            - k_s: float (specular coefficient)
            - n: float (specular exponent)
            - k1: float (depth cueing constant 1)
            - k2: float (depth cueing constant 2)
            - d_lambda: float (distance attenuation factor, assumed constant or 1 if not specified)
            - gradient_method: 'central_diff' or 'linear_regression'
            
    Returns:
        2D image array.
    """
    width, height = camera.resolution
    image = np.zeros((height, width, 3))
    
    # Precompute volume bounds and gradient if necessary
    # Note: Gradient estimation is done per-sample in this implementation 
    # to match the source description of "gradient precalculation... for every single sample point"
    # though optimization usually precomputes gradients on the grid.
    
    for y in range(height):
        for x in range(width):
            # 1. Ray Generation
            ray_origin = camera.o
            ray_dir = camera.get_direction(x, y)
            ray_view_dir = normalize(camera.o - ray_origin) # View direction from sample to camera
            
            # Intersect ray with volume AABB
            t_start, t_end = intersect_aabb(ray_origin, ray_dir, volume.bounds)
            if t_start > t_end:
                continue
                
            # 2. Discretization
            num_samples = int((t_end - t_start) / params['sampling_distance'])
            
            # Accumulators
            acc_color = np.array([0.0, 0.0, 0.0])
            acc_opacity = 0.0
            
            # 3. Front-to-Back Compositing Loop
            for i in range(num_samples):
                t = t_start + i * params['sampling_distance']
                sample_pos = ray_origin + t * ray_dir
                
                # Check if sample is inside volume (safety guard)
                if not volume.contains(sample_pos):
                    continue

                # 4. Classification
                scalar_val = volume.interpolate_trilinear(sample_pos)
                base_color, opacity = transfer_function(scalar_val)
                
                # 5. Shading (Phong Illumination)
                # Estimate Gradient
                if params['gradient_method'] == 'central_diff':
                    grad = estimate_gradient_central_diff(volume, sample_pos)
                else:
                    grad = estimate_gradient_linear_regression(volume, sample_pos)
                
                # Compute Normal
                normal = normalize(grad)
                
                # Compute Highlight Direction
                # L is normalized light direction
                # V is normalized view direction (from sample to camera)
                # Note: In standard Phong, V is from surface to eye. 
                # Source defines V as normalized vector in direction of observer.
                view_dir = normalize(camera.o - sample_pos)
                highlight_dir = normalize(view_dir + light.L)
                
                # Compute Depth Cueing
                # d(xi) is perpendicular distance from image plane to sample point
                # Assuming image plane is at t=0 relative to camera origin for simplicity,
                # or use dot product with camera forward if plane is defined.
                # Source uses d(xi) in denominator.
                depth_dist = t # Approximation if camera origin is at t=0
                
                # Phong Formula
                # C_lambda(xi) = ( Cp_lambda * ka_lambda + ( kd_lambda * (N(xi) * L) + ks_lambda * (N(xi) * H)^n ) / (k1 + k2 * d(xi)) ) / d_lambda
                
                # Dot products
                diff_term = max(0.0, dot(normal, light.L))
                spec_term = max(0.0, dot(normal, highlight_dir)) ** params['n']
                
                # Depth cueing denominator
                depth_denom = params['k1'] + params['k2'] * depth_dist
                
                # Avoid division by zero
                if depth_denom < 1e-6:
                    depth_denom = 1e-6
                    
                # Attenuation factor d_lambda (Source lists d_lambda in denominator of whole expression)
                # If d_lambda is not provided, assume 1.0
                d_lambda = params.get('d_lambda', 1.0)
                if d_lambda < 1e-6:
                    d_lambda = 1e-6

                shaded_color = np.zeros(3)
                for c_idx, c in enumerate(['r', 'g', 'b']):
                    # Extract coefficients for this channel
                    ka = params['k_a'][c_idx] if isinstance(params['k_a'], np.ndarray) else params['k_a']
                    kd = params['k_d'][c_idx] if isinstance(params['k_d'], np.ndarray) else params['k_d']
                    ks = params['k_s'][c_idx] if isinstance(params['k_s'], np.ndarray) else params['k_s']
                    cp = light.Cp[c_idx]
                    
                    # Calculate term
                    ambient = cp * ka
                    diffuse_spec = (kd * diff_term + ks * spec_term) / depth_denom
                    
                    shaded_color[c_idx] = (ambient + diffuse_spec) / d_lambda
                
                # 6. Compositing Update
                # Cout_lambda(ui) = Cin_lambda(ui) + (1 - Ain(ui)) * C_lambda(xi) * alpha(xi)
                # Aout(ui) = Ain(ui) + (1 - Ain(ui)) * alpha(xi)
                
                transmittance = 1.0 - acc_opacity
                acc_color += transmittance * shaded_color * opacity
                acc_opacity += transmittance * opacity
                
                # Early termination if fully opaque
                if acc_opacity >= 1.0:
                    break
            
            image[y, x] = acc_color

    return image

def estimate_gradient_central_diff(volume, pos):
    """Estimates gradient using central differences."""
    # Implementation depends on volume grid coordinates
    # dx = (V(x+1,y,z) - V(x-1,y,z)) / 2
    # dy = (V(x,y+1,z) - V(x,y-1,z)) / 2
    # dz = (V(x,y,z+1) - V(x,y,z-1)) / 2
    # [G] Placeholder for specific grid coordinate mapping logic
    pass

def estimate_gradient_linear_regression(volume, pos):
    """Estimates gradient using 4D linear regression."""
    # [G] Placeholder for 4D linear regression implementation over local voxel neighborhood
    pass
```

### Parameters and Typical Values
*   **Sampling Distance:** Defines the even spacing between sample points. Not given in the sources.
*   **$k_1, k_2$:** Constants for linear depth cueing approximation. Not given in the sources.
*   **$n$:** Specular highlight exponent. Not given in the sources.
*   **$k_a^\lambda, k_d^\lambda, k_s^\lambda$:** Reflection coefficients. Not given in the sources.
*   **$d^\lambda$:** Distance attenuation factor. Not given in the sources.

### Edge Cases and Pitfalls
*   **Gradient Estimation Quality:** The quality of the Phong result is heavily dependent on the gradient estimation method. Central Differences is fast but yields low-quality results. 4D Linear Regression provides higher quality (lower approximation error) but is computationally expensive [S1, unofficial].
*   **Performance:** The method is very expensive in time and calculation consumption compared to other DVR methods because it requires gradient precalculation for every single sample point and costly vector-based light intensity calculations [S1, unofficial].
*   **Opacity Invariance:** The opacity $\alpha(x_i)$ is determined during classification and is *not* altered by the shading step. It remains identical to the classification opacity [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** High. The method requires computing gradients for every sample point and performing vector operations (dot products, normalization) for lighting calculations. It is significantly slower than non-photorealistic methods like Tone or Contour Rendering [S1, unofficial].
*   **Memory Complexity:** Standard for DVR. Requires storage for the volume data and temporary buffers for ray accumulation. Gradient precalculation can be done on-the-fly or precomputed, impacting memory usage [S1, unofficial].
*   **Strengths:** Produces the most photo-realistic image quality among implemented standard DVR methods. Provides good depth effect and an exceptional plastic appearance due to specular highlights and diffuse lighting [S1, unofficial].
*   **Limitations:** Computationally expensive. Requires careful tuning of transfer functions and lighting parameters. Gradient estimation artifacts can degrade visual quality if low-quality methods (like Central Differences) are used [S1, unofficial].
*   **When to use:** When photo-realism and depth perception are prioritized over rendering speed.
*   **When not to use:** When real-time performance is required or when abstract visualization (e.g., contour lines, tone shading) is more appropriate for conveying specific informational contents [S1, unofficial].

## Background
The method is described in the context of implementing importance-driven volume rendering. It is based on the standard rendering pipeline introduced by [4] (referenced in [S1, unofficial]). The specific implementation and analysis of Phong Illumination within DVR are detailed in the work by Thurner et al. [S1, unofficial]. The method is compared against other techniques such as Local Maximum Intensity Projection (LMIP), Tone Rendering, and Contour Rendering to demonstrate its superior realism [S1, unofficial].

## Sources
[S1] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
