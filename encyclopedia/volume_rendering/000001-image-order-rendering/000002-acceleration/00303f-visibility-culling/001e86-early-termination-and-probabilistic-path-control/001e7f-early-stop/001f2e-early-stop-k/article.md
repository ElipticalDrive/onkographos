# Early Stop + k

Also called: ES+k

## Overview
Early Stop + k (ES+k) is an optimization for sphere tracing procedural volumes that prevents unnecessary intermediate convergence checks during BVH traversal by computing $k$ additional octaves per traversal step [S1]. It builds on [Early Stop](1790-early-stop.md) by trading a small increase in per-step function evaluations for a significant reduction in the total number of BVH traversal steps [S1]. Unlike standard Early Stop (ES+0), which may converge too slowly and incur high step counts, ES+k maintains a balance between step reduction and computational overhead, with ES+1 generally offering the best performance across diverse scenes [S1].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, BVH structure, noise function $f$, maximum distance $d_{max}$, and integer parameter $k \ge 0$ [G].
- Output: Transmittance and color along the ray, or termination status [G].

**Data Structures:**
- The method relies on the BVH hierarchy [G]. No new data structures are introduced; it modifies the evaluation logic within the existing traversal loop [G].

**Parameters:**
- $k$: The number of additional octaves computed per BVH node visit [S1].
  - Typical values: $0, 1, 2, 3$ [S1].
  - Recommended default: $k=1$ [S1].
  - Note: Optimal $k$ varies by scene and view; fixed $k > 1$ is risky [S1].

**Algorithm:**
The core modification occurs during the BVH traversal loop [G]. For each step in the BVH traversal, the algorithm computes the function value for $k$ additional octaves beyond the current level, rather than just the current level or none [S1]. This look-ahead ensures that the distance estimate is sufficiently tight to avoid premature convergence checks that would otherwise force a re-evaluation or slow convergence [G].

1. Initialize ray parameters: $t = 0$, $d_{max}$, $k$ [G].
2. While $t < d_{max}$:
    a. Traverse BVH to find the smallest bounding volume containing the point $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [G].
    b. Let $L$ be the octave level associated with the current BVH node [G].
    c. Compute the noise function value $V$ for octaves $L, L+1, \dots, L+k$ [S1].
    d. Determine the step size $\Delta t$ based on the tightest bound from these $k+1$ octaves [G].
    e. Update $t \leftarrow t + \Delta t$ [G].
    f. Perform convergence check (if applicable) or continue traversal [G].
