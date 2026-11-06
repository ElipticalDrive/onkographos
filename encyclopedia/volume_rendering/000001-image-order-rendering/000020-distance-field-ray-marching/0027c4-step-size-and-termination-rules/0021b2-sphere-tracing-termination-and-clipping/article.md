# Sphere Tracing Termination and Clipping

Also called: Sphere Tracing Stopping Criteria, Sphere Tracing Termination Conditions

## Overview
Sphere Tracing Termination and Clipping defines the conditions under which a ray-marching loop, which advances along a ray by querying a distance field, must halt. It solves the problem of determining when a ray has sufficiently sampled the scene to render a pixel without infinite iteration or excessive computational cost. It sets itself apart from sibling methods like [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by relying on the geometric properties of the distance function (specifically the Lipschitz continuity) to guarantee safety, and from [Step Size Optimization](2054-step-size-optimization.md) by focusing on the *stopping* logic rather than the *step size* calculation itself. It builds on the shared principles of [Step Size and Termination Rules](4132-step-size-and-termination-rules.md).

## How it works
The method iteratively advances a ray $r(t) = \mathbf{o} + t\mathbf{d}$ along direction $\mathbf{d}$ from origin $\mathbf{o}$. At each step $k$, the current position is $\mathbf{p}_k$ and the accumulated distance is $t_k$. The distance field function $f(\mathbf{p})$ returns the signed distance to the nearest surface.

### Inputs and Outputs
*   **Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), distance field function $f(\mathbf{p})$, maximum iterations $k_{max}$, stopping threshold $\delta$, oscillation threshold factor $\alpha$ (typically 6), far plane depth $t_{max}$.
*   **Outputs:** Intersection status (hit/miss), intersection distance $t_{hit}$, or termination reason (max steps, far plane, oscillation).

### Data Structures
*   **Iteration State:**
    *   `current_dist` ($t$): Float, accumulated distance along ray.
    *   `prev_dist` ($d_{bk-1}$): Float, distance value $f(\mathbf{p}_{k-1})$ from previous step.
    *   `curr_dist` ($d_{bk}$): Float, distance value $f(\mathbf{p}_k)$ from current step.
    *   `step_count` ($k$): Integer, current iteration index.
*   **Constants:**
    *   $\delta$: Stopping distance threshold. Typical value: $0.0003$ [S1].
    *   $k_{max}$: Maximum sphere tracing iterations. Typical value: $200$ [S1].
    *   $t_{max}$: Far plane clipping depth. Typical value: $5.0$ [S1].
    *   $\alpha$: Oscillation detection factor. Typical value: $6$ [S1].

### Numbered Steps
1.  **Initialization:**
    *   Set $t = 0$.
    *   Set $k = 0$.
    *   Set $\mathbf{p} = \mathbf{o}$.
    *   Query $d_{prev} = f(\mathbf{p})$.
    *   If $d_{prev} < \delta$, terminate as hit with $t_{hit} = t$ [S1].

2.  **Loop:**
    *   Increment $k \leftarrow k + 1$.
    *   **Max Steps Check:** If $k > k_{max}$, terminate as miss (max steps reached) [S1].
    *   **Far Plane Clipping:** If $t > t_{max}$, terminate as miss (far plane) [S1].
    *   **Step Execution:**
        *   Calculate step size $\Delta t = d_{prev}$ (assuming $f$ is a valid SDF and Lipschitz constant is 1, or scaled appropriately) [Parent, S1].
        *   Update position: $\mathbf{p} \leftarrow \mathbf{p} + \Delta t \cdot \mathbf{d}$.
        *   Update distance: $t \leftarrow t + \Delta t$.
        *   Query new distance: $d_{curr} = f(\mathbf{p})$.
    *   **Convergence Check:** If $d_{curr} < \delta$, terminate as hit with $t_{hit} = t$ [S1].
    *   **Oscillation Check:**
        *   Calculate difference: $\Delta d = |d_{curr} - d_{prev}|$.
        *   If $\Delta d < \alpha \cdot \delta$ (where $\alpha=6$), terminate as oscillation (potential infinite loop or noise) [S1].
        *   Update $d_{prev} \leftarrow d_{curr}$.
    *   **Repeat Loop.**

