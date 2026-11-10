# SDF Scaling and Early Ray Termination for Fast Sphere Tracing

Also called: SDF Scaling, Two-pass SDF Scaling

## Overview
This method accelerates sphere tracing by using a two-pass approach: a low-resolution pre-pass that applies Signed Distance Function (SDF) scaling and early termination to generate a coarse distance buffer, followed by a full-resolution pass that uses this buffer to initialize ray origins closer to surfaces [S1]. It distinguishes itself from sibling methods like [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) and [Sphere Tracing](1674-sphere-tracing.md) by explicitly modifying the SDF geometry in the pre-pass to prevent missed small features and reduce step counts, while avoiding the memory overhead of pyramidal buffers used in [Inverse Pyramid Rendering](1080-standard-constant-step-ray-marching.md) [S1].

## How it works
The method operates in two distinct passes. The first pass renders at a reduced resolution to create a "head start" for the second pass. The second pass performs full-resolution sphere tracing, initializing rays based on the distance data from the first pass.

### Inputs and Outputs
*   **Inputs**:
    *   Scene definition containing implicit surfaces defined by SDFs $f(\mathbf{p})$.
    *   Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$).
    *   Target resolution $R_{target}$ (e.g., 3840x2160).
    *   Low-resolution scale factor $s$ (e.g., $s=4$, resulting in $R_{low} = R_{target}/s$).
    *   Convergence threshold $\epsilon_{base}$ (standard for full res).
    *   Early termination threshold $\epsilon_{early}$ (larger than $\epsilon_{base}$).
    *   Shadow hardness parameter $k$ (optional, for shadow rays).
*   **Outputs**:
    *   Final rendered image at $R_{target}$.
    *   Intermediate low-resolution distance buffer $B_{low}$ of size $R_{low}$.

### Data Structures
*   **Low-Resolution Buffer ($B_{low}$)**: A 2D array of size $W_{low} \times H_{low}$, where $W_{low} = W_{target}/s$ and $H_{low} = H_{target}/s$. Each element stores a scalar distance value $d_{approx}$.
*   **Ray Batch Mapping**: Each pixel $(u_{low}, v_{low})$ in $B_{low}$ corresponds to a $s \times s$ block of pixels in the full-resolution image. For $s=4$, 16 full-resolution rays share the same $d_{approx}$ value.

### SDF Scaling Logic
Before rendering in the low-resolution pass, the SDF values are modified based on the camera's position relative to the implicit surface. This ensures that small objects are not missed due to the lower sampling rate.

1.  **Camera Position Check**: For each ray, evaluate the SDF at the camera origin or a point along the ray to determine if the camera is "in front" (outside) or "behind" (inside) the surface.
    *   Let $d_{cam} = f(\mathbf{o})$.
    *   If $d_{cam} > 0$, the camera is outside.
    *   If $d_{cam} < 0$, the camera is inside.
2.  **Scaling Operation**:
    *   **Outside ($d_{cam} > 0$)**: Inflate the SDF. The effective distance function becomes $f'(\mathbf{p}) = f(\mathbf{p}) + \delta$, where $\delta > 0$ is a scaling offset. This expands the footprint of the surface, ensuring it covers more pixels in the low-res buffer.
    *   **Inside ($d_{cam} < 0$)**: Deflate the SDF. The effective distance function becomes $f'(\mathbf{p}) = f(\mathbf{p}) - \delta$. This contracts the volume.
    *   **Hole Handling**: For SDFs representing holes (created via subtraction operations), the logic is inverted to improve approximation:
        *   If camera is in front ($d_{cam} > 0$), decrease the volume size of the hole-generating SDF (deflate).
        *   If camera is behind ($d_{cam} < 0$), scale up the hole-generating SDF (inflate).

### Early Ray Termination (Pass 1)
During the low-resolution sphere tracing, the convergence threshold is increased to $\epsilon_{early}$, where $\epsilon_{early} > \epsilon_{base}$.
*   **Termination Condition**: The ray marching loop terminates when $f'(\mathbf{p}) < \epsilon_{early}$.
*   **Purpose**: This provides a fast, approximate distance to the surface. It prevents rays from overshooting small features that might be missed at low resolution.
*   **Trade-off**: Higher $\epsilon_{early}$ values cause edges to appear rounded and reduce geometric accuracy [S1].

### Step-by-Step Algorithm

