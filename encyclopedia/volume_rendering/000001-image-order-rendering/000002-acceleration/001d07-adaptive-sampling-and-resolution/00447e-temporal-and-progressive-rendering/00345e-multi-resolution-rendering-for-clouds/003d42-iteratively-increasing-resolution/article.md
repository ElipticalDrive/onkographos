# Iteratively Increasing Resolution

Also called: Multi-resolution rendering strategy

## Overview
Iteratively Increasing Resolution is a rendering strategy that reduces computation time by starting at a low resolution and gradually increasing it to the target final resolution [S1, unofficial]. It solves the problem of expensive high-resolution ray-surface intersection tests by deferring them until necessary, using cone-tracing to guarantee that no surface points are skipped during the resolution transition [S1, unofficial]. Unlike standard adaptive sampling which varies sample counts per pixel, this method varies the spatial resolution of the entire rendering pass, building on the acceleration principles of [Multi-Resolution Rendering for Clouds](7260-multi-resolution-rendering-for-clouds.md) but applied specifically to sphere tracing and SDF rendering.

## How it works
**Inputs:**
- A signed distance function (SDF) or SDF lower estimate (SDFE) $f$.
- Camera parameters (position, direction, resolution).
- Target final resolution $(W_{final}, H_{final})$.
- Initial low resolution $(W_{start}, H_{start})$.

**Outputs:**
- Final rendered image at target resolution.
- State textures storing intermediate rendering data (e.g., distance traveled).

**Data Structures:**
- **Ping-Pong Textures:** Two screen-sized textures are used to store the state for each pixel, such as the distance traveled along the ray ($t$) or accumulated color. These are swapped (ping-ponged) between passes.
- **State Field:** Each pixel stores the current ray parameter $t$ (distance from origin) where the ray currently stands.

**Algorithm Steps:**
1. **Initialization:** Start rendering at the lowest resolution $(W_{start}, H_{start})$. Initialize state textures with $t=0$ (or the ray origin).
2. **Cone-Tracing Pass:** For each pixel, perform sphere tracing using cone-tracing logic. The ray is advanced by a step size $d$ determined by the SDFE value, but bounded by the cone termination condition to ensure no surface is missed when resolution increases.
3. **Termination Check:** A pixel is considered complete if it intersects the surface (SDF value $\approx 0$) or exceeds a maximum distance. Incomplete pixels retain their current $t$ in the state texture.
4. **Resolution Increase:** If the target resolution is not reached, increase the resolution (e.g., double width and height). The starting position for each new pixel in the higher resolution is always the closest pixel in the previous resolution. This ensures continuity.
5. **Incremental Continuation:** In the next frame (or pass), computation continues from the stored $t$ values. If the camera or scene has not changed, the state is preserved. If changed, the process may restart or adjust.
6. **Repeat:** Steps 2-5 are repeated until the target resolution is achieved.

**Cone-Tracing Formula:**
To support resolution increase, cone-tracing ensures that no surface point lies inside the cone defined by the pixel's footprint. The terminating condition for the pixel cone, assuming rectangular pixels, is [S1, unofficial]:

$$ f(p + tv) < \frac{p^2}{\text{width}^2 \text{height}^2} $$

where:
- $p$ is the pixel size (assumed square for simplicity, or derived from aspect ratio).
- $\text{width} \cdot \text{height}$ is the total number of pixels in the current resolution.
- $p$ is typically $1/\text{width}$ or $1/\text{height}$ depending on normalization. The source uses $p^2 / (\text{width}^2 \text{height}^2)$ which simplifies to $(1/\text{width}\cdot\text{height})^2$ if $p=1$. Let $\alpha = \frac{p^2}{\text{width}^2 \text{height}^2}$.

**Modified SDFE for Safe Travel:**
The condition above can be incorporated into the SDFE to avoid special termination checks. Let $\alpha = \frac{p^2}{\text{width}^2 \text{height}^2}$. The modified one-dimensional SDFE $F(t)$ is [S1, unofficial]:

$$ F(t) = \frac{f(p + tv) - \alpha t}{1 + \alpha} $$

