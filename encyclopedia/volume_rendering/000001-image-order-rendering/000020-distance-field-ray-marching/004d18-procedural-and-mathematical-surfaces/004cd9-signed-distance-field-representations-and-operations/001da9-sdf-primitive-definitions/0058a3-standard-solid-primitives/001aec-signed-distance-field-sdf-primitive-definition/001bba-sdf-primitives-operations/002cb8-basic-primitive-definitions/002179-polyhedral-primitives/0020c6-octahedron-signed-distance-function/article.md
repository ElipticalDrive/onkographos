# Octahedron Signed Distance Function

Also called: Octahedron SDF, sdOctahedron

## Overview
Computes the exact signed distance from a query point to the surface of a regular octahedron centered at the origin. It solves the geometric query required for ray marching or collision detection involving octahedral primitives. It is distinguished from other polyhedral SDFs by its reliance on coordinate permutation to handle the octahedron's symmetry, reducing the problem to a single face evaluation [S1, unofficial].

## How it works
**Inputs:**
- $p \in \mathbb{R}^3$: The query point in local space.
- $s \in \mathbb{R}$: The size parameter. In the source implementation, this represents the distance from the center to the vertices along the axes (the "radius" of the octahedron).

**Outputs:**
- $d \in \mathbb{R}$: The signed distance. Negative inside, positive outside, zero on the surface.

**Data Structures:**
- Standard 3D vector $p = (p_x, p_y, p_z)$.

**Steps:**
1.  **Symmetry Reduction:** Compute $p' = |p| = (|p_x|, |p_y|, |p_z|)$. This folds the space into the first octant, exploiting the octahedron's symmetry [S1, unofficial].
2.  **Central Region Check:** Calculate $m = p'_x + p'_y + p'_z - s$.
    - If $3 \cdot p'_x < m$, the point is closest to the face normal to the x-axis. Set $q = p'$.
    - Else if $3 \cdot p'_y < m$, the point is closest to the face normal to the y-axis. Set $q = (p'_y, p'_z, p'_x)$ (permutation).
    - Else if $3 \cdot p'_z < m$, the point is closest to the face normal to the z-axis. Set $q = (p'_z, p'_x, p'_y)$ (permutation).
    - Else, the point is in the central region (closest to the origin relative to the face planes). Return $d = m \cdot \tan(\pi/6)$. Note: $\tan(\pi/6) \approx 0.57735027$ [S1, unofficial].
3.  **Face Distance Calculation:** For the selected $q = (q_x, q_y, q_z)$:
    - Calculate $k = \text{clamp}(0.5 \cdot (q_z - q_y + s), 0, s)$.
    - The distance to the face is the Euclidean norm of the vector from the query point to the closest point on the face plane within the face bounds:
      $$ d = \sqrt{q_x^2 + (q_y - s + k)^2 + (q_z - k)^2} $$
    - Return $d$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_octahedron(p: list[float], s: float) -> float:
    """
    Computes the signed distance to an octahedron of size s.
    p: [x, y, z]
    s: size (distance from center to vertex)
    """
    # Step 1: Symmetry reduction
    px, py, pz = abs(p[0]), abs(p[1]), abs(p[2])
    
    # Step 2: Central region check
    m = px + py + pz - s
    
    # Determine closest face via permutation
    # Note: The conditions 3*p < m identify which coordinate is the "smallest" relative to the sum,
    # effectively identifying the closest face normal direction in the folded space.
    if 3.0 * px < m:
        qx, qy, qz = px, py, pz
    elif 3.0 * py < m:
        qx, qy, qz = py, pz, px
    elif 3.0 * pz < m:
        qx, qy, qz = pz, px, py
    else:
        # Central region: closest to origin
        return m * math.tan(math.pi / 6.0)
    
    # Step 3: Face distance
    # k is the projection onto the edge direction to find the closest point on the face
    k = max(0.0, min(s, 0.5 * (qz - qy + s)))
    
    # Vector from q to the closest point on the face
    dx = qx
    dy = qy - s + k
    dz = qz - k
    
    return math.sqrt(dx*dx + dy*dy + dz*dz)
```

**Parameters:**
- $s$: Typical values depend on scene scale. Must be positive.

**Edge Cases:**
- $s=0$: The octahedron is a point. The function returns $|p_x| + |p_y| + |p_z|$ (L1 norm) if not in central region logic, but the central region logic dominates near origin. Specifically, if $p=0$, $m=-s$, returns $-s \tan(\pi/6)$.
- Precision: The permutation logic relies on strict inequalities. Points exactly on the boundaries between regions (e.g., $3p_x = m$) are handled by the `elif` chain, ensuring continuity.

## Subtopics

### Equilateral Triangle Signed Distance Function
Computes the signed distance to an equilateral triangle with inscribed radius $r$. It transforms the query point into a canonical orientation, reflects it if necessary to handle symmetry, clamps to the triangle bounds, and returns the distance. It is chosen when 2D triangular primitives are needed in SDF workflows. Based on Inigo Quilez's exact function [S2, unofficial].

**Steps:**
1.  $k = \sqrt{3.0}$.
2.  Transform $p$: $p_x = |p_x| - r$, $p_y = p_y + r/k$.
3.  If $p_x + k \cdot p_y > 0.0$, reflect: $p = \frac{1}{2}(p_x - k \cdot p_y, -k \cdot p_x - p_y)$.
4.  Clamp $x$: $p_x = p_x - \text{clamp}(p_x, -2r, 0)$.
5.  Return $-|p| \cdot \text{sign}(p_y)$.

## Operating characteristics
- **Time Complexity:** $O(1)$. Constant number of arithmetic operations, comparisons, and one square root.
- **Memory Complexity:** $O(1)$. No auxiliary data structures.
- **Strengths:** Exact analytical solution; no approximation errors; differentiable almost everywhere (gradients can be computed via finite differences or analytical derivation for normals).
- **Limitations:** Only applies to regular octahedra. Irregular octahedra require decomposition or approximation.
- **When to use:** Ray marching, collision detection, or procedural modeling involving octahedral shapes.

## Background
The octahedron SDF is a standard primitive in the family of procedural SDFs popularized by Inigo Quilez and others in the ray marching community. It is derived from the geometric properties of the regular octahedron, specifically its symmetry and planar faces. The implementation cited here follows the GLSL convention `sdOctahedron` [S1, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
