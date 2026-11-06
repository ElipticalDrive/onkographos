# FBM Sphere Tracing with BVH Acceleration

Also called: High-level view algorithm of how step size is computed during the FBM loop, Procedural FBM Sphere Tracing

## Overview
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by computing valid Sphere Distance Bounds (SDB) and adaptive step sizes using a Bounding Volume Hierarchy (BVH) derived from the FBM structure itself [S1]. Unlike naive sphere tracing which evaluates the full FBM sum at every step, this approach reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness [S1]. It builds on [Distance-field ray marching](21-distance-field-ray-marching.md) by replacing explicit spatial acceleration structures with a procedural hierarchy that allows for early termination of octave summation and tighter step size estimation [S1].

## How it works
The algorithm integrates the FBM summation loop with the sphere tracing step size computation. It maintains the current FBM value $F$, the derivative along the ray $dF$, and the accumulated step size. For each octave, it computes a bounding volume for the remaining octaves to determine if further evaluation is necessary or if a safe step size can be computed.

**Inputs:**
- $\mathbf{p}$: Current ray position in $\mathbb{R}^3$.
- $n$: Number of octaves in the FBM sum.
- $v(\cdot)$: Threshold function (hypertexture), approximated by its 1st order affine Taylor polynomial [S1].
- $\mathbf{u}$: Ray direction vector.
- $k$: Early stop threshold parameter.

**Outputs:**
- $F$: Final FBM value.
- $\text{step}$: Computed step size for the current sphere trace iteration.

**Data Structures and Parameters:**
- $\lambda_{\text{noise}}, \gamma_{\text{noise}}$: Lipschitz constants for the noise function. Default values used for Perlin noise [Per85] [S1].
- $\lambda_{Fn}, \gamma_{Fn}$: Global Lipschitz constants for the full FBM sum.
- $\lambda_{Fi}, \gamma_{Fi}$: Local Lipschitz constants for the current octave $i$ (used in Adaptive Lipschitz mode) [S1].
- $o_i(\mathbf{p})$: The value of the $i$-th octave of the FBM at point $\mathbf{p}$.
- $o'_i(\mathbf{p})$: The derivative of the $i$-th octave along the ray direction $\mathbf{u}$, i.e., $\nabla o_i(\mathbf{p}) \cdot \mathbf{u}$ [S1].
- $s(i+1, n)$: The bounding volume (maximum possible remaining contribution) of octaves from $i+1$ to $n$ [S1].
- $\text{BVH}$: The bounding volume for the remaining sum, calculated as $F + s(i+1, n)$ [S1].
- $\text{inclFuncParams}$: Parameters for the inclusion function intersection test, containing $(\text{BVH}, \lambda, dF, \gamma)$ [S1].

**Algorithm Steps:**

1.  **Initialization:**
    Set $F \leftarrow 0$, $dF \leftarrow 0$, $\text{step} \leftarrow 0$.
    Set global Lipschitz constants $\lambda \leftarrow \lambda_{Fn}$ and $\gamma \leftarrow \gamma_{Fn}$ [S1].

