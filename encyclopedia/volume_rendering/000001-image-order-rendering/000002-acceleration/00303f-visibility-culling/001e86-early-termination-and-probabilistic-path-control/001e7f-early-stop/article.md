# Early Stop

Also called: Early stop+0, Early stop+1

## Overview
Early Stop (ES) is an optimization for sphere tracing procedural volumes that limits the computational cost of evaluating noise frequencies by terminating the summation loop early based on the ray's position relative to a Bounding Volume Hierarchy (BVH) [S1]. It solves the problem of excessive function evaluations in detailed procedural noise by recognizing that higher-frequency octaves contribute negligibly when the ray is far from the isosurface [S1]. Unlike standard sphere tracing which computes all octaves, ES trades a small potential increase in step count for a drastic reduction in per-step function evaluations, with variants like Early Stop+1 offering the best balance between performance and convergence speed [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, isovalue $v$, BVH structure where each node $i$ provides an upper bound function $BVH_{i+}(\mathbf{p})$ and lower bound function $BVH_{i-}(\mathbf{p})$, and the noise octave functions $f_i(\mathbf{p})$.
*   **Output:** Intersection point $\mathbf{p}$ and normal $\mathbf{n}$, or null if no intersection is found.

**Data Structures:**
*   **BVH:** A hierarchy of bounding volumes. Each level $i$ corresponds to a specific frequency octave $f_i$. The BVH provides inclusion functions (bounds) for the noise function at that frequency. Specifically, $BVH_{i+}(\mathbf{p})$ is the upper bound of $f_i(\mathbf{p})$ and $BVH_{i-}(\mathbf{p})$ is the lower bound.
*   **Noise Functions:** $f_i(\mathbf{p})$ represents the $i$-th octave of the procedural noise (e.g., FBM).

**Algorithm Steps:**
1.  **Initialize:** Set current position $\mathbf{p} = \mathbf{o}$ and step size $t = 0$.
2.  **Loop:** While $t$ is within the ray length limit:
    a.  **Find Minimum Index ($i_{min}$):** Determine the first BVH level $i$ such that the position $\mathbf{p}$ is outside the bounding volume for that level. This is defined as:
        $$ i_{\min} = \min\{i \mid BVH_{i+}(\mathbf{p}) < v\} $$
        If the ray is inside the volume and searching for the isosurface, this can equivalently be defined using the lower bound:
        $$ i_{\min} = \min\{i \mid BVH_{i-}(\mathbf{p}) > v\} $$
        [S1].
    b.  **Compute Octaves:** Evaluate the noise functions $f_i(\mathbf{p})$ starting from $i=0$ up to $i = i_{\min} + k$, where $k$ is the "Early Stop" parameter (typically $k=1$).
    c.  **Calculate Step Size:** Compute the step size $t_{step}$ based on the accumulated noise value $F(\mathbf{p}) = \sum_{i=0}^{i_{\min}+k} f_i(\mathbf{p})$ and its derivative (if using Taylor-based sphere tracing) [S1].
    d.  **Advance Ray:** Update position $\mathbf{p} \leftarrow \mathbf{p} + t_{step} \cdot \mathbf{d}$ and $t \leftarrow t + t_{step}$.
    e.  **Check Convergence:** If $|F(\mathbf{p}) - v| < \epsilon$ (convergence threshold), return $\mathbf{p}$ and $\mathbf{n}$. If the step size is too small or max iterations reached, terminate.
3.  **Return:** Null or failure.

**Parameters:**
*   **$k$:** The number of additional octaves to compute beyond $i_{min}$. Typical values are $k=0$ (Early Stop+0), $k=1$ (Early Stop+1), or $k>1$ [S1].
*   **$v$:** The isovalue used for comparison against BVH bounds [S1].

