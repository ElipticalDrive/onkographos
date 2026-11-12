# Triangular Prism Signed Distance Function

Also called: Signed Triangle Prism, Triangular Prism SDF, sdTriPrism

## Overview
Computes the signed distance from a 3D point to an equilateral triangular prism aligned with the Z-axis. It differs from general polyhedral SDFs by exploiting the symmetry of the equilateral cross-section to use a closed-form analytical solution, avoiding iterative projection or mesh intersection.

## How it works
**Inputs:**
- `p`: Query point $(x, y, z)$.
- `h`: Parameter vector $(h_x, h_y)$. $h_x$ controls the triangle size, $h_y$ is the half-height.

**Outputs:**
- Scalar signed distance $d$. Positive outside, negative inside.

**Data Structures:**
- None required. Pure arithmetic.

**Steps:**
1. Compute absolute coordinates: $q = |p|$.
2. Compute the distance to the triangular cross-section in the XY plane.
3. Combine with the Z-axis distance.

**Formula:**
$$
q = |p|
$$
$$
d = \max(q_z - h_y, \max(q_x \cdot \frac{\sqrt{3}}{2} + q_y \cdot \frac{1}{2}, -q_y) - h_x \cdot \frac{1}{2})
$$

Where:
- $q_x, q_y, q_z$ are the components of $q$.
- $\frac{\sqrt{3}}{2} \approx 0.866025$.
- $h_x$ is the half-width of the triangle base (distance from center to side).
- $h_y$ is the half-height of the prism.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sd_tri_prism(p: np.ndarray, h: np.ndarray) -> float:
    """
    Compute signed distance to a triangular prism.
    
    Args:
        p: Query point (x, y, z).
        h: Parameters (h_x, h_y). h_x is half-width of triangle, h_y is half-height.
    
    Returns:
        Signed distance.
    """
    # Step 1: Absolute coordinates
    q = np.abs(p)
    qx, qy, qz = q[0], q[1], q[2]
    hx, hy = h[0], h[1]
    
    # Step 2: Triangle distance in XY plane
    # The term max(qx * sqrt(3)/2 + qy * 1/2, -qy) computes the distance 
    # to the boundary of an equilateral triangle centered at origin, 
    # oriented with one vertex pointing up (or down depending on convention).
    # Note: The source formula uses +qy*0.5. This orientation assumes a specific 
    # alignment of the triangle vertices relative to the axes.
    
    tri_dist = max(qx * 0.866025 + qy * 0.5, -qy) - hx * 0.5
    
    # Step 3: Combine with Z distance
    d = max(qz - hy, tri_dist)
    
    return d
```

**Edge Cases:**
- **Precision:** The constant $0.866025$ should be computed as `np.sqrt(3)/2` or a high-precision literal to avoid artifacts near the triangle edges.
- **Orientation:** The formula assumes a specific orientation of the equilateral triangle in the XY plane. Rotating the triangle requires rotating the input point $p$ before applying the formula.

## Subtopics

### [Octahedron Signed Distance Function](2339-octahedron-signed-distance-function.md)
Dual polyhedron, different symmetry.

### [Hexagonal Prism Signed Distance Function](2348-hexagonal-prism-signed-distance-function.md)
Six-sided base, similar structure but different constants.

### [Dodecahedron Modulo](3737-dodecahedron-modulo.md)
Complex polyhedron, requires modulo operations.

## Operating characteristics
- **Time Complexity:** $O(1)$ constant time.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Extremely fast, no branching for distance calculation (only max), suitable for real-time ray marching.
- **Limitations:** Only supports equilateral triangles. Non-equilateral prisms require different formulations or approximations.

## Background
The formula is derived from the analytical solution for the distance to an equilateral triangle, extended to a prism by taking the maximum with the height constraint. It is commonly used in ray marching for procedural geometry.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