This transformation offsets the surface by a value that increases with distance. The Lipschitz constant of $F$ is bounded by $1 + \alpha$, ensuring stability. This allows the algorithm to approximate the safe travel distance without needing a special termination condition for the cone.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iteratively_increasing_resolution(sdf, camera, start_res, target_res, max_passes):
    """
    Renders a scene using iteratively increasing resolution with cone-tracing.
    """
    current_w, current_h = start_res
    # Initialize state textures: store distance t for each pixel
    state_tex = create_texture(current_w, current_h, init_value=0.0)
    next_state_tex = create_texture(current_w, current_h)
    
    for pass_idx in range(max_passes):
        # Calculate alpha for current resolution
        # p is pixel size. Assuming normalized coordinates where pixel size is 1/resolution
        # alpha = (1/current_w)^2 / (current_w^2 * current_h^2) ? 
        # Source says: alpha = p^2 / (width^2 * height^2). 
        # If p is pixel width in world units, and image is normalized, p ~ 1/w.
        # Let's assume p=1 for normalized pixel index, then alpha = 1/(w^2 h^2).
        # However, standard cone tracing uses pixel footprint. 
        # We follow the source formula exactly.
        alpha = 1.0 / (current_w * current_w * current_h * current_h)
        
        # Render pass
        for y in range(current_h):
            for x in range(current_w):
                # Get current ray t from state
                t = state_tex[y, x]
                
                # If pixel is already done (e.g., t > max_dist or surface hit), skip
                if is_complete(state_tex[y, x]):
                    next_state_tex[y, x] = t
                    continue
                
                # Ray origin and direction for this pixel
                ray_o, ray_d = camera.get_ray(x, y)
                
                # Perform sphere tracing with modified SDFE
                # F(t) = (f(ray_o + t*ray_d) - alpha*t) / (1 + alpha)
                # We step until F(t) <= 0 or max iterations
                safe_t = t
                for step in range(max_steps_per_pixel):
                    pos = ray_o + safe_t * ray_d
                    f_val = sdf(pos)
                    
                    # Modified SDFE value
                    f_mod = (f_val - alpha * safe_t) / (1.0 + alpha)
                    
                    if f_mod <= 0:
                        # Surface hit or cone boundary reached
                        break
                    
                    # Step size is f_val (standard sphere tracing)
                    # But we must ensure we don't overstep the cone
                    # The modified SDFE handles the cone constraint implicitly
                    step_size = f_val
                    safe_t += step_size
                    
                    # Check for max distance or convergence
                    if step_size < epsilon:
                        break
                
                next_state_tex[y, x] = safe_t
        
        # Swap textures
        state_tex, next_state_tex = next_state_tex, state_tex
        
        # Check if target resolution reached
        if current_w >= target_res[0] and current_h >= target_res[1]:
            break
            
        # Increase resolution
        current_w *= 2
        current_h *= 2
        # Resize state textures to new resolution
        # "starting position for each new pixel is always the closest pixel in the previous resolution"
        state_tex = resize_texture(state_tex, current_w, current_h, interpolation='nearest')
```

**Edge Cases and Pitfalls:**
- **Precision:** The transformation $F(t)$ offsets the surface by a value that increases with distance. Floating-point precision errors may accumulate over many steps.
- **Boundaries:** The cone-tracing condition assumes rectangular pixels. Non-rectangular pixels or anisotropic scaling may require adjustment of $\alpha$.
- **Degenerate Input:** If the SDF is not a valid lower estimate (Lipschitz constant > 1), the sphere tracing may overstep. The source assumes $Lip f \le 1$.

## Subtopics

### Single-scale Training and Multi-scale Testing Protocol
A training and evaluation protocol designed to simulate zoom-in effects and test anti-aliasing performance across resolutions [S2, unofficial]. It trains a model at a single downsampled resolution and renders at multiple higher scales. Steps: 1) Define dataset's default resolution (e.g., 1352×1014) as full resolution baseline. 2) Downsample training data by a fixed factor (e.g., 4×) and train exclusively on this lower resolution. 3) Evaluate rendering quality at progressively higher resolutions (1×, 2×, and 4× relative to training resolution). 4) When comparing against baselines like Mip-Splatting, integrate the baseline into the pipeline and ensure consistency in maximum sampling frequency computation between both methods [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces total render time by performing most of the ray-surface intersection work at lower resolutions. The cost is proportional to the sum of pixel counts across all resolution levels, which is less than rendering at full resolution from the start.
- **Memory Complexity:** Requires storage for state textures (distance traveled) at the current resolution. Ping-ponging two textures doubles this requirement.
- **Measured Performance:** Implemented in GLSL with C++ test application. Used with relaxed and enhanced sphere tracing. Showed reduced render times for 3D fractals and SDF scenes [S1, unofficial].
- **Strengths:** High-quality rendering for stationary scenes; real-time responsiveness when camera/scene changes (by restarting or incrementing). General applicability to any sphere tracing variant.
- **Limitations:** Requires multiple passes or frames to complete. Not suitable for highly dynamic scenes where state cannot be preserved. Precision issues with the modified SDFE transformation over long distances.
- **When to Use:** Static or slowly changing scenes where high resolution is needed but initial low-res passes can identify complex areas.
- **When Not to Use:** Highly dynamic scenes where state preservation is not feasible. Scenes with very thin structures that might be missed by coarse initial passes (though cone-tracing mitigates this).

## Background
Originated from work on accelerating sphere tracing, introduced by Csaba Bálint and Gábor Valasek in 2018 [S1, unofficial]. Sphere tracing was introduced by Hart in 1994 [S1, unofficial]. The method builds on improvements by Keinert et al. [S1, unofficial]. It is used in real-time rendering of SDF representations, such as 3D fractals, where traditional triangle-based engines are inefficient [S1, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
