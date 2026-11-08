# Early Stop

Also called: Early stop+0, Early stop+1

## Overview
Early Stop (ES) is an optimization for sphere tracing procedural volumes that limits the computation of noise frequencies ($f_i$) by breaking the evaluation loop as soon as the ray position is determined to be outside the bounding volume of the current frequency level [S1]. It solves the problem of unnecessary function evaluations when a ray has moved sufficiently far from the isosurface, thereby reducing the computational cost per step [S1]. Unlike standard sphere tracing which computes all octaves up to $n$ at every step, ES dynamically determines the minimum necessary octave $i_{min}$ based on BVH bounds, trading potential step size reduction for significant savings in function evaluations [S1].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Procedural volume defined by a sum of noise functions $F_n(\mathbf{p}) = \sum_{i=0}^{n} f_i(\mathbf{p})$.
- Isovalue $v$ defining the isosurface.
- BVH structure where each level $i$ provides an upper bound function $BVH_{i+}(\mathbf{p})$ and optionally a lower bound $BVH_{i-}(\mathbf{p})$.
- Parameter $k$: the number of additional frequencies to compute beyond the minimum required (typically $k=1$).

**Outputs:**
- Intersection distance $t$ along the ray.
- Normal $\mathbf{n}$ at the intersection point.

**Data Structures:**
- **BVH Levels:** A hierarchy where level $i$ corresponds to the bounding volume of the noise function $f_i$. Each level $i$ provides a cheap-to-evaluate inclusion function $BVH_{i+}(\mathbf{p})$ which bounds the range of $f_i(\mathbf{p})$.
- **Frequency Functions:** $f_i(\mathbf{p})$, typically octaves of a noise function (e.g., FBM), which are more expensive to evaluate than the BVH bounds.

**Algorithm Steps:**
1. **Initialize:** Start sphere tracing from the ray origin. Let current position be $\mathbf{p}$.
2. **Determine Minimum Index ($i_{min}$):** For the current position $\mathbf{p}$, find the smallest index $i$ such that the upper bound of the BVH at that level is strictly below the isovalue $v$. This indicates that $f_i(\mathbf{p})$ (and consequently the total sum, assuming monotonicity or specific bounding properties) cannot contribute to the isosurface at this location in the same way higher frequencies might, or more precisely, that the position is outside the bounding volume of $f_i$.
   $$ i_{min} = \min\{i \mid BVH_{i+}(\mathbf{p}) < v\} $$
   If searching for the isosurface from inside the volume, this can equivalently be defined using the lower bound: $BVH_{i-}(\mathbf{p}) > v$ [S1].
3. **Compute Frequencies:** Evaluate the noise functions $f_i(\mathbf{p})$ starting from $i=0$ up to $i_{min} + k$.
   - Compute $BVH_{i+}(\mathbf{p})$ for each $i$ as you compute $f_i(\mathbf{p})$.
   - If $BVH_{i+}(\mathbf{p}) < v$, the position is outside the bounding volume for level $i$.
   - Stop evaluating further $f_i$ functions once $i > i_{min} + k$.
4. **Calculate Step Size:** Use the computed sum $F_{i_{min}+k}(\mathbf{p})$ to determine the step size $\Delta t$ for sphere tracing. The step size is typically derived from the distance to the isosurface estimate provided by the current frequency sum.
5. **Advance Ray:** Update position $\mathbf{p} \leftarrow \mathbf{p} + \Delta t \cdot \mathbf{d}$.
6. **Check Termination:** If the ray has exited the volume or converged to the isosurface within a tolerance, return the result. Otherwise, go to Step 2.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def early_stop_sphere_trace(ray_origin, ray_dir, isovalue, k, max_steps):
    p = ray_origin
    for step in range(max_steps):
        # Step 1: Find i_min
        i_min = 0
        while i_min < n:
            # Compute BVH upper bound for level i_min
            bvh_upper = compute_bvh_upper_bound(i_min, p)
            if bvh_upper < isovalue:
                break
            i_min += 1
        
        # Step 2: Compute frequencies up to i_min + k
        # Note: In practice, one might compute f_i incrementally if using FBM
        current_sum = 0.0
        for i in range(i_min + k + 1):
            noise_val = compute_noise_function(i, p)
            current_sum += noise_val
            
        # Step 3: Determine step size
        # The step size is based on the distance to the isosurface
        # For sphere tracing, this is often related to the value of the function
        # Assuming a standard sphere tracing step size calculation:
        # dist_to_surface = |current_sum - isovalue| / |gradient|
        # However, for procedural noise, a simplified step size is often used:
        # delta_t = |current_sum - isovalue| * scale_factor
        # Or more commonly in sphere tracing of implicit surfaces:
        # delta_t = |F(p) - v| / max_gradient_estimate
        
        # Let's assume a generic step size function that uses the computed sum
        delta_t = compute_step_size(current_sum, isovalue, p)
        
        # Safety check to prevent infinite loops
        if delta_t < epsilon:
            break
            
        # Step 4: Advance
        p = p + delta_t * ray_dir
        
        # Step 5: Check convergence
        if abs(compute_noise_function(n, p) - isovalue) < tolerance:
            return p, compute_normal(n, p)
            
    return None, None