3. Return accumulated radiance/transmittance [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_trace_early_stop_k(ray_origin, ray_direction, bvh, noise_func, k, max_dist):
    """
    Sphere tracing with Early Stop + k optimization.
    
    Args:
        ray_origin: (x, y, z)
        ray_direction: (x, y, z) normalized
        bvh: BVH structure with nodes containing octave levels
        noise_func: Function returning noise value for a given octave and point
        k: Integer, number of additional octaves to compute per step
        max_dist: Maximum ray length
        
    Returns:
        transmittance, color
    """
    t = 0.0
    transmittance = 1.0
    color = (0.0, 0.0, 0.0)
    epsilon = 1e-6  # [G] Standard minimum step size guard
    
    while t < max_dist:
        # Traverse BVH to find the smallest bounding volume containing point [G]
        # (Implementation depends on specific BVH library; assume node retrieval here)
        point = (ray_origin[0] + t * ray_direction[0],
                 ray_origin[1] + t * ray_direction[1],
                 ray_origin[2] + t * ray_direction[2])
        node = bvh.find_node_containing(point)
        
        if node is None:
            # Ray exited the volume
            break
            
        # Get the base octave level for this node
        base_octave = node.octave_level
        
        # Compute k additional octaves beyond the current level
        # This prevents unnecessary intermediate convergences [S1]
        min_step = float('inf')
        
        for i in range(k + 1):
            current_octave = base_octave + i
            # Evaluate noise function at current point for current octave
            noise_val = noise_func(point, current_octave)
            
            # Calculate step size based on the bound provided by this octave
            # Standard sphere tracing uses the noise value as the step bound [G]
            step = max(noise_val, epsilon)
            
            # Take the minimum step size to ensure safety
            if step < min_step:
                min_step = step
        
        # Guard against zero or negative step sizes
        if min_step <= epsilon:
            min_step = epsilon
            
        t += min_step
        
        # Accumulate radiance (simplified standard volume rendering integration) [G]
        density = noise_func(point, base_octave)
        color = (color[0] + density * transmittance * min_step,
                 color[1] + density * transmittance * min_step,
                 color[2] + density * transmittance * min_step)
        transmittance *= (1.0 - density * min_step)
        
    return transmittance, color
```

**Edge Cases and Pitfalls:**
- **Choosing $k$**: A fixed $k > 1$ is tricky because different scenes (even views of the same object) require different optimal $k$ values [S1]. For example, on the "Sponge" scene, $k=2$ is optimal, while on "Sponge (far)", $k=1$ is better [S1].
- **Performance Degradation**: If $k$ is too high, performance decreases due to excessive function evaluations per step [S1]. For instance, $k=3$ on the Sponge scene resulted in significantly higher time (3513.6 ms) compared to $k=1$ (749.6 ms) [S1].
- **ES+0 Performance**: $k=0$ (standard Early Stop) can perform poorly due to slow convergence and increased number of steps, sometimes worse than no Early Stop at all [S1].

## Operating characteristics
**Measured Performance:**
- **Sponge Scene (Sub-stepping enabled):**
  - ES+0 ($k=0$): 273.0 avg steps, 3.65 avg octaves, 972.0 ms, ×1.0 speedup [S1].
  - ES+1 ($k=1$): 147.8 avg steps, 5.23 avg octaves, 749.6 ms, ×1.4 speedup [S1].
  - ES+2 ($k=2$): 127.3 avg steps, 6.31 avg octaves, 782.5 ms, ×1.3 speedup [S1].
  - ES+3 ($k=3$): 279.6 avg steps, 13.45 avg octaves, 3513.6 ms, ×1.2 speedup [S1].
  - No ES: 116.2 avg steps, 10.22 avg octaves, 1123.6 ms, reference [S1].
- **Sponge (far) Scene (Sub-stepping enabled):**
  - ES+0 ($k=0$): 273.0 avg steps, 3.65 avg octaves, 972.0 ms, ×1.0 speedup [S1].
  - ES+1 ($k=1$): 147.8 avg steps, 5.23 avg octaves, 749.6 ms, ×1.4 speedup [S1].
  - ES+2 ($k=2$): 127.3 avg steps, 6.31 avg octaves, 782.5 ms, ×1.3 speedup [S1].
  - Note: Performance decreases for $k > 1$ on this scene [S1].

**Scaling:**
- ES+1 consistently outperforms ES+3 and no Early Stop across tested scenes [S1].
- Using $k > 1$ was never found to be faster when using quadratic bounds [S1].

**Strengths and Limitations:**
- **Strengths**: Reduces the number of BVH traversal steps, which can be a bottleneck in large scenes [S1]. ES+1 provides a good balance between step reduction and computational overhead [S1].
- **Limitations**: Requires tuning $k$ for specific scenes; a fixed $k > 1$ is not universally optimal [S1]. ES+0 can be slower than no Early Stop in some cases [S1].

**When to Use:**
- Use ES+1 as a default choice for most scenes [S1].
- Consider tuning $k$ for specific scenes if performance is critical, but be aware that optimal $k$ varies [S1].
- Avoid ES+0 if slow convergence is observed [S1].
- Avoid $k > 1$ if using quadratic bounds [S1].

## Background
**Origin:**
- The method was introduced in the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1].
- It is part of a series of optimizations for sphere tracing procedural volumes, including Early Stop and quadratic Taylor bounds [S1].

**History and Development:**
- The authors observed that standard Early Stop (ES+0) could converge slowly, leading to increased step counts and worse performance than no Early Stop in some cases [S1].
- They proposed computing additional octaves per step to improve convergence speed, leading to the ES+k strategy [S1].
- Experimental results showed that ES+1 provided the best performance across various scenes, while higher $k$ values could degrade performance [S1].

**Where it is used:**
- The method is applicable to sphere tracing of procedural volumetric noise, particularly in large and detailed scenes where BVH traversal is a significant cost [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