**Pitfalls and Edge Cases:**
*   **Early Stop+0 ($k=0$):** Computing only the minimum required octaves causes slow convergence at every level of the BVH traversal. The ray steps safely to the current outer bounding volume and repeats, massively increasing the total number of steps. This is particularly detrimental for "grazing rays" (rays passing close to but not entering a bounding volume) [S1].
*   **Step Size Reduction:** Stopping early reduces the number of $f_i$ computed, which may result in smaller step sizes. This can increase the total number of steps required to converge, potentially raising the final computational cost [S1].
*   **Diminishing Returns:** Using $k > 1$ provides diminishing returns. While it slightly increases step size and makes convergence issues less likely, the added cost of computing more $f_i$ is usually not worth the small gains compared to $k=1$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def early_stop_sphere_trace(ray_origin, ray_direction, isovalue, bvh, k=1, max_steps=1000):
    """
    Perform sphere tracing with Early Stop optimization.
    
    Args:
        ray_origin: Initial point p.
        ray_direction: Normalized direction d.
        isovalue: v, the target isovalue.
        bvh: Object providing BVH_i_plus(p) and BVH_i_minus(p) for each level i.
        k: Number of extra octaves to compute beyond i_min.
        max_steps: Maximum number of ray march steps.
        
    Returns:
        Intersection point or None.
    """
    p = ray_origin
    t = 0.0
    
    for step in range(max_steps):
        # 1. Find i_min: first level where position is outside the bounding volume
        i_min = 0
        while True:
            # Check upper bound for current level i
            # If BVH_i_plus(p) < v, then p is outside the bounding volume for level i
            if bvh.bvh_i_plus(i_min, p) < isovalue:
                break
            i_min += 1
            
        # 2. Compute noise functions up to i_min + k
        # Note: In a real implementation, f_i might be computed incrementally or cached.
        # Here we assume a function compute_noise(p, max_octave) that sums f_0 to f_max_octave.
        # However, to be precise with the "Early Stop" definition, we only compute up to i_min + k.
        # We must compute all f_i from 0 to i_min + k to get the correct sum F(p).
        
        # Optimization: If we are reusing previous computations, we might skip some,
        # but the core definition is breaking the loop at i_min + k.
        
        # Compute the sum of octaves
        # F(p) = sum(f_i(p) for i in 0..i_min+k)
        # In practice, f_i might be derived from a base noise function.
        # For this pseudocode, we assume a generic noise evaluator.
        noise_value = 0.0
        for i in range(i_min + k + 1):
            noise_value += bvh.noise_octave(i, p)
            
        # 3. Calculate step size
        # Standard sphere tracing step size is based on the distance to the isosurface.
        # For FBM, this is often approximated or uses Taylor inclusion functions.
        # Here we use a simplified step size calculation based on the noise value.
        # A more robust implementation would use the derivative for Taylor sphere tracing.
        # Assuming a simple distance estimate:
        dist = abs(noise_value - isovalue)
        
        # Safety check for step size
        if dist < 1e-6:
            return p # Convergence reached
            
        # Step size is typically proportional to the distance to the surface
        # In sphere tracing, t_step = dist / |gradient| or similar.
        # For procedural noise, a common heuristic is t_step = dist * scale_factor.
        # The source notes that step size might get smaller with fewer fi.
        # We use a simplified step size for demonstration.
        t_step = dist * 0.5 # Heuristic factor
        
        # 4. Advance ray
        p = p + t_step * ray_direction
        t += t_step
        
        # Check if we are too far or too close to origin
        if t > 100.0: # Arbitrary max distance
            return None
            
    return None
```

## Subtopics

### [Early Stop + k](1938-early-stop-k.md)
A variant where the sum is computed up to $F_{i_{min}+k}$. It trades a small increase in per-step function evaluations for a reduction in total BVH traversal steps. ES+1 is generally recommended as it balances performance and convergence, avoiding the slow convergence of ES+0 and the diminishing returns of ES+k for $k>1$.

## Operating characteristics
*   **Time Complexity:** The cost per step is reduced from $O(N)$ (where $N$ is total octaves) to $O(i_{min} + k)$. However, the total number of steps may increase due to smaller step sizes, especially with $k=0$ [S1].
*   **Memory Complexity:** Requires storage for the BVH structure and noise functions. No significant additional memory compared to standard sphere tracing [S1].
*   **Performance:** Early Stop+1 drastically diminishes performance loss from grazing rays at negligible cost compared to standard sphere tracing. Early Stop+0 causes massive increases in total steps and is inefficient [S1].
*   **Strengths:** Reduces computational cost of noise evaluation in detailed scenes. Effective for procedural volumes with hierarchical noise structures (FBM) [S1].
*   **Limitations:** May increase the number of ray march steps. Requires a BVH structure compatible with the noise function's frequency octaves [S1].
*   **When to Use:** When rendering procedural volumetric noise with many octaves, especially in large and detailed scenes where noise evaluation is a bottleneck [S1].
*   **When Not to Use:** When the scene is simple or the noise has few octaves, the overhead of BVH traversal may outweigh the benefits. Avoid Early Stop+0 due to poor convergence [S1].

## Background
Early Stop was introduced in the context of fast sphere tracing of procedural volumetric noise for very large and detailed scenes [S1]. It leverages the hierarchical nature of Bounding Volume Hierarchies (BVH) applied to noise functions, specifically Fractional Brownian Motion (FBM), to avoid unnecessary computations of high-frequency octaves when the ray is far from the isosurface [S1]. The method addresses the inefficiency of standard sphere tracing, which computes all octaves at every step, by recognizing that higher frequencies are bounded by their BVH nodes and can be skipped if the ray is outside the corresponding bounding volume [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
