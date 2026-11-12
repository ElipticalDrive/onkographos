# Elongation

Also called: Elongation operator, elongation transform

## Overview
Elongation modifies a Signed Distance Function (SDF) to stretch or compress geometry along a specific axis or vector, effectively creating tapered or tapered-like structures from base primitives. It sets itself apart from uniform scaling by allowing non-uniform, directional deformation of the domain, which is essential for modeling organic or tapered shapes like bones, fibers, or nozzles. It builds on [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) by applying a non-isometric transformation to the query point.

## How it works
**Inputs and Outputs:**
- Input: Query point $\mathbf{p} \in \mathbb{R}^3$, elongation vector $\mathbf{s}$, and base SDF function $d_{base}(\mathbf{p})$.
- Output: Transformed distance value $d_{elongated}(\mathbf{p})$.

**Data Structures:**
- $\mathbf{p}$: 3D vector $(p_x, p_y, p_z)$.
- $\mathbf{s}$: 3D vector $(s_x, s_y, s_z)$ defining the direction and magnitude of elongation. In many implementations, $\mathbf{s}$ is aligned with a principal axis (e.g., $\mathbf{s} = (0, s, 0)$ for Y-axis elongation).

**Steps:**
1. **Transform Position**: Compute the transformed point $\mathbf{p}'$ by applying the elongation vector to $\mathbf{p}$. The specific transformation depends on the implementation, but generally involves subtracting the elongation component from the absolute value of the position components to create a tapering effect, or applying a linear scaling factor based on the position along the elongation axis.
2. **Evaluate Base SDF**: Query the base SDF at the transformed point $\mathbf{p}'$ to get the raw distance $d_{raw} = d_{base}(\mathbf{p}')$.
3. **Adjust Distance**: Modify $d_{raw}$ to account for the distortion introduced by the non-isometric transformation. This often involves adding a correction term derived from the elongation vector or the difference between the original and transformed positions.

**Formulas:**
A common implementation for elongation along the Y-axis with vector $\mathbf{s} = (0, s, 0)$ involves:
$$ \mathbf{p}' = (p_x, p_y + s \cdot p_z, p_z) $$
Or more complex tapering:
$$ \mathbf{p}' = (p_x \cdot (1 - s \cdot p_y), p_y, p_z \cdot (1 - s \cdot p_y)) $$
The final distance is:
$$ d_{elongated}(\mathbf{p}) = d_{base}(\mathbf{p}') + \text{correction}(\mathbf{p}, \mathbf{p}') $$
The correction term ensures the Lipschitz continuity is maintained or approximated, often calculated as the difference in distance due to the stretching.

**Parameters:**
- $\mathbf{s}$: Elongation vector. Typical values depend on the desired shape (e.g., $s=0.1$ for slight taper).
- Base SDF: Any valid SDF (sphere, box, etc.).

**Edge Cases and Pitfalls:**
- **Non-isometric Distortion**: Elongation is not an isometry, so the Lipschitz-1 property may be violated, leading to ray marching artifacts if step sizes are not reduced [S4, unofficial].
- **Precision**: Large elongation factors can cause numerical instability in the base SDF evaluation.
- **Boundary Conditions**: Ensure the base SDF handles the transformed coordinates correctly, especially if the transformation maps points outside the expected domain.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def elongation_sdf(p, s, base_sdf):
    # p: input position (vec3)
    # s: elongation vector (vec3)
    # base_sdf: function returning distance for a point
    
    # Transform position
    # Example: Elongation along Y axis
    # p_prime = (p.x * (1 - s.y * p.y), p.y, p.z * (1 - s.y * p.y))
    # Note: Specific transformation logic depends on the desired elongation type
    
    # For a general elongation vector s, a simple linear transform might be:
    p_prime = p + s * dot(p, s) # Example: stretching along s direction
    
    # Evaluate base SDF
    d_raw = base_sdf(p_prime)
    
    # Correction term (simplified)
    # This part is implementation-specific and may require numerical differentiation
    # or analytical derivation based on the base SDF and transformation
    correction = 0.0 # Placeholder for correction logic
    
    return d_raw + correction
```

## Subtopics

## Operating characteristics
- **Time Complexity**: $O(1)$ per sample, same as base SDF, plus overhead for transformation and correction.
- **Memory Complexity**: $O(1)$, no additional data structures.
- **Performance**: Fast, but may require smaller step sizes during ray marching due to non-isometric distortion, increasing sample count [S4, unofficial].
- **Strengths**: Enables creation of tapered and organic shapes from simple primitives.
- **Limitations**: Distortion can lead to artifacts; requires careful tuning of step sizes.

## Background
Elongation is a domain manipulation technique used in procedural modeling and ray marching for SDFs. It allows for the creation of complex shapes by deforming the space in which the base SDF is evaluated. The technique is widely used in real-time graphics for generating organic models and structures.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