3.  **Termination:**
    *   Return result based on termination reason.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_trace_termination(o, d, f, k_max=200, delta=0.0003, t_max=5.0, alpha=6):
    """
    Performs sphere tracing with specific termination and clipping rules.
    
    Args:
        o: Ray origin (vec3)
        d: Ray direction (vec3, normalized)
        f: Distance field function f(p) -> float
        k_max: Maximum iterations (int)
        delta: Stopping threshold (float)
        t_max: Far plane clipping depth (float)
        alpha: Oscillation factor (float, typically 6)
        
    Returns:
        dict: {'hit': bool, 't': float, 'reason': str}
    """
    t = 0.0
    k = 0
    p = o
    
    # Initial query
    d_prev = f(p)
    
    # Check if already inside or very close
    if d_prev < delta:
        return {'hit': True, 't': t, 'reason': 'initial_convergence'}
    
    while k < k_max:
        k += 1
        
        # Far plane clipping
        if t > t_max:
            return {'hit': False, 't': t, 'reason': 'far_plane'}
        
        # Step size calculation (assuming Lipschitz constant L=1 for standard SDF)
        # If L != 1, delta_t = d_prev / L
        delta_t = d_prev
        
        # Update position and accumulated distance
        p = p + delta_t * d
        t = t + delta_t
        
        # Query distance at new position
        d_curr = f(p)
        
        # Convergence check
        if d_curr < delta:
            return {'hit': True, 't': t, 'reason': 'convergence'}
        
        # Oscillation check
        # |d_bk - d_bk-1| < 6 * delta
        if abs(d_curr - d_prev) < alpha * delta:
            return {'hit': False, 't': t, 'reason': 'oscillation'}
        
        # Update previous distance for next iteration
        d_prev = d_curr
        
    # Max steps reached
    return {'hit': False, 't': t, 'reason': 'max_steps'}