2.  **Octave Loop:**
    Iterate $i$ from $1$ to $n$:
    
    a.  **Accumulate Octave:**
        Update the FBM value: $F \leftarrow F + o_i(\mathbf{p})$.
        Update the derivative along the ray: $dF \leftarrow dF + (u \cdot o'_i(\mathbf{p}))$ [S1].
        
    b.  **Compute Bounding Volume:**
        Calculate the bounding volume for the remaining octaves: $\text{BVH} \leftarrow F + s(i+1, n)$ [S1].
        
    c.  **Determine Step Computation:**
        Compute the flag $\text{compStep}$. It is true if:
        - It is the last octave ($i == n$), OR
        - The ray is outside the bounding volume ($\text{BVH} - v(0) < 0$) [S1].
        
    d.  **Compute Step Size (if $\text{compStep}$ is true):**
        - **Adaptive Lipschitz:** Replace global constants with local ones: $\lambda \leftarrow \lambda_{Fi}$, $\gamma \leftarrow \gamma_{Fi}$ [S1].
        - **Inclusion Function:** Prepare parameters $\text{inclFuncParams} \leftarrow (\text{BVH}, \lambda, dF, \gamma)$.
        - **Intersection:** Compute the current step size: $\text{curr\_step} \leftarrow \text{intersect}(\text{inclFuncParams}, v)$ [S1].
        - **Accumulate Step:** $\text{step} \leftarrow \text{step} + \text{curr\_step}$.
        - **Sub-stepping Guard:** If Sub-stepping is enabled, enforce minimum step size: $\text{step} \leftarrow \max(\text{step}, \text{curr\_step})$ [S1].
        
    e.  **Sub-stepping (if enabled):**
        - Move ray position: $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{u} \cdot \text{curr\_step}$ [S1].
        - Update FBM sum: $F \leftarrow v(\text{curr\_step})$ [S1].
        - Update threshold: $v \leftarrow v$ at $\text{curr\_step}$ [S1].
        - Update derivative: $dF \leftarrow dF + \gamma \cdot \text{curr\_step}$ [S1].
        - Clamp derivative: $dF \leftarrow \max(dF, \lambda)$ [S1].
        
    f.  **Early Stop +k:**
        Decrement $k$. If $k < 0$, break the loop [S1].

3.  **Return:**
    Return $F$ and $\text{step}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fbm_sphere_tracing(p, n, v_func, u, k, lambda_noise, gamma_noise, lambda_Fn, gamma_Fn):
    """
    Computes FBM value and step size using BVH acceleration.
    
    Args:
        p: Current position (vec3)
        n: Number of octaves (int)
        v_func: Threshold function (callable, approximated by 1st order Taylor)
        u: Ray direction (vec3)
        k: Early stop threshold (int)
        lambda_noise: Lipschitz constant for noise
        gamma_noise: Derivative bound for noise
        lambda_Fn: Global Lipschitz constant for FBM
        gamma_Fn: Global derivative bound for FBM
        
    Returns:
        F: Final FBM value (float)
        step: Computed step size (float)
    """
    F = 0.0
    dF = 0.0
    step = 0.0
    lambda_val = lambda_Fn
    gamma_val = gamma_Fn
    
    for i in range(1, n + 1):
        # Accumulate current octave
        # o_i(p) and o'_i(p) are specific to the noise implementation
        octave_val = octave_function(i, p)
        octave_deriv = octave_derivative_function(i, p, u)
        
        F += octave_val
        dF += octave_deriv
        
        # Compute Bounding Volume for remaining octaves
        # s(i+1, n) is the max possible sum of remaining octaves
        remaining_bound = remaining_octaves_bound(i + 1, n)
        BVH = F + remaining_bound
        
        # Determine if step size computation is required
        # compStep is true if last octave OR outside bounding volume
        is_last_octave = (i == n)
        is_outside_bvh = (BVH - v_func(0.0) < 0)
        comp_step = is_last_octave or is_outside_bvh
        
        if comp_step:
            # Adaptive Lipschitz: use local constants for current octave
            lambda_val = local_lipschitz(i)
            gamma_val = local_derivative_bound(i)
            
            # Compute step size using inclusion function
            incl_params = (BVH, lambda_val, dF, gamma_val)
            curr_step = intersect_inclusion_function(incl_params, v_func)
            
            # Accumulate step
            step += curr_step
            
            # Sub-stepping guard (optional if Sub-stepping is on)
            # step = max(step, curr_step) 
            
            # Sub-stepping related updates
            # Move along ray
            p = p + u * curr_step
            
            # Update sum and threshold
            # Note: v_func(curr_step) implies updating the threshold value based on distance
            F = v_func(curr_step)
            # v is updated at curr_step (implementation detail for hypertexture)
            
            # Update derivative and clamp
            dF += gamma_val * curr_step
            dF = max(dF, lambda_val)
            
            # If Improved Quadratic (not detailed in high-level alg, but mentioned in notes)
            # k decrement happens here
            
        # Early Stop +k
        k -= 1
        if k < 0:
            break
            
    return F, step
```

**Edge Cases and Pitfalls:**
- **Thread Divergence:** The FBM sum loop may exit earlier for some threads than others due to Early Stop. However, rays from the same warp are spatially coherent, so divergence is typically limited to a single octave [S1].
- **GPU Memory Usage:** The method avoids $O(n)$ per-thread data storage required by dynamic re-ordering methods [WH96], making it more suitable for fast GPU implementation [S1].
- **Threshold Approximation:** The threshold function $v(p)$ is approximated by its 1st order (affine) Taylor polynomial. Higher order functions were not implemented as no noticeable error was observed [S1].
- **Noise Parameters:** The algorithm requires $\lambda_{\text{noise}}$ and $\gamma_{\text{noise}}$ for arbitrary noise functions. Default values are used for Perlin noise [Per85] [S1].

## Subtopics

### [FBM Sphere Tracing with Implicit BVH](1901-fbm-sphere-tracing-with-implicit-bvh.md)
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by treating the cascaded sum of octaves as a dynamic, implicit Bounding Volume Hierarchy (BVH). It progressively evaluates octaves while bounding the remaining potential contribution to determine if the total function value can exceed the isosurface threshold, allowing for early termination and empty space skipping. Unlike naive sphere tracing which evaluates the full FBM at every step, this approach reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness. It builds on [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) by replacing the explicit spatial tree with a procedural hierarchy derived from the FBM structure itself.

## Operating characteristics
- **Time Complexity:** The average number of octaves computed per step is reduced from $O(N)$ to $O(k)$ in empty space, where $k \ll N$ [S1].
- **Memory Complexity:** $O(1)$ per thread, avoiding the $O(n)$ storage required by dynamic re-ordering methods [S1].
- **Performance:** Tested on an Nvidia RTX 4080 Ti at 1024×1024 resolution on a "Sponge" scene (20-octave Perlin noise) [S1]:
    - **Ray Marching with Early Stop:** Reduced average octaves from 49,596.3 to 24,252.6 (close view) and 56,577.8 to 10,347.0 (far view). Time reduced from 770.0ms to 399.8ms (1.9x speedup) and 843.0ms to 178.4ms (4.7x speedup) [S1].
    - **Sphere Tracing with Adaptive Lipschitz:** Reduced average steps from 428.1 to 331.7 (close view) and 223.7 to 154.5 (far view). Time reduced from 110.2ms to 87.1ms (1.3x speedup) and 42.4ms to 28.3ms (1.5x speedup) [S1].
    - **Sub-stepping:** Further reduced steps from 331.7 to 265.2 (close view) and 154.5 to 273.0 (far view, steps increased but octaves decreased significantly). Time reduced from 87.1ms to 72.5ms (1.2x speedup) and 28.3ms to 22.0ms (1.3x speedup) [S1].
    - **Early Stop +k:** With $k=1$, average octaves reduced to 3,815.0 (close view) and 749.6 (far view). Time reduced to 65.0ms (1.1x speedup over ES only) and 16.7ms (1.4x speedup over ES only) [S1].
- **Strengths:** Significant performance gains in scenes with large empty spaces or intricate procedural noise. Low memory overhead. Suitable for GPU implementation due to limited thread divergence [S1].
- **Limitations:** Step sizes may be slightly smaller than dynamic re-ordering methods [WH96], but the overhead is much lower [S1]. Requires careful tuning of Lipschitz constants and early stop parameters [S1].

## Background
The method was introduced in the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1]. It addresses the computational cost of rendering high-frequency procedural noise using sphere tracing. The approach leverages the hierarchical nature of FBM to create a procedural BVH, allowing for adaptive step sizes and early termination of octave summation. It was implemented in GLSL fragment shaders and tested on high-end GPUs [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
