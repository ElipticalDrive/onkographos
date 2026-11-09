# FBM Sphere Tracing with BVH Acceleration

Also called: High-level view algorithm of how step size is computed during the FBM loop, Procedural FBM Sphere Tracing

## Overview
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by computing valid Sphere Distance Bounds (SDB) and adaptive step sizes using a dynamic Bounding Volume Hierarchy (BVH) derived from the FBM structure itself [S1]. It reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness, by progressively evaluating octaves while bounding the remaining potential contribution to determine if the total function value can exceed the isosurface threshold [S1]. Unlike naive sphere tracing which evaluates the full FBM at every step, this approach supports features like Adaptive Lipschitz constants, Sub-stepping, Early Stop+k, and Quadratic Taylor inclusion functions [S1].

## How it works
The algorithm computes a step size and the final FBM value by iterating through the octaves of the noise function. It maintains a running sum of the FBM value ($F$), the derivative along the ray ($dF$), and the accumulated step size. At each octave, it constructs a bounding volume for the remaining octaves to determine if further evaluation is necessary or if a larger step can be taken.

**Inputs:**
*   $\mathbf{p}$: Current 3D position.
*   $n$: Total number of octaves in the FBM sum.
*   $v(\cdot)$: Threshold function (hypertexture), typically approximated by its 1st order affine Taylor polynomial [S1].
*   $\mathbf{u}$: Normalized ray direction vector.
*   $k$: Early Stop threshold parameter.
*   $\lambda_{noise}, \gamma_{noise}$: Lipschitz constants for the base noise function (e.g., Perlin noise). Default values depend on the specific noise implementation [S1].

**Outputs:**
*   $F$: Final FBM value.
*   $step$: Computed step size for the ray march.

**Data Structures:**
*   $F$: Scalar, accumulated FBM value.
*   $dF$: Scalar, accumulated derivative along the ray direction.
*   $step$: Scalar, accumulated step size.
*   $BVH$: Scalar bounding volume representing the maximum possible contribution of remaining octaves.
*   $inclFuncParams$: Tuple $(BVH, \lambda, dF, \gamma)$ passed to the intersection function.

**Parameters:**
*   $n$: Typical value 20 (as used in Sponge scene tests) [S1].
*   $k$: Typical values tested: 0, 1, 2, 3 [S1].
*   $\lambda, \gamma$: Global Lipschitz constants initialized as $\lambda_{Fn}$ and $\gamma_{Fn}$ [S1].