```

### Parameters with Typical Values
*   $\delta = 0.0003$: Stopping threshold [S1].
*   $k_{max} = 200$: Maximum iterations [S1].
*   $t_{max} = 5.0$: Far plane clipping depth [S1].
*   $\alpha = 6$: Oscillation detection multiplier [S1].

### Edge Cases and Pitfalls
*   **Precision:** Floating-point errors can cause the oscillation check to trigger prematurely or fail to detect true oscillation. The threshold $\alpha \cdot \delta$ provides a buffer [S1].
*   **Boundaries:** The far plane clipping $t_{max}$ must be set according to the scene scale. A value of $5.0$ is typical for normalized unit scenes [S1].
*   **Oscillation:** In noisy distance fields or near complex geometry, the distance value may oscillate between two close values. The check $|d_{bk} - d_{bk-1}| < 6\delta$ prevents infinite loops in these scenarios [S1].
*   **Self-Intersection:** Standard sphere tracing with a global $\delta$ can cause self-intersection artifacts in secondary rays. This is addressed by variants like [Dynamic Epsilon for Self-Intersection Prevention](4318-dynamic-epsilon-for-self-intersection-prevention.md).

## Subtopics

### [Affine Arithmetic Early Skip Test](4313-affine-arithmetic-early-skip-test.md)
The Affine Arithmetic Early Skip Test is a constant-time heuristic used in distance-field ray marching to detect if a ray does not collide with an object within a specific working interval, thereby avoiding expensive root-finding operations. It distinguishes itself from sibling methods like Bracketed Newton Bisection (BNB) by providing immediate non-collision detection and a conservative starting point for subsequent searches, rather than searching for monotonic intervals of derivatives.

### [Dynamic Epsilon for Self-Intersection Prevention](4318-dynamic-epsilon-for-self-intersection-prevention.md)
Dynamic epsilon is a method for preventing self-intersection artifacts in secondary rays (e.g., refraction) during sphere tracing by computing a local offset $\varepsilon_{\text{dynamic}}$ at each intersection point, rather than using a global constant. It solves the problem where global epsilon values are inappropriate for sphere tracing with screen-space termination metrics because intersection distances from the surface exhibit large variations. Unlike global epsilon, dynamic epsilon adapts to the local surface distance, making it easier and more robust to tune than a single global value.

### [Convex Object Optimization](4367-convex-object-optimization.md)
Convex Object Optimization is a technique for accelerating sphere tracing through objects represented by a signed distance bound (SDB) that are enclosed within a convex bounding volume. It solves the problem of expensive interior traversal by avoiding costly marching through the inside of the object, which typically requires many iterations and gradient computations. It sets itself apart from standard sphere tracing by using the bounding volume to determine entry and exit points, thereby reducing the number of iterations required to find a ray-surface intersection.

### [Discontinuity Reduction via Fixed-Point Iteration](4368-discontinuity-reduction-via-fixed-point-iteration.md)
Discontinuity reduction via fixed-point iteration is a post-processing step applied to intersection points retrieved by sphere tracing to smooth out discontinuities in the signed distance function (SDF) $f(p)$ and reduce artifacts in procedural texturing. It sets itself apart from its siblings by trading a slight increase in average geometric error for significantly reduced visual discontinuities across the screen-space domain, whereas [Affine Arithmetic Early Skip Test](4313-affine-arithmetic-early-skip-test.md) focuses on skipping empty space and [Dynamic Epsilon for Self-Intersection Prevention](4318-dynamic-epsilon-for-self-intersection-prevention.md) focuses on preventing self-intersection artifacts.

### [Ray Intersection with Grazing Point Detection](4463-ray-intersection-with-grazing-point-detection.md)
A ray intersection method that performs sphere tracing to find the primary intersection while simultaneously tracking the 'grazing point' (the point on the surface where the ray direction is tangent to the surface normal, indicated by a zero directional derivative). It returns the intersection time, validity, grazing point status, and the grazing point position. It sets itself apart from standard sphere tracing by explicitly identifying and refining grazing contacts, which are often missed or poorly resolved by standard termination rules.

### [Gradient Magnitude Windowing](8299-gradient-magnitude-windowing.md)
Gradient Magnitude Windowing is a binary classification technique used in volume rendering to determine whether a ray sample lies on an object surface based on the magnitude of the volume gradient. It serves as a filter in contour rendering pipelines, discarding samples where the gradient magnitude $|g|$ is below a specified threshold $t_g$. It is distinct from its siblings in that it relies solely on the scalar magnitude of the gradient for a hard binary decision, whereas [Affine Arithmetic Early Skip Test](4313-affine-arithmetic-early-skip-test.md) uses interval arithmetic for early skipping, [Dynamic Epsilon for Self-Intersection Prevention](4318-dynamic-epsilon-for-self-intersection-prevention.md) adjusts step sizes to prevent self-intersection, and [Convex Object Optimization](4367-convex-object-optimization.md) exploits geometric convexity.

### Cone Tracing Error Threshold
A method to determine the adaptive error threshold $\hat{\epsilon}$ based on the ray distance and pixel footprint, ensuring rays terminate when the pixel cone intersects the surface [S2]. It calculates the error threshold as $\hat{\epsilon}(\hat{x}) = \hat{x} \hat{r}_p$, where $\hat{x}$ is the distance along the ray in undeformed space and $\hat{r}_p$ is the radius of one pixel in undeformed space [S2]. The pixel radius in undeformed space is calculated as $\hat{r}_p = s r_p J_D^{-1}$, where $s$ is the distance in deformed space, $r_p$ is the pixel radius in deformed space, and $J_D^{-1}$ is the inverse Jacobian determinant [S2]. If the threshold is too high, artifacts manifest as the surface 'swimming', appearing in different places depending on view direction [S2].

### Blend Mode
A rendering mode that traverses voxels and sums their value until value accumulation results in enough content, either being opaque for early termination or containing greater than zero content for blending with a user-specified background [S3, unofficial]. It traverses voxels and sums their value, checking if value accumulation results in enough content: either opaque for early termination or greater than zero for blending with background [S3, unofficial]. The high cost of voxel access can shrink or invert performance gains of block walking and culling in this mode [S3, unofficial].

### Optimized Binary Search with Local Lipschitz Bounds
A binary search method that recursively subdivides an interval, using local Lipschitz bounds to exclude intervals earlier and shrink sub-intervals asymmetrically [S4]. It uses the local Lipschitz bound $\lambda(e)$ to exclude intervals earlier in the recursive bisection process [S4]. It shrinks the interval $e=[a,b]$ to $[e_a, e_b]$ using $e_a = a + |f(a)|/\lambda(e)$ and $e_b = b - |f(b)|/\lambda(e)$ to improve bisection efficiency [S4]. Given an interval $[a, b]$ with function values $f(a)$ and $f(b)$ and local Lipschitz bound $\lambda(e)$, it shrinks the interval to $[e_a, e_b]$ using the update rules: $e_a = a + |f(a)|/\lambda(e)$ and $e_b = b - |f(b)|/\lambda(e)$ [S4]. Compared to standard binary search which cuts the segment in two equal parts, this method reduces the number of field function queries similarly to Segment Tracing and provides timing speed-ups [S4].

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of iterations until termination. In empty space, $k$ is small due to large steps. Near surfaces, $k$ increases as steps shrink.
*   **Memory Complexity:** $O(1)$ auxiliary memory, as only the current position, distance, and iteration count are stored.
*   **Measured Performance:** With typical parameters ($\delta=0.0003, k_{max}=200, t_{max}=5.0$), the method is efficient for real-time rendering [S1].
*   **Scaling:** Performance scales with scene complexity and ray direction. Oscillation checks add minimal overhead but prevent worst-case infinite loops [S1].
*   **Strengths:** Guarantees convergence for Lipschitz-continuous SDFs; prevents infinite loops via oscillation detection; simple to implement.
*   **Limitations:** Fixed thresholds may not be optimal for all scenes; oscillation detection may terminate valid rays in noisy fields; far plane clipping must be tuned to scene scale.
*   **When to Use:** Real-time rendering with SDFs; when memory is constrained; when robustness against infinite loops is required.
*   **When Not to Use:** When high precision is required far from the surface (use Cone Tracing); when dealing with non-Lipschitz functions; when secondary rays require adaptive epsilon.

## Background
The method originates from the foundational work on sphere tracing by John C. Hart in 1994 [S1]. The specific termination criteria, including the oscillation check and far plane clipping, were detailed in the "Neural Geometric Level of Detail" paper by Laine et al. (2021) [S1]. The paper established the use of $\delta = 0.0003$, $k_{max} = 200$, and $t_{max} = 5.0$ as standard parameters for real-time rendering with implicit shapes [S1]. The oscillation check $|d_{bk} - d_{bk-1}| < 6\delta$ was introduced to handle noise and precision issues in neural implicit representations [S1].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
[S3] [PDF] Sparse Volume Rendering using Hardware Ray Tracing and Block ..., https://gpuopen.com/download/SA2021_BlockWalk.pdf, fetched 2026-10-26 (unofficial)
[S4] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
