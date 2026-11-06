# Early Termination

Also called: Early stopping

## Overview
Early termination is a rendering optimization that halts the integration along a ray once the accumulated opacity exceeds a threshold, thereby skipping samples that contribute negligibly to the final color [S2, unofficial]. It differs from [Early ray termination](1019-early-ray-termination.md) by explicitly defining the termination condition based on the difference between current and previous accumulated color contributions relative to an epsilon threshold, ensuring that the remaining unrendered portion of the ray does not significantly alter the result [S2, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Ray origins, ray directions, radiance field query function (returning RGB and density $\sigma$), maximum sample count ($N_{max}$), near/far planes, step size, and termination epsilon ($\epsilon$).
- **Outputs**: Final pixel color, accumulated opacity, depth, and number of samples used.

**Data Structures**
- **Accumulators**: Floating-point variables for accumulated color $C_{out}$, accumulated opacity $\alpha_{acc}$, and depth $D_{acc}$.
- **Ray Parameters**: $t_{near}$, $t_{far}$, step size $s$.

**Numbered Steps**
1.  **Initialization**: Initialize $C_{out} = (0,0,0)$, $\alpha_{acc} = 0$, $D_{acc} = 0$, and sample counter $n = 0$ [S2, unofficial].
2.  **Loop Condition**: While $n < N_{max}$ and $\alpha_{acc} < 1 - \epsilon$:
    a.  Compute sample position $t = t_{near} + n \cdot s$.
    b.  Query radiance field for color $c$ and density $\sigma$ at $t$ [S1, unofficial].
    c.  Compute transmittance $T = \exp(-\sigma \cdot s)$ [G].
    d.  Compute contribution: $contrib = c \cdot (1 - T)$ [G].
    e.  Update accumulators:
        $$C_{out} \leftarrow C_{out} + contrib \cdot (1 - \alpha_{acc})$$
        $$\alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot (1 - T)$$
        $$D_{acc} \leftarrow D_{acc} + contrib \cdot t \cdot (1 - \alpha_{acc}) / C_{out} \quad \text{(if } C_{out} > 0 \text{)}$$
    f.  Increment $n$.
3.  **Termination Check**: If $\alpha_{acc} \ge 1 - \epsilon$, terminate early [S2, unofficial].
4.  **Background**: If $\alpha_{acc} < 1$, blend with background color $C_{bg}$: $C_{final} = C_{out} + C_{bg} \cdot (1 - \alpha_{acc})$ [G].

**Formulas**
- **Termination Criterion**: The ray terminates when the accumulated opacity satisfies:
  $$\alpha_{acc} > 1 - \epsilon$$
  This is equivalent to checking if the potential color change is less than $\epsilon$ [S2, unofficial].

**Parameters**
- $\epsilon$ (early_stop_eps): Default $10^{-4}$ [S1, unofficial]. Controls precision vs. speed.
- $N_{max}$ (max_samples): Default not specified, but typically determined by scene bounds and step size [S1, unofficial].
- $s$ (render_step_size): Default $10^{-3}$ [S1, unofficial].
- $t_{near}$ (near_plane): Default $0.0$ [S1, unofficial].
- $t_{far}$ (far_plane): Default $10^{10}$ [S1, unofficial].
- $\alpha_{thre}$: Default $0.0$ [S1, unofficial]. Used in occupancy grid culling, not directly in the epsilon termination logic [S1, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: If $\epsilon$ is too large, visible artifacts may appear due to premature termination [S2, unofficial].
- **Transparency**: In highly transparent regions, $\alpha_{acc}$ may never reach $1 - \epsilon$, requiring $N_{max}$ to prevent infinite loops [S1, unofficial].
- **Background**: Proper blending with background color is required when early termination occurs [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray_with_early_termination(ray_origin, ray_direction, radiance_field, 
                                      max_samples, near_plane, far_plane, 
                                      step_size, epsilon, background_color):
    """
    Renders a single ray using early termination based on opacity threshold.
    """
    # Initialize accumulators
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    accumulated_depth = 0.0
    num_samples = 0
    
    # Calculate total distance to cover
    total_distance = far_plane - near_plane
    num_steps = int(total_distance / step_size)
    
    for i in range(num_steps):
        # Check max samples limit
        if num_samples >= max_samples:
            break
            
        # Check termination condition
        if accumulated_opacity >= 1.0 - epsilon:
            break
            
        # Compute sample position
        t = near_plane + i * step_size
        
        # Query radiance field for color and density
        # Note: In practice, this may involve coordinate transformation and grid lookup
        color, density = radiance_field.query(t)
        
        # Compute transmittance
        transmittance = math.exp(-density * step_size)
        
        # Compute contribution
        # Weight is the fraction of light absorbed at this step
        weight = (1.0 - transmittance) * (1.0 - accumulated_opacity)
        
        # Update color accumulator
        accumulated_color[0] += color[0] * weight
        accumulated_color[1] += color[1] * weight
        accumulated_color[2] += color[2] * weight
        
        # Update opacity accumulator
        accumulated_opacity += weight
        
        # Update depth accumulator (weighted average)
        if accumulated_color[0] + accumulated_color[1] + accumulated_color[2] > 1e-8:
            accumulated_depth = (accumulated_depth * (1.0 - weight) + t * weight) / (1.0 - (1.0 - accumulated_opacity))
        
        num_samples += 1
        
    # Blend with background if not fully opaque
    if accumulated_opacity < 1.0:
        accumulated_color[0] += background_color[0] * (1.0 - accumulated_opacity)
        accumulated_color[1] += background_color[1] * (1.0 - accumulated_opacity)
        accumulated_color[2] += background_color[2] * (1.0 - accumulated_opacity)
        
    return accumulated_color, accumulated_opacity, accumulated_depth, num_samples
```

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of samples, but typically much less due to early termination.
- **Memory Complexity**: $O(1)$ per ray.
- **Performance**: Higher $\epsilon$ values reduce rendering time significantly but may introduce artifacts [S2, unofficial]. Lower $\epsilon$ values produce higher quality images at the cost of more samples [S2, unofficial].
- **Strengths**: Simple to implement, effective for opaque or semi-opaque volumes.
- **Limitations**: Less effective for highly transparent volumes where opacity accumulates slowly [S2, unofficial].

## Background
Early termination is a standard technique in direct volume rendering to optimize ray casting performance [S2, unofficial]. It is often used in conjunction with occupancy grids to further cull empty space [S1, unofficial]. The method is widely used in real-time and offline rendering pipelines where performance is critical.

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/presentation/380929057/10-raycasting, fetched 2026-10-25 (unofficial)
