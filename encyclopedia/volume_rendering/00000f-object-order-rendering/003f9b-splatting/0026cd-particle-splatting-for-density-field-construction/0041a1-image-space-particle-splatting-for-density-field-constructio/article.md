# Image-Space Particle Splatting for Density Field Construction

Also called: Image-Space Volume Visualization for Dynamic Particle Simulations, Image-space approach, Perspectively correct glyph raycasting

## Overview
This method visualizes dynamic particle simulations by reconstructing a density field in image space using perspectively correct splatting and ray casting, avoiding the resolution limits and geometric smoothing artifacts of object-space uniform grids [S1, unofficial]. It differs from its parent [Particle Splatting for Density Field Construction](3916-particle-splatting-for-density-field-construction.md) by evaluating the density volume directly in the image buffer via multiple render targets rather than constructing a 3D grid on the CPU, thereby preserving sharp geometric features and reducing memory usage for high-resolution views [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Particle positions $p_j$ and masses $m_j$ stored in a GPU vertex buffer [S1, unofficial].
- Camera parameters (eye position $e$, viewing direction $v$) and bounding box of the particle system [S1, unofficial].
- Kernel support radius $h$ [S1, unofficial].

**Outputs:**
- Final composited image of the frame [S1, unofficial].
- Feedback data (number of active rays) for adaptive sub-pass control [S1, unofficial].

### Data Structures
1.  **Ray Parameter Textures:** Floating point textures (32-bit precision) storing ray direction, length, and current sampling position for each pixel. Dimensions equal the viewport resolution [S1, unofficial].
2.  **Density Slice Textures:** Floating point textures (16-bit precision) storing density values. Multiple render targets (MRT) are used to store several density slices simultaneously [S1, unofficial].
3.  **Particle Vertex Buffer:** Stores $p_j$ and $m_j$ for all particles, updated once per frame [S1, unofficial].
4.  **Depth Buffer:** Used to terminate rays that step behind the bounding box or exceed opacity thresholds [S1, unofficial].

### Parameters
-   $h$: Support radius of the kernel function [S1, unofficial].
-   $\lambda_{min}$: Start parameter of the ray (front face of bounding box) [S1, unofficial].
-   $\lambda_{max}$: End parameter of the ray (back face of bounding box) [S1, unofficial].
-   $\Delta\lambda$: Distance between density slices [S1, unofficial].
-   $s_{max}(f)$: Maximum number of sub-passes for frame $f$ [S1, unofficial].
-   Typical values: Viewport $512 \times 512$, $\Delta\lambda = 1/256$ [S1, unofficial].

### Method Steps

1.  **Initialization (Frame $f=0$):**
    Calculate the maximum required sub-passes to sample the entire bounding box:
    $$ s_{max}(0) = \max_{\forall rays} \left\lceil \frac{-\lambda_{min}}{\lambda_{max} \Delta \lambda} \right\rceil $$
    [S1, unofficial].

2.  **Particle Sorting (Optional Optimization):**
    Sort particle indices according to their image-space depth using a Radix Sort on the GPU (CUDA) [S1, unofficial]. This allows determining the range of particles contributing to each density slice in $O(n)$ time on the CPU, or via parallel reduction in $O(m \log(n))$ for $m$ slices [S1, unofficial].

3.  **Culling:**
    Move particles that do not contribute to the current density slice outside the viewing frustum. This requires uniform depth spacing between slices [S1, unofficial].

4.  **Sub-pass Loop (for $s = 0$ to $s_{max}(f)-1$):**
    
    4.1. **Position Update:**
    Update the sampling position $p$ for each active ray:
    $$ \lambda = \lambda_{min} + s \times \Delta\lambda $$
    $$ p = e + \lambda v $$
    [S1, unofficial].
    -   If the new $p$ lies inside the bounding box, mark the ray as active.
    -   If the ray has stepped behind the bounding box, terminate it. The depth test kills fragments for inactive rays [S1, unofficial].

    4.2. **Splatting (Density Evaluation):**
    Splat perspectively correct footprints of the particle kernel functions into the density slice textures (Multiple Render Targets) [S1, unofficial].
    -   The image space size of the point-sprite is determined by $h$ [S1, unofficial].
    -   Use the method of perspectively correct glyph raycasting [S1, unofficial].
    -   Compute density $\bar{\rho}(p)$ at the sampling position $p$ using additive blending:
        $$ \bar{\rho}(p) = \sum_{j} m_j W(|p - p_j|, h) $$
        where $W$ is a symmetric polynomial density kernel (e.g., smooth step function) with support radius $h$ [S1, unofficial].
    -   The first slice is placed at $\lambda$, and subsequent slices are $\Delta\lambda$ apart [S1, unofficial].

    4.3. **Occlusion Query (Adaptive Control):**
    Issue hardware occlusion queries to count active rays. To minimize overhead, issue six queries during the last few sub-passes of the current frame at sub-pass indices:
    $$ s_{max}(f) - \{50, 20, 5, 2, 1, 0\} $$
    [S1, unofficial].

5.  **Volume Ray Casting and Compositing:**
    Perform direct volume rendering using ray casting on the reconstructed density slices [S1, unofficial].
    -   **Interpolation:** Interpolate between density slices within a stack for a continuous iso-surface. Reuse one slice from the previous sub-pass to interpolate between stacks [S1, unofficial].
    -   **Gradient Computation:** Compute gradients for illumination. For a single particle, the gradient is the vector from $p_j$ to $p$. The overall gradient is the summation of weighted gradients via additive blending [S1, unofficial].
    -   **Termination:** Accumulate opacity along active rays. Terminate rays if accumulated opacity exceeds a threshold or if the depth buffer indicates the ray is behind the bounding box [S1, unofficial].
    -   Composite the result with previous sub-passes to generate the final image [S1, unofficial].

6.  **Display and Feedback Adjustment:**
    Display the image of frame $f$ [S1, unofficial].
    -   Exploit frame-to-frame coherence by postponing the evaluation of occlusion query results until the frame is finished [S1, unofficial].
    -   Collect query results in order. Determine $s_{max}(f+1)$ for the next frame:
        -   Set $s_{max}(f+1)$ to the sub-pass of the first query with too few active rays.
        -   If all queries show too few active rays, reduce: $s_{max}(f+1) = s_{max}(f) / 2$.
        -   If no query has few enough active rays, increase: $s_{max}(f+1) = s_{max}(f) + 25$, limited by $s_{max}(0)$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_frame(f, max_sub_passes, lambda_min, lambda_max, delta_lambda):
    if f == 0:
        # Calculate max sub-passes to cover the bounding box
        max_sub_passes = ceil(-lambda_min / (lambda_max * delta_lambda))
    return max_sub_passes

def update_ray_positions(rays, s, lambda_min, delta_lambda):
    lambda_val = lambda_min + s * delta_lambda
    for ray in rays:
        # Update sampling position
        ray.p = ray.e + lambda_val * ray.v
        
        # Check bounds
        if ray.p is inside bounding_box:
            ray.active = True
        elif ray.p is behind bounding_box:
            ray.active = False
    return rays

def splat_particles(particles, rays, density_textures, h, kernel_func):
    # Use Multiple Render Targets for density slices
    for slice_idx, texture in enumerate(density_textures):
        lambda_val = lambda_min + slice_idx * delta_lambda
        # Determine which rays are active at this lambda
        active_rays = [r for r in rays if r.active and r.p is inside bounding_box]
        
        # Clear texture for this slice
        texture.clear()
        
        # Splat particles into the texture
        # Note: In practice, this is done via GPU rasterization of point sprites
        # Here we simulate the density calculation for clarity
        for ray in active_rays:
            density = 0.0
            for particle in particles:
                # Perspective correction is handled by the rasterizer/glyph raycasting method
                dist = distance(ray.p, particle.p)
                if dist < h:
                    density += particle.mass * kernel_func(dist, h)
            texture.write(ray.pixel_coord, density)

def compute_gradients(particles, rays, gradient_textures, h):
    for slice_idx, texture in enumerate(gradient_textures):
        texture.clear()
        for ray in rays:
            if ray.active:
                grad = vector(0, 0, 0)
                for particle in particles:
                    dist = distance(ray.p, particle.p)
                    if dist < h:
                        # Gradient is vector from particle to sampling position
                        vec = ray.p - particle.p
                        # Weighted by kernel derivative or mass (simplified)
                        grad += vec * particle.mass
                texture.write(ray.pixel_coord, grad)

def ray_cast_and_composite(density_textures, gradient_textures, rays, threshold):
    final_image = empty_image(viewport_size)
    
    for ray in rays:
        if not ray.active:
            continue
            
        accumulated_opacity = 0.0
        color = vector(0, 0, 0)
        
        # Iterate through density slices
        for slice_idx, density_texture in enumerate(density_textures):
            # Interpolate if necessary between slices
            density = density_texture.read(ray.pixel_coord)
            gradient = gradient_textures[slice_idx].read(ray.pixel_coord)
            
            # Volume rendering step (simplified)
            # In practice, this involves stepping along the ray between slices
            # and accumulating color/opacity based on density and gradient (for lighting)
            opacity_contribution = 1.0 - exp(-density)
            color_contribution = compute_lighting(gradient, opacity_contribution)
            
            color += color_contribution * (1.0 - accumulated_opacity)
            accumulated_opacity += opacity_contribution * (1.0 - accumulated_opacity)
            
            if accumulated_opacity > threshold:
                break
                
        final_image.write(ray.pixel_coord, color)
        
    return final_image

def adjust_sub_passes(query_results, current_max_sub_passes):
    # query_results is a list of active ray counts for the sampled sub-passes
    # Find the first sub-pass with too few active rays
    for i, count in enumerate(query_results):
        if count < threshold:
            return query_results[i].sub_pass_index
            
    # If all have too few, reduce
    if all(count < threshold for count in query_results):
        return max(1, current_max_sub_passes / 2)
        
    # If none have too few, increase
    return min(current_max_sub_passes + 25, initial_max_sub_passes)
```

### Edge Cases and Pitfalls
-   **Perspective Correction:** Splatting must be perspectively correct to avoid distorted or cube-like appearances of the filter kernel in image space [S1, unofficial].
-   **Non-Uniform Depth Slices:** Culling efficiency is nullified if density slices do not have uniform depth values, as determining contributing particles becomes computationally expensive [S1, unofficial].
-   **Latency:** Hardware occlusion queries introduce latency. Results are not immediately available; frame-to-frame coherence is exploited to hide this latency by adjusting $s_{max}$ for the next frame [S1, unofficial].
-   **Precision:** Ray parameters use 32-bit floats, density values use 16-bit floats. Insufficient precision may affect accuracy [S1, unofficial].

## Operating characteristics
-   **Time Complexity:**
    -   Sorting particles: $O(n)$ with Radix Sort on CPU/GPU for all slices, or $O(m \log(n))$ for $m$ slices using parallel reduction [S1, unofficial].
    -   Splatting: Proportional to the number of active particles and pixels.
    -   Ray Casting: Proportional to the number of active rays and slices.
-   **Memory Complexity:**
    -   Ray parameters: $8 \text{ MB}$ for a $512 \times 512$ window [S1, unofficial].
    -   Density slices: $12 \text{ MB}$ for 8 slices in a $512 \times 512$ window [S1, unofficial].
    -   Total: $\approx 20 \text{ MB}$, compared to $256 \text{ MB}$ for a full $512^3$ object-space volume [S1, unofficial].
-   **Performance:**
    -   Tested on Intel Core2 2.4 GHz, NVIDIA GeForce 280 GTX [S1, unofficial].
    -   Laser ablation dataset (742,141 particles) rendered at $1500 \times 450$ at $0.7 \text{ fps}$ [S1, unofficial].
-   **Strengths:**
    -   Avoids resolution limits of uniform grids; conveys all geometric features without smoothing sharp corners [S1, unofficial].
    -   Lower memory footprint for high-resolution views [S1, unofficial].
    -   Adaptive sub-pass count reduces computation for sparse scenes [S1, unofficial].
-   **Limitations:**
    -   Requires GPU hardware for occlusion queries and MRTs.
    -   Sorting overhead may not be beneficial for small datasets [S1, unofficial].
    -   Non-uniform depth slicing negates culling benefits [S1, unofficial].

## Background
The method was developed by M. Falk, S. Grottel, and T. Ertl [S1, unofficial]. It is used for visualizing signal transduction in cells, protein surfaces (SES/MSS approximation), and laser ablation molecular dynamics simulations [S1, unofficial]. It builds on the concept of metaballs for density field definition and uses perspectively correct glyph raycasting presented by Klein et al. [KE04] [S1, unofficial].

## Sources
[S1] Interactive Image-Space Volume Visualization for Dynamic Particle Simulations, https://openalex.org/W2284871597, fetched 2026-10-30 (unofficial)
