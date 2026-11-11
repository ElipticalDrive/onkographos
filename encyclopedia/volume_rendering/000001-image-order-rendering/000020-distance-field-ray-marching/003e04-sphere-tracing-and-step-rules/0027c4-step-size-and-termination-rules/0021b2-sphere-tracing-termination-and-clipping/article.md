# Sphere Tracing Termination and Clipping

Also called: Sphere Tracing Stopping Criteria, Sphere Tracing Termination Conditions

## Overview
[G] Sphere Tracing Termination and Clipping defines the conditions under which the iterative ray-marching loop halts, ensuring the algorithm converges to a surface intersection or correctly identifies a miss. It solves the problem of infinite loops and geometric inaccuracies by combining convergence thresholds, oscillation detection, and spatial clipping. It distinguishes itself from sibling methods like [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by strictly enforcing the Lipschitz safety constraint ($\Delta t \le f(\mathbf{p})$) for termination logic, and from [Sphere Tracing](1674-sphere-tracing.md) by explicitly handling edge cases such as oscillation and far-plane clipping to guarantee robustness in real-time rendering.

## How it works
The method operates on a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, querying a Signed Distance Function (SDF) $f(\mathbf{p})$ [G].

**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) [G].
*   SDF function $f(\mathbf{p})$ [G].
*   Maximum iteration count $k_{max}$ [S1].
*   Stopping distance threshold $\delta$ [S1].
*   Far plane clipping depth $t_{clip}$ [S1].
*   Oscillation tolerance factor $\alpha$ (typically 6) [S1].

**Outputs:**
*   Intersection distance $t_{hit}$ (if found) [G].
*   Boolean flag `hit` indicating success [G].

**Data Structures:**
*   `current_t`: Float, current ray parameter, initialized to 0 [G].
*   `prev_d`: Float, distance value from previous iteration, initialized to $f(\mathbf{o})$ [G].
*   `current_d`: Float, distance value at current position, $f(\mathbf{o} + \text{current\_t} \cdot \mathbf{d})$ [G].

**Parameters:**
*   $\delta = 0.0003$ [S1].
*   $k_{max} = 200$ [S1].
*   $t_{clip} = 5.0$ [S1].

**Steps:**
1.  Initialize `current_t` = 0.
2.  Evaluate `current_d` = $f(\mathbf{o} + \text{current\_t} \cdot \mathbf{d})$.
3.  Initialize `prev_d` = `current_d`.
4.  Loop while `current_t` < $t_{clip}$ and iteration count < $k_{max}$:
    a.  **Convergence Check:** If `current_d` < $\delta$, terminate loop and return `hit` = true, `t_hit` = `current_t` [S1].
    b.  **Oscillation Check:** If iteration > 0 and $|\text{current\_d} - \text{prev\_d}| < 6\delta$, terminate loop and return `hit` = false [S1].
    c.  **Step Update:** Update `prev_d` = `current_d`.
    d.  **Step Size Calculation:** Calculate step size $\Delta t = \max(0.0, \text{current\_d})$ [G]. If Lipschitz constant $\lambda > 1$, scale: $\Delta t = \max(0.0, \text{current\_d}) / \lambda$ [Parent].
    e.  **Ray Advance:** `current_t` = `current_t` + $\Delta t$.
    f.  **Far Plane Clip:** If `current_t` > $t_{clip}$, terminate loop and return `hit` = false [S1].
    g.  **Evaluation:** `current_d` = $f(\mathbf{o} + \text{current\_t} \cdot \mathbf{d})$.
