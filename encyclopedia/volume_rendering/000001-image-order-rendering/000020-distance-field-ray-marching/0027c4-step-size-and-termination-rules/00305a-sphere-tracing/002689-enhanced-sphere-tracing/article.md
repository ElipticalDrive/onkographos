# Enhanced Sphere Tracing

Also called: Aggressive Marching, Aggressive Ray Marching, Aggressive Sphere Tracing, Over-relaxed sphere tracing, Sphere tracing with over-relaxation

## Overview
Enhanced Sphere Tracing is a ray marching strategy that accelerates convergence by stepping a multiple of the queried signed distance function (SDF) value, rather than the value itself [S1][S3, unofficial]. It solves the problem of slow convergence at grazing angles and allows the ray to penetrate the surface, enabling sampling of the interior for training supervision [S1]. It differs from [Cone Intersection](1553-cone-intersection.md) and [Segment Tracing](1675-segment-tracing.md) by using a scalar multiplier on the step size rather than geometric bounds or local Lipschitz constants to determine the step, and from [Over-relaxation Sphere Tracing](3805-over-relaxation-sphere-tracing.md) primarily in its implementation details regarding safety checks, though both share the core over-relaxation principle [S2][S3, unofficial].

## How it works
**Inputs:**
*   Ray origin $o$ and direction $\vec{v}$ (normalized).
*   SDF function $f(p)$ returning signed distance.
*   Aggressive factor $\alpha > 1$ (default 1.5) [S1].
*   Convergence threshold $\epsilon$ (e.g., $5 \times 10^{-5}$) [S1].
*   Maximum step count $k_{max}$ (e.g., 1024) [S3, unofficial].

**Outputs:**
*   Intersection point $p$ or null if not found within $k_{max}$.

**Data Structures:**
*   Current position $p$, initialized to $o$.
*   Step counter $s$, initialized to 0.

**Steps:**
1.  **Query:** Evaluate $d = f(p)$.
2.  **Check Convergence:** If $|d| < \epsilon$, return $p$ [S1].
3.  **Calculate Step:** Compute step size $s_{step} = \alpha \cdot d$. Note: If $d < 0$ (inside), the step moves further inside; if $d > 0$ (outside), it moves toward the surface [S1].
4.  **Advance:** Update position $p \leftarrow p + s_{step} \cdot \vec{v}$ [S1].
5.  **Increment:** $s \leftarrow s + 1$.
6.  **Termination:** If $s \ge k_{max}$, return null [S3, unofficial].
7.  **Loop:** Repeat from Step 1.

**Formulas:**
The convergence condition for the distance to the surface after $k$ steps is given by [S1]:
$$ |d_0 (1 - \alpha \sin \theta)^k| < \epsilon $$
where $d_0$ is the initial distance, $\theta$ is the angle between the ray direction and the surface normal, and $\alpha$ is the aggressive factor. The minimum number of steps $k_{min}$ required is [S1]:
$$ k > k_{min} = \frac{\log \epsilon - \log d_0}{\log |1 - \alpha \sin \theta|} $$
This derivation assumes $|1 - \alpha \sin \theta| < 1$ [S1].

**Parameters:**
*   $\alpha$: Aggressive marching multiplier. Default 1.5. Range 1.2 to 1.8 is effective [S1].
*   $\epsilon$: Convergence threshold. Typical value $5 \times 10^{-5}$ [S1].
*   $k_{max}$: Maximum steps. Typical value 1024 [S3, unofficial].

