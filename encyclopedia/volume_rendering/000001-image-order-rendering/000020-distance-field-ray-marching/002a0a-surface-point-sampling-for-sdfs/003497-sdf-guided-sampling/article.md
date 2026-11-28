# SDF-guided Sampling

Also called: Shao et al. SDF-guided Sampling, Surface-guided Sampling Strategy

## Overview
SDF-guided Sampling is a surface-aware sampling strategy that uses a Signed Distance Field (SDF) to estimate ray-surface intersections, followed by fine-grained sampling around the intersection point to accelerate rendering and emphasize the relationship between surface and radiance fields [S1, S2]. It differs from standard NeRF sampling by reducing the number of required sampling points through early surface localization, and from pure mesh-based methods by retaining the flexibility of implicit field evaluation without requiring explicit mesh vertex interpolation [S1, S2].

## How it works
**Inputs:**
- Ray origin $o$ and direction $d$ [S2].
- Depth bounds $[t_n, t_f]$ defining the integration interval along the ray [S2].
- Surface field function $s(x)$, which outputs a scalar value for any point $x$ [S2].
- Parameters:
  - $N_s$: Number of uniform sampling points for initial coarse sampling [S2].
  - $N_r$: Number of sampling points for fine-grained sampling around the surface [S2].
  - $\delta$: Interval size for fine-grained sampling around the intersection [S2].
  - Threshold $T = 0.5$ for determining intersection [S2].

**Outputs:**
- A set of sample points concentrated around the estimated surface intersection, used to evaluate the radiance field [S2].

**Data Structures:**
- No complex data structures are required beyond the ray parameters and the implicit field function $s(x)$; the method relies on querying the field at discrete points [G].

**Steps:**
1. **Coarse Uniform Sampling:** Generate $N_s$ points uniformly distributed along the ray within the depth bounds $[t_n, t_f]$ [S2]. The $i$-th point $x_i$ is calculated as:
   $$ x_i = o + t_i d $$
   where $t_i$ are uniformly spaced values in $[t_n, t_f]$ [S2].

2. **Intersection Estimation:** Query the surface field value $s(x_i)$ for each coarse sample point [S2]. Identify the first intersection position $t_{int}$ by finding the smallest $t_i$ such that the surface field value exceeds the threshold:
   $$ t_{int} = \min \{ t_i \mid s(o + t_i d) \ge 0.5 \} $$
   [S2].

3. **Fine-Grained Sampling:** Perform fine-grained sampling around the intersection in an interval of $\delta$ with $N_r$ sampling points to consider the radiance field surrounding the real surface [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linspace(start, stop, num):
    """Generate num evenly spaced values over [start, stop]."""
    step = (stop - start) / (num - 1) if num > 1 else 0
    return [start + i * step for i in range(num)]

def sdf_guided_sampling(o, d, t_n, t_f, s_field, N_s, N_r, delta, threshold=0.5):
    """
    Perform SDF-guided sampling along a ray.
    
    Args:
        o: Ray origin (3D vector)
        d: Ray direction (3D vector, assumed normalized)
        t_n: Near depth bound (float)
        t_f: Far depth bound (float)
        s_field: Function that returns surface field value s(x) for point x
        N_s: Number of coarse sampling points
        N_r: Number of fine sampling points
        delta: Interval size for fine sampling around intersection
        threshold: Threshold for surface intersection detection (default 0.5)
        
    Returns:
        List of sample points (3D vectors) concentrated around the surface
    """
    # Assumes standard vector addition and scalar multiplication
    
    # Step 1: Coarse uniform sampling
    t_coarse = linspace(t_n, t_f, N_s)
    points_coarse = [o + t * d for t in t_coarse]
    
    # Step 2: Find first intersection
    t_int = None
    for i in range(N_s):
        t_i = t_coarse[i]
        x_i = points_coarse[i]
        s_val = s_field(x_i)
        if s_val >= threshold:
            t_int = t_i
            break
            
    if t_int is None:
        return []
        
    # Step 3: Fine-grained sampling around intersection
    t_fine = linspace(t_int - delta, t_int + delta, N_r)
    points_fine = [o + t * d for t in t_fine]
    
    return points_fine
```

**Edge Cases and Pitfalls:**
- **No Intersection:** If the ray does not intersect the surface within $[t_n, t_f]$ (i.e., no $s(x_i) \ge 0.5$), the method yields no fine samples [G].
- **Threshold Sensitivity:** The threshold of $0.5$ is specific to the DoubleField formulation [S2]. Other SDF representations might use $0.0$ (zero-level set) [G]. The implementer must ensure the threshold matches the normalization of the provided SDF field [G].
- **Discrete Sampling:** The intersection detection relies on discrete coarse samples [G]. If the surface is thin or the coarse sampling is too sparse, the intersection might be missed or inaccurately located [G].

## Operating characteristics
- **Time Complexity:** Dominated by $N_s$ queries to the SDF field for coarse sampling, plus $N_r$ queries for fine sampling, totaling $N_s + N_r$ field evaluations [S2].
- **Memory Complexity:** Auxiliary memory scales linearly with the number of sample points, $O(N_s + N_r)$, as it only stores the generated coordinates [G].
- **Performance:** Reported to be much faster than standard NeRF sampling due to fewer sampling points needed for integration [S2].
- **Strengths:** Accelerates rendering by focusing samples on the surface; emphasizes the relationship between surface and radiance fields, facilitating training and fine-tuning [S2].
- **Limitations:** Requires a well-defined SDF field; performance depends on the accuracy of the SDF; may fail if the surface is not well-represented by the SDF threshold [S1, S2].
- **When to Use:** When a high-quality SDF is available and rendering speed is a priority, particularly for human reconstruction and rendering where surface detail is critical [S2].

## Background
The method was proposed by Shao et al. in the context of the DoubleField network, which bridges neural surface and radiance fields for high-fidelity human reconstruction and rendering [S2]. It was introduced to accelerate the rendering process and strengthen the learned relationship between the surface and radiance fields [S2]. The approach is part of a broader trend of using implicit representations (SDFs) to guide sampling in neural rendering, as noted in recent surveys of learning-based 3D representations [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] DoubleField: Bridging the Neural Surface and Radiance Fields for High-fidelity Human Reconstruction and Rendering, https://doi.org/10.1109/cvpr52688.2022.01541, fetched 2026-10-29
