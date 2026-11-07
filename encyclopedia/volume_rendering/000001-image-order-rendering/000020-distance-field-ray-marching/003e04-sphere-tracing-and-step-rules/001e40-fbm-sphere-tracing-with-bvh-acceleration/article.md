# FBM Sphere Tracing with BVH Acceleration

Also called: High-level view algorithm of how step size is computed during the FBM loop, Procedural FBM Sphere Tracing

## Overview
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by computing valid Sphere Distance Bounds (SDB) and adaptive step sizes using a dynamic Bounding Volume Hierarchy (BVH) derived from the FBM structure itself [S1]. It reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness, by progressively evaluating octaves while bounding the remaining potential contribution to determine if the total function value can exceed the isosurface threshold [S1]. Unlike naive sphere tracing which evaluates the full FBM at every step, this approach supports features like Adaptive Lipschitz constants, Sub-stepping, Early Stop+k, and Quadratic Taylor inclusion functions [S1].

## How it works
The algorithm computes a step size $\Delta t$ and the final FBM value $F$ by iterating through the octaves of the FBM sum. It maintains a running sum of the FBM value $F$, the derivative along the ray $dF$, and the accumulated step size $step$. At each octave, it computes a bounding volume $BVH$ representing the maximum possible contribution of the remaining octaves. If the ray is outside this bounding volume (or it is the last octave), it computes a valid step size using inclusion functions and adaptive Lipschitz constants. It then performs sub-stepping to update the position and thresholds, and applies an early stop mechanism to terminate the octave loop early if the remaining contribution is negligible.

**Inputs:**
- $\mathbf{p}$: Current 3D position in world space.
- $n$: Number of octaves in the FBM sum. Typical value: 20 [S1].
- $v(\cdot)$: Threshold function (hypertexture). In implementation, approximated by its 1st order (affine) Taylor polynomial [S1].
- $\mathbf{u}$: Ray direction vector.
- $k$: Early Stop threshold parameter. Typical values tested: 0, 1, 2, 3 [S1].
- $\lambda_{noise}, \gamma_{noise}$: Lipschitz constants for the noise function. Default values used: Perlin noise [Per85] with or without absolute value [S1].

**Outputs:**
- $F$: Final FBM value.
- $step$: Computed step size (SDB).

**Data Structures:**
- Scalars: $F, dF, step, BVH, compStep, curr\_step, k$.
- Vectors: $\mathbf{p}, \mathbf{u}$.
- Constants: $\lambda, \gamma$ (Lipschitz constants).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fbm_sphere_tracing(p, n, v, u, k, lambda_noise, gamma_noise):
    # Initialize accumulators
    F = 0.0
    dF = 0.0
    step = 0.0
    
    # Set global Lipschitz constants
    # lambda_Fn and gamma_Fn are global bounds for the full FBM
    lambda_Fn = n * lambda_noise
    gamma_Fn = n * gamma_noise
    lambda = lambda_Fn
    gamma = gamma_Fn
    
    # Iterate through each octave
    for i in range(1, n + 1):
        # Add current octave value to F
        # o_i(p) is the i-th octave of the noise function at p
        o_i = noise_octave(i, p)
        F = F + o_i
        
        # Add derivative along ray to dF
        # o'_i(p) is the derivative of the i-th octave at p
        # u is the ray direction
        o_prime_i = noise_derivative_octave(i, p)
        dF = dF + dot(u, o_prime_i)
        
        # Compute Bounding Volume H (BVH) for remaining octaves
        # s(i+1, n) is the sum of absolute maximums of remaining octaves
        # Assuming noise is bounded in [-1, 1], s(i+1, n) = (n - i) * scale_factor
        # The source uses s(i+1, n) directly as a bound on remaining contribution
        s_remaining = sum_of_remaining_octave_bounds(i + 1, n)
        BVH = F + s_remaining
        
        # Determine if step size computation is required
        # compStep is true if it is the last octave OR if the ray is outside the bounding volume
        # v(0) is the threshold value at the current step (often 0 for isosurface)
        is_last_octave = (i == n)
        is_outside_bvh = (BVH - v(0) < 0)
        compStep = is_last_octave or is_outside_bvh
        
        if compStep:
            # Update Lipschitz constants to local values (Adaptive Lipschitz)
            # lambda_Fi and gamma_Fi are local bounds for the current octave
            lambda_Fi = lambda_noise
            gamma_Fi = gamma_noise
            lambda = lambda_Fi
            gamma = gamma_Fi
            
            # Prepare inclusion function parameters
            inclFuncParams = (BVH, lambda, dF, gamma)
            
            # Compute current step size using intersection with inclusion function
            # intersect returns the distance to the nearest surface based on bounds
            curr_step = intersect(inclFuncParams, v)
            
            # Update total step size
            step = step + curr_step
            
            # Enforce minimum step size (optional if Sub-stepping is on)
            # step = max(step, curr_step) 
            # Note: The source code comment says "Can be removed if SS is on"
            # We keep it for correctness if SS is not used, but in the main loop 
            # with SS, this line is effectively skipped or redundant.
            # For clarity, we assume SS is active as per the detailed sub-stepping logic below.
            
            # Sub-stepping related updates
            # Move ray position p along the ray by curr_step
            p = p + u * curr_step
            
            # Update sum F using threshold at current step
            # v(curr_step) approximates the change in threshold
            F = v(curr_step)
            
            # Update threshold v at current step
            # This updates the reference threshold for the next iteration
            v_current = v(curr_step)
            
            # Update derivative dF
            # dF += gamma * curr_step
            dF = dF + gamma * curr_step
            
            # Clamp dF
            dF = max(dF, lambda)
            
            # Apply Early Stop+k
            k = k - 1
            if k < 0:
                break # Early stop+k
    
    return F, step