**Pitfalls:**
*   **Thin Geometry:** If geometry is super thin, a single aggressive step may trespass two surfaces, leaving the ray outside the shape and causing infinite marching or small holes [S1].
*   **Non-Conservative Fields:** The method assumes the SDF is conservative (never overstates distance). If the field overstates distance, over-relaxation is unsound and may skip surfaces [S3, unofficial].
*   **Grazing Angles:** While it improves performance, extremely small $\theta$ can still lead to slow convergence if $\alpha$ is not tuned correctly [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def enhanced_sphere_tracing(ray_origin, ray_direction, sdf, alpha=1.5, epsilon=5e-5, max_steps=1024):
    """
    Perform enhanced sphere tracing.
    
    Args:
        ray_origin: Vector3, origin of the ray.
        ray_direction: Vector3, normalized direction of the ray.
        sdf: Function, signed distance function f(p).
        alpha: Float, aggressive factor (> 1).
        epsilon: Float, convergence threshold.
        max_steps: Int, maximum number of steps.
        
    Returns:
        Vector3 or None: Intersection point or None if not found.
    """
    p = ray_origin
    for i in range(max_steps):
        d = sdf(p)
        
        # Check convergence
        if abs(d) < epsilon:
            return p
        
        # Calculate step size
        step_size = alpha * d
        
        # Advance position
        p = p + step_size * ray_direction
        
    return None
```

## Subtopics

### Overstepping
A technique where the ray marches a small distance further outside the Minimum Distance Sphere (MDS) edge to decrease the number of marches, increasing performance [S5, unofficial]. It calculates the over-stepping distance as $o(p) = (1 + \frac{\sigma}{\text{range } [0,1]}) \frac{|f(p)|}{\lambda}$, where $\sigma$ is the overstepping factor (typically $\approx 1.2$) and $\lambda$ is the global Lipschitz bound [S2]. The algorithm marches to the edge of the MDS, then further outside. It compares the original MDS with the new MDS; if they overlap, it continues marching further. If they do not overlap, it steps back to the edge of the first MDS and continues normally [S5, unofficial]. This is not directly compatible with local Lipschitz bound approaches as it relies on a global bound $\lambda$ [S2].

### [Parallel Sphere Carving Algorithm](4165-parallel-sphere-carving-algorithm.md)
A GPU-friendly implementation of the Sphere Carving method for extracting surfaces from conservative SDFs. It solves the problem of efficiently generating a set of empty spheres that bound a surface, optimizing for parallel execution on graphics hardware. Unlike sequential sphere carving, it uses four distinct parallel kernels to gather intersecting pairs, compute intersection triplets, filter interior points, and augment the sphere set simultaneously. It builds on Enhanced Sphere Tracing by providing a structured, parallelizable iteration step for the sphere carving process.

## Operating characteristics
**Time Complexity:** Reduces the number of steps $k$ needed to reach convergence $\epsilon$ compared to conventional sphere tracing ($\alpha=1.0$) [S1]. For $d=1.0$, $\epsilon=5 \times 10^{-5}$, and $\theta=10$ degrees, steps decrease from 52 to 33 with $\alpha=1.5$ [S1]. Using $\sigma \approx 1.2$ in overstepping variants reduces field function queries by a factor of 1.2 [S2].

**Memory Complexity:** No additional memory beyond standard ray marching state [S1].

**Performance:**
*   On 432x756 resolution using ANGLE/Metal backend, worst-angle performance cost increases from 0.105 ms/frame to 0.165 ms/frame [S3, unofficial].
*   Coverage improvement for high conditioning ratios (e.g., 60:1) increases from 0.00% to 12.42% [S3, unofficial].
*   For conditioning ratio 120:1, coverage improves from 0.00% to 12.78% [S3, unofficial].

**Strengths:**
*   Accelerates convergence for rays grazing surfaces [S1].
*   Allows sampling behind the surface (SDF < 0), which is crucial for supervision during neural network training [S1].
*   Prevents valid geometry from vanishing at grazing angles under high non-uniform scaling ratios when used with conservative fields [S3, unofficial].

**Limitations:**
*   Can cause artifacts (small holes) in super thin geometry if a single step trespasses two surfaces [S1].
*   Requires a conservative SDF (never overstates distance) to be sound [S3, unofficial].
*   Not directly compatible with local Lipschitz bound approaches due to reliance on global bounds in some variants [S2].

## Background
Enhanced Sphere Tracing, also known as Aggressive Marching or Over-relaxed Sphere Tracing, was attributed to Keinert et al. in 2014 [S3, unofficial]. It was developed to address the inefficiency of standard sphere tracing, particularly at grazing angles where the angle between the camera ray and the surface normal is small [S1]. The method has been adopted in differentiable rendering pipelines to stabilize training by allowing interior sampling [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
[S3] Over-relax the viewport sphere tracer so scaled fields stop dropping out, https://github.com/kmatzen/sinter/pull/79, fetched 2026-10-27 (unofficial)
[S4] aparis69/Segment-Tracing, https://github.com/aparis69/Segment-Tracing, fetched 2026-10-27 (unofficial)
[S5] Sphere Tracing GPU An evaluation of the Sphere Tracing algorithm and a GPU designed to run it, https://odr.chalmers.se/items/37b0e53f-fc76-4b0e-96d8-8e864573ada9/full, fetched 2026-10-29 (unofficial)
