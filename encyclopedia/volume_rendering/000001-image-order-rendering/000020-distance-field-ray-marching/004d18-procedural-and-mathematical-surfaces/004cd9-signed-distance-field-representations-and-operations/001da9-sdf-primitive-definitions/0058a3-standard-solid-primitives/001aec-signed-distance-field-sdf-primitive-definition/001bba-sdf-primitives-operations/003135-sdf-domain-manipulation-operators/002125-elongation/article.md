# Elongation


## Overview
Elongation modifies a Signed Distance Function (SDF) by applying a non-isometric domain transformation to stretch or compress geometry along a specified vector, creating tapered or elongated structures from base primitives [S1, unofficial]. It sets itself apart from uniform scaling by selectively deforming the spatial domain using absolute-value subtraction and distance correction, which is essential for modeling organic or tapered shapes like bones or fibers [G]. It builds on [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) by applying a non-isometric transformation to the query point [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Query point $\mathbf{p} \in \mathbb{R}^3$, elongation vector $\mathbf{s} \in \mathbb{R}^3$, and base SDF function $d_{\text{base}}(\cdot)$ [S1, unofficial].
- Output: Transformed distance value $d_{\text{elongated}}(\mathbf{p})$ [S1, unofficial].

**Data Structures:**
- $\mathbf{p}$: 3D vector $(p_x, p_y, p_z)$ representing the query position [S1, unofficial].
- $\mathbf{s}$: 3D vector $(s_x, s_y, s_z)$ defining the direction and magnitude of elongation [S1, unofficial].

**Steps:**
1. Compute the intermediate vector $\mathbf{q} = |\mathbf{p}| - \mathbf{s}$, where $|\mathbf{p}|$ denotes component-wise absolute value [S1, unofficial].
2. Evaluate the base SDF at the positive part of $\mathbf{q}$, defined as $\mathbf{q}_+ = \max(\mathbf{q}, 0.0)$ [S1, unofficial].
3. Add the maximum of the negative part of $\mathbf{q}$ (clamped to zero) to the result: $d_{\text{elongated}}(\mathbf{p}) = d_{\text{base}}(\mathbf{q}_+) + \max(-\mathbf{q}, 0.0)$ [S1, unofficial].

**Formulas:**
The core transformation follows:
$$ \mathbf{q} = |\mathbf{p}| - \mathbf{s} $$
$$ \mathbf{q}_+ = \max(\mathbf{q}, 0.0) $$
$$ d_{\text{elongated}}(\mathbf{p}) = d_{\text{base}}(\mathbf{q}_+) + \max(-\mathbf{q}, 0.0) $$
where $|\cdot|$ is the component-wise absolute value, $\max(\cdot, 0.0)$ clamps negative components to zero, and $\max(-\mathbf{q}, 0.0)$ extracts the negative components of $\mathbf{q}$ as positive corrections [S1, unofficial].

**Parameters:**
- $\mathbf{s}$: Elongation vector. Typical values depend on the desired shape (e.g., $s_y = 0.1$ for slight taper) [G].
- Base SDF: Any valid SDF (sphere, box, etc.) [G].

**Edge Cases and Pitfalls:**
- **Non-isometric Distortion**: Elongation is not an isometry, so the Lipschitz-1 property may be violated, leading to ray marching artifacts if step sizes are not reduced [G].
- **Precision**: Large elongation factors can cause numerical instability in the base SDF evaluation [G].
- **Boundary Conditions**: Ensure the base SDF handles the transformed coordinates correctly, especially if the transformation maps points outside the expected domain [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def elongation_sdf(p, s, base_sdf):
    # p: input position (vec3)
    # s: elongation vector (vec3)
    # base_sdf: function returning distance for a point
    q = (abs(p.x), abs(p.y), abs(p.z)) - s
    q_pos = (max(q.x, 0.0), max(q.y, 0.0), max(q.z, 0.0))
    d_raw = base_sdf(q_pos)
    correction = (max(-q.x, 0.0), max(-q.y, 0.0), max(-q.z, 0.0))
    return d_raw + sum(correction)
```

## Subtopics
- End-point Corrector: Addresses radius shrinkage and field variation divergence near skeleton end-points in vector field visualization [S1, unofficial]. It sets itself apart from other correction methods by forcing an additional sampling position near end-points to increase the correlation between the minima of individual primitives and the global field [S1, unofficial]. Choose it when visualizing vector fields with skeleton-based elongation to prevent geometric artifacts at termination points. [End-point Corrector](4063-end-point-corrector.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per sample, same as base SDF, plus overhead for transformation and correction [G].
- **Memory Complexity**: $O(1)$, no additional data structures [G].
- **Performance**: Fast, but may require smaller step sizes during ray marching due to non-isometric distortion, increasing sample count [G].
- **Strengths**: Enables creation of tapered and organic shapes from simple primitives [G].
- **Limitations**: Distortion can lead to artifacts; requires careful tuning of step sizes [G].
- **When to use**: When modeling tapered, organic, or elongated primitives from standard SDFs [G].
- **When not to use**: When strict isometric preservation or uniform scaling is required, or when ray marching step sizes cannot be dynamically adjusted [G].

## Background
- **Origin**: Not given in the sources.
- **History and development**: Elongation is a domain manipulation technique used in procedural modeling and ray marching for SDFs [S1, unofficial]. It allows for the creation of complex shapes by deforming the space in which the base SDF is evaluated [S1, unofficial].
- **Where it is used**: Widely used in real-time graphics for generating organic models and structures [G].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