```

**Parameters with Typical Values:**
- $n$: 20 (Sponge scene tests) [S1].
- $k$: 0, 1, 2, 3 [S1].
- $\lambda_{noise}, \gamma_{noise}$: Default values for Perlin noise [S1].

**Edge Cases and Pitfalls:**
- **Thread Divergence:** The FBM sum loop may exit earlier for some threads than others due to Early Stop. However, rays from the same warp are spatially coherent, so divergence is typically limited to a single octave [S1].
- **GPU Memory Usage:** The method avoids $O(n)$ per-thread data storage required by dynamic re-ordering methods, making it more suitable for fast GPU implementation [S1].
- **Threshold Approximation:** The threshold function $v(p)$ is approximated by its 1st order (affine) Taylor polynomial. Higher order approximations were not implemented as they showed no noticeable error [S1].
- **Noise Bounds:** The algorithm assumes the noise function is bounded in $[-1, 1]$. If arbitrary noise is used, $\lambda_{noise}$ and $\gamma_{noise}$ must be provided [S1].

## Subtopics

### [FBM Sphere Tracing with Implicit BVH](1901-fbm-sphere-tracing-with-implicit-bvh.md)
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by treating the cascaded sum of octaves as a dynamic, implicit Bounding Volume Hierarchy (BVH). It progressively evaluates octaves while bounding the remaining potential contribution to determine if the total function value can exceed the isosurface threshold, allowing for early termination and empty space skipping. Unlike naive sphere tracing which evaluates the full FBM at every step, this approach reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness. It builds on [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) by replacing the explicit spatial tree with a procedural hierarchy derived from the FBM structure itself.

## Operating characteristics
**Time and Memory Complexity:**
- The method reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ [S1].
- Memory usage is $O(1)$ per thread, avoiding the $O(n)$ storage required by dynamic re-ordering methods [S1].

**Measured Performance:**
- Tests were performed on an Nvidia RTX 4080 Ti at 1024×1024 resolution [S1].
- **Ray-marching on Sponge scene (close view):** Early Stop reduced avg octaves from 49596.3 to 24252.6, time from 770.0ms to 399.8ms, achieving a 1.9x speedup [S1].
- **Ray-marching on Sponge (far) scene:** Early Stop reduced avg octaves from 56577.8 to 10347.0, time from 843.0ms to 178.4ms, achieving a 4.7x speedup [S1].
- **Sphere tracing on Sponge scene:** Adaptive Lipschitz reduced avg steps from 428.1 to 331.7, time from 110.2ms to 87.1ms, achieving a 1.3x speedup over naive [S1].
- **Sphere tracing on Sponge (far) scene:** Adaptive Lipschitz reduced avg steps from 223.7 to 154.5, time from 42.4ms to 28.3ms, achieving a 1.5x speedup over naive [S1].
- **Sphere tracing on Sponge scene with Sub-stepping:** Reduced avg steps from 331.7 to 265.2, time from 87.1ms to 72.5ms, achieving a 1.2x speedup over Adaptive Lipschitz alone [S1].
- **Sphere tracing on Sponge (far) scene with Sub-stepping:** Reduced avg steps from 154.5 to 273.0 (note: steps increased but octaves decreased significantly), time from 28.3ms to 22.0ms, achieving a 1.3x speedup over Adaptive Lipschitz alone [S1].
- **Sphere tracing on Sponge scene with ES+k=1:** Avg octaves 3815.0, time 65.0ms, 1.1x speedup over ES only (72.5ms) [S1].
- **Sphere tracing on Sponge (far) scene with ES+k=1:** Avg octaves 749.6, time 16.7ms, 1.4x speedup over ES only (22.7ms) [S1].

**Strengths and Limitations:**
- **Strengths:** Significant performance gains in empty space and for high-octave FBM. Low memory overhead. Suitable for GPU implementation.
- **Limitations:** Dynamic re-ordering [WH96] could yield slightly better step sizes, but with much larger performance overhead (memory requirement + sorting) [S1].

**When to Use:**
- Use when rendering procedural volumetric noise with many octaves where empty space skipping is beneficial.
- Use when GPU memory is a constraint.

**When Not to Use:**
- Use naive sphere tracing if the scene is dense and Early Stop provides little benefit.
- Use dynamic re-ordering if step size accuracy is critical and memory overhead is acceptable.

## Background
The method was introduced in the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1]. It was implemented in GLSL fragment shaders and tested on scenes with 20-octave FBM of Perlin noise. The paper highlights the efficiency of the BVH approach on GPU hardware, particularly in managing thread divergence and memory usage.

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