```
*Note: The exact step size calculation depends on the specific sphere tracing implementation and the properties of the noise function. The key aspect of Early Stop is limiting the number of `compute_noise_function` calls to $i_{min} + k$.* [S1]

**Parameters:**
- $k$: The number of additional frequencies to compute beyond $i_{min}$. Typical value is $k=1$ [S1].
- $v$: Isovalue for the isosurface [S1].

**Edge Cases and Pitfalls:**
- **Early Stop+0 ($k=0$):** Computing only $f_{i_{min}}$ causes slow convergence at every level of the BVH traversal. The ray steps safely to the current outer bounding volume and repeats, massively increasing the total number of steps. This is particularly bad for grazing rays (rays passing close to but not entering a bounding volume) [S1].
- **Step Size Reduction:** Stopping early reduces the number of $f_i$ computed, which may result in smaller step sizes. This can increase the total number of steps required to converge, potentially raising the final cost. A trade-off must be found [S1].
- **Grazing Rays:** Rays passing very close to a bounding volume without entering it can be significantly slowed down if $k$ is too small. Early Stop+1 mitigates this by considering one extra level of the BVH [S1].

## Subtopics

### [Early Stop + k](1938-early-stop-k.md)
Early Stop + k (ES+k) is an optimization for sphere tracing procedural volumes that prevents unnecessary intermediate convergence checks during BVH traversal by computing $k$ additional octaves per traversal step. It builds on [Early Stop](1790-early-stop.md) by trading a small increase in per-step function evaluations for a significant reduction in the total number of BVH traversal steps. Unlike standard Early Stop (ES+0), which may converge too slowly and incur high step counts, ES+k maintains a balance between step reduction and computational overhead, with ES+1 generally offering the best performance across diverse scenes.

## Operating characteristics
- **Time Complexity:** The cost per step is reduced from $O(n)$ function evaluations to $O(i_{min} + k)$. However, the total number of steps may increase due to smaller step sizes, especially if $k$ is small. The overall performance gain depends on the scene complexity and the distribution of $i_{min}$ along the ray path [S1].
- **Memory Complexity:** Minimal additional memory is required, primarily for storing the BVH structure and the current state of the ray marching. The BVH bounds are computed on the fly or stored in a compact structure [S1].
- **Measured Performance:** Early Stop+1 ($k=1$) provides results very similar to full sphere tracing (computing all $f_i$ up to $n$) but with significantly reduced computational cost. It drastically diminishes the performance loss associated with grazing rays and slow convergence seen in Early Stop+0 [S1].
- **Strengths:** Reduces the number of expensive noise function evaluations. Effective for large and detailed scenes where many parts of the volume are empty or far from the isosurface.
- **Limitations:** May increase the total number of steps if step sizes become too small. The benefit diminishes for $k > 1$ as the added cost of computing more $f_i$ is not worth the small gains in step size [S1].
- **When to Use:** Use when rendering procedural volumetric noise with sphere tracing, especially for large scenes with complex BVH structures. Use $k=1$ for the best balance between performance and accuracy. Avoid $k=0$ due to severe performance degradation on grazing rays.

## Background
Early Stop was introduced in the context of fast sphere tracing of procedural volumetric noise for very large and detailed scenes [S1]. It leverages the hierarchical structure of the noise function (often represented as a BVH) to avoid unnecessary computations. The method addresses the inefficiency of standard sphere tracing, which evaluates all octaves at every step, regardless of whether they are necessary for the current position [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
