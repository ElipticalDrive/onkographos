# Triangle Point Generation (Linear Interpolation with Snap)
Also called: Linear Interpolation with Snap, Snap Interpolation
## Overview
This method computes the precise intersection point on a triangle edge where a scalar field equals a threshold, using linear interpolation between two grid vertices. It distinguishes itself from siblings like [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md) by avoiding iterative convergence, instead applying a deterministic "snap" heuristic to vertices near the threshold to stabilize the interpolation result [S1, unofficial]. It serves as the geometric foundation for [Triangle Point Sampling](17806-triangle-point-sampling.md), which requires robust point generation to ensure uniform spatial distribution [S1, unofficial].
## How it works
**Inputs:**
- $p_1, p_2$: 3D coordinates of the two vertices defining the edge.
- $v_1, v_2$: Scalar field values at $p_1$ and $p_2$ respectively.
- $snap$: Threshold parameter for snapping. Default: $0.01$ [S1, unofficial].

**Outputs:**
- $point$: The 3D coordinate of the intersection point on the edge.

**Data Structures:**
- No complex structures required; operates on scalar values and 3D vectors.

**Steps:**
1. Check if $|v_1| < snap$. If true, return $p_1$ [S1, unofficial].
2. Check if $|v_2| < snap$. If true, return $p_2$ [S1, unofficial].
3. Check if $|v_2 - v_1| < snap$. If true, return $p_1$ [S1, unofficial].
4. Calculate the interpolation factor $\mu$:
   $$ \mu = \frac{v_2}{v_2 - v_1} $$
5. Calculate the intersection point:
   $$ point = \mu \cdot p_1 + (1 - \mu) \cdot p_2 $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triangle_point_generation(p1, v1, p2, v2, snap=0.01):
    # Step 1: Check if v1 is close to zero
    if abs(v1) < snap:
        return p1
    
    # Step 2: Check if v2 is close to zero
    if abs(v2) < snap:
        return p2
    
    # Step 3: Check if the difference between v1 and v2 is small
    if abs(v2 - v1) < snap:
        return p1
    
    # Step 4: Calculate mu
    mu = v2 / (v2 - v1)
    
    # Step 5: Calculate point
    point = mu * p1 + (1 - mu) * p2
    
    return point
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** The formula for $\mu$ divides by $v_2 - v_1$. The check $|v_2 - v_1| < snap$ prevents this when the denominator is near zero [S1, unofficial].
- **Precision:** The snap threshold handles numerical instability when values are very close to the threshold or to each other [S1, unofficial].

## Subtopics

### [Triangle Point Sampling](17806-triangle-point-sampling.md)
Provides probabilistic selection and coordinate transformation for uniform mesh sampling, building on this method's point generation.

## Operating characteristics
- **Time Complexity:** $O(1)$ constant time.
- **Memory Complexity:** $O(1)$ constant space.
- **Strengths:** Simple, fast, and stable for edges near the threshold due to snapping [S1, unofficial].
- **Limitations:** The snapping behavior may introduce small geometric inaccuracies compared to pure linear interpolation when high precision is required away from the threshold [S1, unofficial].
- **When to use:** When robustness near the threshold is more critical than perfect geometric accuracy, or when performance is paramount.
- **When not to use:** When exact linear interpolation is required without heuristic snapping.

## Background
Originated in the context of marching tets reconstruction algorithms, specifically within the `marchingtets-reconstruction` project [S1, unofficial]. It is used in surface reconstruction pipelines where stable edge intersection points are needed for generating triangle meshes from scalar fields [S1, unofficial].

## Sources
[S1] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