**Algorithm Steps:**
1.  **Initialization**: Set $F \leftarrow 0$, $dF \leftarrow 0$, $step \leftarrow 0$. Set global Lipschitz constants $\lambda \leftarrow \lambda_{Fn}$ and $\gamma \leftarrow \gamma_{Fn}$ [S1].
2.  **Loop**: For each octave $i$ from 1 to $n$:
    a.  **Accumulate Value**: Add the current octave value to the sum: $F \leftarrow F + o_i(\mathbf{p})$ [S1].
    b.  **Accumulate Derivative**: Add the derivative of the current octave along the ray direction: $dF \leftarrow dF + (\mathbf{u} \cdot o'_i(\mathbf{p}))$ [S1].
    c.  **Compute BVH**: Calculate the bounding volume for the remaining octaves: $BVH \leftarrow F + s(i+1, n)$, where $s(i+1, n)$ is the sum of the maximum possible contributions of octaves $i+1$ to $n$ [S1].
    d.  **Determine Step Computation**: Check if step size computation is required. It is true if it is the last octave ($i == n$) OR if the ray is outside the bounding volume ($BVH - v(0) < 0$) [S1].
    e.  **Compute Step Size**: If step computation is required:
        i.  **Adaptive Lipschitz**: Replace global constants with local values for the current octave: $\lambda \leftarrow \lambda_{Fi}$, $\gamma \leftarrow \gamma_{Fi}$ [S1].
        ii. **Prepare Parameters**: Set $inclFuncParams \leftarrow (BVH, \lambda, dF, \gamma)$ [S1].
        iii. **Intersect**: Compute the current step size: $curr\_step \leftarrow intersect(inclFuncParams, v)$ [S1].
        iv. **Accumulate Step**: $step \leftarrow step + curr\_step$ [S1].
        v.  **Sub-stepping Guard**: If Sub-stepping is not enabled, enforce minimum step size: $step \leftarrow \max(step, curr\_step)$ [S1].
    f.  **Sub-stepping**: If Sub-stepping is enabled:
        i.  Move ray position: $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{u} \cdot curr\_step$ [S1].
        ii. Update sum: $F \leftarrow v(curr\_step)$ [S1].
        iii. Update threshold: $v \leftarrow v$ at $curr\_step$ [S1].
        iv. Update derivative: $dF \leftarrow dF + \gamma \cdot curr\_step$ [S1].
        v.  Clamp derivative: $dF \leftarrow \max(dF, \lambda)$ [S1].
    g.  **Early Stop+k**: Decrement $k$. If $k < 0$, break the loop [S1].
3.  **Return**: Return final $F$ and $step$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fbm_sphere_tracing(p, n, v_func, u, k, lambda_noise, gamma_noise):
    """
    Computes FBM value and step size using BVH acceleration.
    
    Args:
        p: Current 3D position (vec3)
        n: Number of octaves (int)
        v_func: Threshold function (callable)
        u: Ray direction (vec3, normalized)
        k: Early stop threshold (int)
        lambda_noise: Lipschitz constant for noise (float)
        gamma_noise: Gamma constant for noise (float)
        
    Returns:
        F: Final FBM value (float)
        step: Computed step size (float)
    """
    F = 0.0
    dF = 0.0
    step = 0.0
    
    # Global Lipschitz constants
    lambda_val = lambda_noise
    gamma_val = gamma_noise
    
    for i in range(1, n + 1):
        # Add current octave value and derivative
        # Note: o_i(p) and o'_i(p) are specific to the FBM implementation
        octave_val = get_octave_value(i, p)
        octave_deriv = get_octave_derivative(i, p)
        
        F += octave_val
        dF += dot(u, octave_deriv)
        
        # Compute Bounding Volume for remaining octaves
        # s(i+1, n) is the sum of max contributions of remaining octaves
        remaining_sum = get_remaining_octave_bound(i + 1, n)
        BVH = F + remaining_sum
        
        # Determine if step size computation is required
        comp_step = (i == n) or (BVH - v_func(0) < 0)
        
        if comp_step:
            # Adaptive Lipschitz: use local constants for current octave
            lambda_val = get_local_lipschitz(i)
            gamma_val = get_local_gamma(i)
            
            # Prepare inclusion function parameters
            incl_func_params = (BVH, lambda_val, dF, gamma_val)
            
            # Compute current step size
            curr_step = intersect(incl_func_params, v_func)
            
            # Add to total step
            step += curr_step
            
            # Sub-stepping guard (optional if Sub-stepping is on)
            # step = max(step, curr_step) 
            
            # Sub-stepping logic
            # Move along ray
            p = p + u * curr_step
            
            # Update sum and threshold
            F = v_func(curr_step)
            v_func_at_step = v_func(curr_step) # Update threshold value
            
            # Update derivative
            dF += gamma_val * curr_step
            
            # Clamp derivative
            dF = max(dF, lambda_val)
        
        # Early Stop+k
        k -= 1
        if k < 0:
            break
            
    return F, step
```

**Edge Cases and Pitfalls:**
*   **Thread Divergence**: The FBM sum loop may exit earlier for some threads than others due to Early Stop. However, rays from the same warp are spatially coherent, so divergence is typically limited to a single octave [S1].
*   **GPU Memory Usage**: The method avoids $O(n)$ per-thread data storage required by dynamic re-ordering methods, making it more suitable for fast GPU implementation [S1].
*   **Threshold Approximation**: The threshold function $v(\mathbf{p})$ is approximated by its 1st order (affine) Taylor polynomial. Higher order threshold functions were not implemented as the approximation showed no noticeable error in tested scenes [S1].
*   **Noise Parameters**: The algorithm requires $\lambda_{noise}$ and $\gamma_{noise}$ for arbitrary noise functions. Default values depend on the noise type (e.g., Perlin noise with or without absolute value) [S1].

## Subtopics

### [FBM Sphere Tracing with Implicit BVH](1901-fbm-sphere-tracing-with-implicit-bvh.md)
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by treating the cascaded sum of octaves as a dynamic, implicit Bounding Volume Hierarchy (BVH). It progressively evaluates octaves while bounding the remaining potential contribution to determine if the total function value can exceed the isosurface threshold, allowing for early termination and empty space skipping. Unlike naive sphere tracing which evaluates the full FBM at every step, this approach reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness. It builds on [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) by replacing the explicit spatial tree with a procedural hierarchy derived from the FBM structure itself.

## Operating characteristics
*   **Time Complexity**: The method reduces the average number of octaves computed per ray significantly. In the Sponge scene (close view), Early Stop reduced average octaves from 49,596.3 to 24,252.6, achieving a 1.9x speedup. In the far view, it reduced octaves from 56,577.8 to 10,347.0, achieving a 4.7x speedup [S1].
*   **Memory Complexity**: The method avoids $O(n)$ per-thread data storage, making it memory-efficient for GPU implementation compared to dynamic re-ordering methods [S1].
*   **Performance**: On an Nvidia RTX 4080 Ti at 1024×1024 resolution:
    *   Adaptive Lipschitz reduced average steps from 428.1 to 331.7 (1.3x speedup) in the Sponge scene [S1].
    *   Sub-stepping reduced average steps from 331.7 to 265.2 (1.2x speedup) over Adaptive Lipschitz alone in the Sponge scene [S1].
    *   Early Stop+k=1 reduced average octaves to 3,815.0 and time to 65.0ms (1.1x speedup over ES only) in the Sponge scene [S1].
*   **Strengths**: Significant reduction in the number of octaves computed per ray, especially in empty space. Low memory overhead. Suitable for GPU implementation.
*   **Limitations**: Dynamic re-ordering [WH96] could yield slightly smaller step sizes, but the BVH approach has much lower performance overhead (no memory requirement or sorting), resulting in better overall GPU performance [S1].
*   **When to Use**: Rendering large, detailed procedural volumetric noise scenes where the cost of evaluating noise dominates.
*   **When Not to Use**: Scenes where dynamic re-ordering is feasible and memory is not a constraint, and maximum step size optimality is critical over GPU performance.

## Background
The method was introduced in the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1]. It was implemented in GLSL fragment shaders and tested on scenes such as the "Sponge" scene, which consists of a 20-octave FBM of Perlin noise. The paper highlights the importance of GPU efficiency, noting that thread divergence is limited due to spatial coherence of rays within a warp. The method avoids the need for external object data, making it fully procedural.

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