#### Pass 1: Low-Resolution Pre-processing
1.  Initialize $B_{low}$ with $t_{max}$ (or infinity).
2.  For each pixel $(u_{low}, v_{low})$ in $B_{low}$:
    a.  Construct ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ corresponding to the low-res pixel.
    b.  Initialize $t = 0$.
    c.  Loop:
        i.  Evaluate scaled SDF: $d = f'(\mathbf{o} + t\mathbf{d})$.
        ii. If $d < \epsilon_{early}$, terminate. Set $B_{low}[u_{low}, v_{low}] = t$.
        iii. If $t > t_{max}$, terminate. Set $B_{low}[u_{low}, v_{low}] = t_{max}$.
        iv. Advance: $t \leftarrow t + d$.
3.  Store $B_{low}$.

#### Pass 2: Full-Resolution Rendering
1.  For each pixel $(u, v)$ in the target resolution:
    a.  Map to low-res pixel: $u_{low} = \lfloor u / s \rfloor$, $v_{low} = \lfloor v / s \rfloor$.
    b.  Retrieve approximated distance: $d_{approx} = B_{low}[u_{low}, v_{low}]$.
    c.  **Ray Origin Reconstruction**:
        *   If $d_{approx} < t_{max}$:
            *   Set new ray origin $\mathbf{o}' = \mathbf{o} + d_{approx}\mathbf{d}$.
            *   Set initial step $t = 0$ (relative to $\mathbf{o}'$).
            *   *Note*: The ray now starts closer to the surface.
        *   Else:
            *   Set $\mathbf{o}' = \mathbf{o}$, $t = 0$.
    d.  Perform standard sphere tracing from $\mathbf{o}'$ with threshold $\epsilon_{base}$:
        i.  Loop:
            1.  Evaluate SDF: $d = f(\mathbf{o}' + t\mathbf{d})$.
            2.  If $d < \epsilon_{base}$, hit surface. Compute shading. Terminate.
            3.  If $t > t_{max}$, miss. Terminate.
            4.  Advance: $t \leftarrow t + d$.
    e.  **Shadow Rays**:
        *   If shading requires shadows, use $d_{approx}$ to initialize shadow ray origin.
        *   Set shadow hardness $k$. Higher $k$ yields harder shadows.
        *   Trace shadow ray from $\mathbf{o}' + d_{approx}\mathbf{d}$ towards light source.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sphere_trace_sdf_scaling(scene, camera, resolution, scale_factor=4, 
                             epsilon_base=0.001, epsilon_early=0.05, 
                             t_max=100.0, delta_scale=0.1):
    """
    Implements SDF Scaling and Early Ray Termination.
    
    Args:
        scene: Object with method evaluate_sdf(p) returning signed distance.
        camera: Object with origin o and direction d(u, v).
        resolution: Tuple (W, H) for target resolution.
        scale_factor: Integer, ratio of target to low-res resolution (e.g., 4).
        epsilon_base: Convergence threshold for full-res pass.
        epsilon_early: Larger threshold for low-res pass.
        t_max: Maximum ray distance.
        delta_scale: Magnitude of SDF inflation/deflation.
        
    Returns:
        Image buffer at target resolution.
    """
    W, H = resolution
    W_low, H_low = W // scale_factor, H // scale_factor
    
    # Buffer to store approximated distances from low-res pass
    low_res_buffer = [[t_max for _ in range(W_low)] for _ in range(H_low)]
    
    # --- Pass 1: Low-Resolution Pre-processing ---
    for v_low in range(H_low):
        for u_low in range(W_low):
            # Get ray direction for low-res pixel
            d = camera.direction(u_low, v_low)
            o = camera.origin
            
            t = 0.0
            while t < t_max:
                p = o + t * d
                # Determine camera position relative to surface for scaling logic
                # Evaluate at a point slightly ahead or at origin to decide scaling
                # Note: In practice, this check might be done per-pixel or per-ray bundle
                # Here we assume a simple check at the current point or origin
                cam_dist = scene.evaluate_sdf(o)
                
                # Apply SDF Scaling
                f_val = scene.evaluate_sdf(p)
                
                # Scaling Logic
                if cam_dist > 0:
                    # Camera outside: Inflate SDF
                    f_scaled = f_val + delta_scale
                else:
                    # Camera inside: Deflate SDF
                    f_scaled = f_val - delta_scale
                
                # Hole handling would require checking if p is inside a hole SDF
                # For simplicity, we assume standard SDF scaling here.
                # If scene supports hole subtraction, additional logic is needed:
                # if is_hole_sdf and cam_dist > 0: f_scaled = f_val - delta_scale
                # if is_hole_sdf and cam_dist < 0: f_scaled = f_val + delta_scale
                
                # Early Termination
                if f_scaled < epsilon_early:
                    low_res_buffer[v_low][u_low] = t
                    break
                
                # Step
                t += f_scaled
            
            # If loop finished without break, t_max is stored (already initialized)

    # --- Pass 2: Full-Resolution Rendering ---
    image = [[(0, 0, 0) for _ in range(W)] for _ in range(H)]
    
    for v in range(H):
        for u in range(W):
            d = camera.direction(u, v)
            o = camera.origin
            
            # Map to low-res buffer
            u_low = u // scale_factor
            v_low = v // scale_factor
            
            d_approx = low_res_buffer[v_low][u_low]
            
            # Ray Origin Reconstruction
            if d_approx < t_max:
                o_prime = o + d_approx * d
                t = 0.0
            else:
                o_prime = o
                t = 0.0
            
            # Full-Resolution Sphere Tracing
            while t < t_max:
                p = o_prime + t * d
                f_val = scene.evaluate_sdf(p)
                
                if f_val < epsilon_base:
                    # Hit surface
                    color = compute_shading(scene, p, d)
                    image[v][u] = color
                    break
                
                t += f_val
            
            if t >= t_max:
                image[v][u] = (0, 0, 0) # Miss
                
    return image
```

### Parameters
*   **Scale Factor ($s$)**: Typically 4 (reducing 4K to 960x540) [S1].
*   **$\epsilon_{early}$**: Must be larger than $\epsilon_{base}$. Exact value depends on scene scale; too high causes rounding artifacts [S1].
*   **$\delta$ (Scaling Offset)**: Not explicitly defined in sources, but described as "scale up" or "scale down" [S1]. Implementation requires a scalar value.
*   **Shadow Hardness ($k$)**: Controls shadow penumbra width. Higher $k$ = harder shadows [S1].

### Edge Cases and Pitfalls
*   **Rounding Artifacts**: High $\epsilon_{early}$ values cause sharp edges to become rounded, reducing geometric fidelity [S1].
*   **Hole Approximation**: Incorrect scaling logic for subtracted SDFs (holes) can lead to visual artifacts. The scaling direction must be inverted for hole-generating primitives relative to the camera position [S1].
*   **Precision**: Low-resolution buffer stores distance values. Precision loss in $B_{low}$ may affect the accuracy of the ray origin reconstruction in Pass 2.

## Operating characteristics
*   **Time Complexity**: The low-resolution pass is $O((W/s) \cdot (H/s))$. The full-resolution pass benefits from reduced step counts because rays start closer to surfaces. In the 'Primitives' scene, shadow rays often terminated after 1 step [S1].
*   **Memory Complexity**: Requires one additional buffer of size $W \cdot H / s^2$. This is significantly less than pyramidal approaches like Inverse Pyramid Rendering [S1].
*   **Performance**:
    *   On NVIDIA RTX-3070Ti, 4K resolution: 3x speedup in frame time compared to state-of-the-art without anti-aliasing [S1].
    *   Shadow ray performance: 2.1x improvement in 'Primitives', 1.52x in 'Columns', 1.8x in 'Greek Temple', and 1.5x in 'Sponza atrium' [S1].
*   **Strengths**: Reduces step count significantly; low memory overhead; enables anti-aliasing without proportional cost increase [S1].
*   **Limitations**: Introduces geometric approximation errors (rounded edges); requires two passes; sensitive to $\epsilon_{early}$ selection [S1].

## Background
This method was proposed in the paper "Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere Tracing" [S1]. It addresses the trade-off between rendering speed and visual quality in sphere tracing, specifically targeting the issue of missed small features at low resolutions and the high cost of full-resolution tracing. It builds on the principles of [Step Size and Termination Rules](4132-step-size-and-termination-rules.md) and improves upon [Sphere Tracing](1674-sphere-tracing.md) by introducing geometric scaling in a pre-pass.

## Sources
[S1] Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere ..., https://diglib.eg.org/items/7f21e58c-639e-4359-bf94-414738612006, fetched 2026-10-27