5.  If loop completes without convergence, return `hit` = false [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_trace_termination(o, d, f, k_max=200, delta=0.0003, t_clip=5.0, alpha=6):
    current_t = 0.0
    current_d = f(o + current_t * d)
    prev_d = current_d
    
    for i in range(k_max):
        # 1. Convergence check
        if current_d < delta:
            return True, current_t
        
        # 2. Oscillation check (skip first iteration to avoid false positive on initialization)
        if i > 0 and abs(current_d - prev_d) < alpha * delta:
            return False, current_t
            
        prev_d = current_d
        
        # 3. Step size (Lipschitz constant lambda=1 assumed)
        # Clamp to non-negative to prevent backward stepping if ray starts inside volume
        step = max(0.0, current_d)
        
        # 4. Advance ray
        current_t += step
        
        # 5. Far plane clipping / Escape check
        if current_t > t_clip:
            return False, current_t
            
        # 6. Evaluate new position
        current_d = f(o + current_t * d)
        
    # Max iterations reached
    return False, current_t
```

**Edge Cases and Pitfalls:**
*   **Oscillation:** Standard sphere tracing may fail to converge if the ray bounces between two points very close to the surface. The oscillation check $|d_k - d_{k-1}| < 6\delta$ is critical for stability [S1].
*   **Precision:** The threshold $\delta$ must be chosen carefully relative to the scene scale. If the threshold is too high, artifacts manifest as the surface "swimming" [S2].
*   **Far Plane:** Clipping at a fixed depth (e.g., 5.0) terminates rays that exceed the bound [S1].

## Subtopics

### [Affine Arithmetic Early Skip Test](4313-affine-arithmetic-early-skip-test.md)
Uses interval arithmetic to detect non-collision within a working interval, avoiding expensive root-finding.

### [Dynamic Epsilon for Self-Intersection Prevention](4318-dynamic-epsilon-for-self-intersection-prevention.md)
Computes a local offset $\varepsilon_{\text{dynamic}}$ to prevent self-intersection in secondary rays, adapting to local surface distance variations.

### [Convex Object Optimization](4367-convex-object-optimization.md)
Accelerates interior traversal by using convex bounding volumes to determine entry and exit points, reducing iteration counts.

### [Discontinuity Reduction via Fixed-Point Iteration](4368-discontinuity-reduction-via-fixed-point-iteration.md)
Post-processes intersection points to smooth SDF discontinuities, trading slight geometric error for visual stability.

### [Ray Intersection with Grazing Point Detection](4463-ray-intersection-with-grazing-point-detection.md)
Tracks the grazing point (tangent contact) alongside the primary intersection to resolve grazing angles missed by standard rules.

### [Gradient Magnitude Windowing](8299-gradient-magnitude-windowing.md)
Filters samples based on gradient magnitude $|g|$ for binary surface classification in contour rendering.

### Cone Tracing Error Threshold
A method to determine the adaptive error threshold $\hat{\epsilon}$ based on the ray distance and pixel footprint, ensuring rays terminate when the pixel cone intersects the surface [S2]. It calculates the threshold as $\hat{\epsilon}(\hat{x}) = \hat{x} \hat{r}_p$, where $\hat{x}$ is the distance in undeformed space and $\hat{r}_p$ is the pixel radius in undeformed space ($\hat{r}_p = s r_p J_D^{-1}$) [S2]. If the threshold is too high, artifacts manifest as the surface "swimming" [S2].

### Blend Mode
A rendering mode that traverses voxels and sums their value until accumulation results in enough content: either opaque for early termination or greater than zero for blending with a user-specified background [S3, unofficial]. The high cost of voxel access can shrink or invert performance gains of block walking and culling in this mode [S3, unofficial].

### Optimized Binary Search with Local Lipschitz Bounds
A binary search method that recursively subdivides an interval, using local Lipschitz bounds $\lambda(e)$ to exclude intervals earlier and shrink sub-intervals asymmetrically [S4]. It shrinks interval $e=[a,b]$ to $[e_a, e_b]$ using $e_a = a + |f(a)|/\lambda(e)$ and $e_b = b - |f(b)|/\lambda(e)$ [S4]. This achieves similar field function query reduction and timing speed-ups compared to standard binary search [S4].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. In practice, iterations are bounded by $k_{max}$ (e.g., 200) [S1].
*   **Memory Complexity:** Not given in the sources. The method requires $O(1)$ auxiliary memory to store current and previous distance values and ray parameters [G].
*   **Measured Performance:** With $\delta=0.0003$ and $k_{max}=200$, the method provides stable real-time performance for implicit surfaces [S1].
*   **Strengths:** Guarantees no overshooting (with $\lambda=1$), handles oscillation, and prevents infinite loops.
*   **Limitations:** Fixed thresholds ($\delta$, $t_{clip}$) may not adapt well to varying scene scales. Oscillation detection may prematurely terminate valid but slow-converging rays.
*   **When to use:** Real-time rendering of implicit surfaces where stability and bounded iteration counts are critical.
*   **When not to use:** Scenes requiring sub-pixel accuracy without adaptive thresholds, or where the Lipschitz constant is significantly greater than 1 without proper scaling.

## Background
The method originates from the foundational sphere tracing algorithm by John C. Hart (1996), which established the safety constraint for step size. The specific termination criteria involving oscillation detection ($|d_k - d_{k-1}| < 6\delta$) and fixed iteration limits ($k=200$) were formalized in the context of Neural Implicit Representations, specifically in the "Neural Geometric Level of Detail" paper (2021) [S1]. The concept of adaptive error thresholds based on pixel footprints (Cone Tracing) was introduced to handle deformed SDFs, addressing artifacts like surface swimming [S2]. Optimized binary search with local Lipschitz bounds was developed to improve efficiency in segment tracing [S4].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
[S3] [PDF] Sparse Volume Rendering using Hardware Ray Tracing and Block ..., https://gpuopen.com/download/SA2021_BlockWalk.pdf, fetched 2026-10-26 (unofficial)
[S4] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
