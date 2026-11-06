# Elongation

Also called: Elongation operator, Elongation transformation

## Overview
Elongation is a domain manipulation operator for Signed Distance Functions (SDFs) that stretches a primitive along a specific axis by warping the input space. It solves the problem of creating elongated geometric primitives (such as capsules or stretched spheres) without deriving a new analytical distance function from scratch. It sets itself apart from sibling operators like [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) by modifying the metric of the space rather than combining multiple shapes, and from [SDF Rounding](1829-sdf-rounding.md) by preserving the sharpness of the original primitive's features while scaling them.

## How it works
**Inputs and Outputs:**
- Input: Query point $\mathbf{p} = (x, y, z)$, elongation vector $\mathbf{s} = (s_x, s_y, s_z)$, and base SDF function $d_{base}$.
- Output: The elongated distance value $d_{result}$.

**Data Structures:**
No complex data structures are required. The operation is a point-wise transformation.

**Steps:**
1.  **Transform Position:** Compute an intermediate vector $\mathbf{q}$ by subtracting the elongation vector $\mathbf{s}$ from the absolute value of the input position $\mathbf{p}$.
    $$ \mathbf{q} = |\mathbf{p}| - \mathbf{s} $$
    *Note: The absolute value $|\mathbf{p}|$ is applied component-wise. This step effectively folds the space and shifts it, preparing it for the base evaluation.* [S1, unofficial]

2.  **Evaluate Base SDF:** Evaluate the base primitive's SDF at the positive part of $\mathbf{q}$. Let $d_{pos} = d_{base}(\max(\mathbf{q}, 0))$. [S1, unofficial]

3.  **Correct for Negative Parts:** Add the contribution of the negative parts of $\mathbf{q}$, clamped to zero. Let $d_{neg} = \max(\min(\mathbf{q}), 0)$. The final distance is:
    $$ d_{result} = d_{pos} + d_{neg} $$
    *Note: This ensures continuity and correct distance calculation when the transformed point crosses into negative coordinate space relative to the elongation shift.* [S1, unofficial]

**Parameters:**
- $\mathbf{s}$: The elongation vector. Typical values depend on the desired aspect ratio. For a capsule of radius $r$ and height $h$, $\mathbf{s}$ is often related to $(0, h/2, 0)$ or similar, depending on the base primitive. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** When $\mathbf{q}$ components are near zero, floating-point precision errors may cause slight artifacts in the distance field. [G]
- **Boundaries:** The operator assumes the base SDF is defined for all inputs. If the base SDF has singularities, they will be mapped to new locations in space.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def elongation(p, s, base_sdf):
    """
    Computes the elongated SDF value.
    
    Args:
        p: Input position vector (x, y, z)
        s: Elongation vector (sx, sy, sz)
        base_sdf: Function evaluating the base primitive's SDF
        
    Returns:
        Elongated distance value
    """
    # Step 1: Transform position
    # Component-wise absolute value of p
    abs_p = [abs(p[0]), abs(p[1]), abs(p[2])]
    # Subtract elongation vector s
    q = [abs_p[0] - s[0], abs_p[1] - s[1], abs_p[2] - s[2]]
    
    # Step 2: Evaluate base SDF at positive part of q
    # max(q, 0) is component-wise
    q_pos = [max(q[0], 0.0), max(q[1], 0.0), max(q[2], 0.0)]
    d_pos = base_sdf(q_pos)
    
    # Step 3: Add maximum of negative part of q (clamped to 0)
    # min(q) finds the smallest component, then clamp to 0
    min_q = min(q[0], q[1], q[2])
    d_neg = max(min_q, 0.0)
    
    return d_pos + d_neg
```

## Subtopics

### End-point Corrector
A technique used in the context of Scale-Invariant Integral Surfaces (SCALIS) to address field variation divergence near skeleton end-points. It forces an additional sampling position near end-points to increase correlation between the minima of individual primitives and the global field. It is chosen when rendering skeletal structures where radius shrinkage at endpoints is visible. The implementation involves adding a segment primitive with constant radius at the end vertices of the skeleton. The length of this corrector segment is proportional to the prescribed radius at the end vertices and a constant depending on the kernel used [S2]. If absent, a heuristic can compensate [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, assuming the base SDF is $O(1)$. It adds a constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$. No additional memory is required.
- **Strengths:** Simple to implement; allows reuse of existing base SDFs to create varied shapes; computationally cheap.
- **Limitations:** The resulting distance field may not be perfectly smooth if the base SDF or the transformation introduces discontinuities in derivatives. The elongation is anisotropic and may distort the metric in ways that affect ray marching step sizes if not accounted for.
- **When to use:** When creating stretched primitives (capsules, stretched spheres) for ray marching or volume rendering.
- **When not to use:** When high-order continuity is required across the elongated regions, or when the elongation vector is dynamic and causes frequent re-evaluation of complex base SDFs.

## Background
The elongation operator is a standard technique in procedural geometry and implicit surface rendering. The specific formulation described in [S1, unofficial] (unofficial) is part of a collection of SDF primitives and transformations implemented in a Python-based sphere tracer. The end-point corrector variant is described in the context of ray tracing scale-invariant integral surfaces [S2].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
