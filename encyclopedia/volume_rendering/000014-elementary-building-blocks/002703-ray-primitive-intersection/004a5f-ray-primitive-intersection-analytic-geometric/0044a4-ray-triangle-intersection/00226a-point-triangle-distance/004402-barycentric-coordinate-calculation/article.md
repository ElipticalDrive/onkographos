# Barycentric Coordinate Calculation
Also called: Barycentric Coordinates, Point-in-Triangle Barycentric Coordinates
## Overview
Computes the barycentric coordinates of a 3D point relative to a triangle defined by three vertices [S1, unofficial]. It solves the problem of expressing a point's position within a triangle's local coordinate system, which is a prerequisite for [Point-Triangle Distance](2776-point-triangle-distance.md) [G]. It sets itself apart from sibling methods by focusing on the geometric projection and area-based calculation rather than distance minimization or mesh sampling [G].
## How it works
**Inputs:** A 3D point $P$, and three triangle vertices $p_1, p_2, p_3$ [S1, unofficial].
**Outputs:** A 3D vector $(\lambda_1, \lambda_2, \lambda_3)$ representing the barycentric coordinates [S1, unofficial].

**Data Structures:**
- `Vector3`: A 3-component floating-point vector [G].
- `norm`, `pointProj`: 3D vectors [G].
- `a1, a2, a3`: Scalar floats representing sub-triangle areas [G].

**Steps:**
1. Compute the triangle normal:
   $$ \text{norm} = \text{Cross}(p_2 - p_1, p_3 - p_1) $$ [S1, unofficial].
2. Project the point $P$ onto the triangle's plane to handle points not exactly on the plane:
   $$ \text{pointProj} = P - \frac{\text{Dot}(P - p_1, \text{norm})}{\|\text{norm}\|^2} \text{norm} $$ [S1, unofficial]. [G] If $\|\text{norm}\|^2$ is zero, the triangle is degenerate and the method fails.
3. Calculate the area of the sub-triangle opposite $p_1$ (formed by $\text{pointProj}, p_2, p_3$):
   $$ a_1 = \text{Area}(\text{pointProj}, p_2, p_3) $$ [S1, unofficial].
4. Calculate the area of the sub-triangle opposite $p_2$ (formed by $p_1, \text{pointProj}, p_3$):
   $$ a_2 = \text{Area}(p_1, \text{pointProj}, p_3) $$ [S1, unofficial].
5. Calculate the area of the sub-triangle opposite $p_3$ (formed by $p_1, p_2, \text{pointProj}$):
   $$ a_3 = \text{Area}(p_1, p_2, \text{pointProj}) $$ [S1, unofficial].
6. Return the normalized barycentric coordinates:
   $$ (\lambda_1, \lambda_2, \lambda_3) = \frac{(a_1, a_2, a_3)}{a_1 + a_2 + a_3} $$ [S1, unofficial].

**Formula:**
The area of a triangle with vertices $A, B, C$ is calculated as:
$$ \text{Area} = \frac{\|\text{Cross}(B - A, C - A)\|}{2} $$ [S1, unofficial].

**Pitfalls:**
- If the point is not on the triangle's plane, it is projected onto the plane before area calculation [S1, unofficial].
- If the triangle is degenerate (area zero), division by zero occurs in the final normalization step [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def cross(v1, v2):
    return (v1[1]*v2[2] - v1[2]*v2[1],
            v1[2]*v2[0] - v1[0]*v2[2],
            v1[0]*v2[1] - v1[1]*v2[0])

def calculate_triangle_area(p1, p2, p3):
    """Returns the area of the triangle defined by p1, p2, p3."""
    v1 = (p2[0]-p1[0], p2[1]-p1[1], p2[2]-p1[2])
    v2 = (p3[0]-p1[0], p3[1]-p1[1], p3[2]-p1[2])
    c = cross(v1, v2)
    return math.sqrt(dot(c, c)) / 2.0

def to_barycentric_coordinates(point, p1, p2, p3):
    """
    Computes barycentric coordinates of point relative to triangle p1, p2, p3.
    Returns (lambda1, lambda2, lambda3).
    """
    # 1. Compute triangle normal
    v1 = (p2[0]-p1[0], p2[1]-p1[1], p2[2]-p1[2])
    v2 = (p3[0]-p1[0], p3[1]-p1[1], p3[2]-p1[2])
    norm = cross(v1, v2)

    # 2. Project point onto triangle's plane
    dot_val = dot((point[0]-p1[0], point[1]-p1[1], point[2]-p1[2]), norm)
    norm_sq = dot(norm, norm)

    if norm_sq == 0:
        raise ValueError("Degenerate triangle: zero area.")

    point_proj = (point[0] - (dot_val / norm_sq) * norm[0],
                  point[1] - (dot_val / norm_sq) * norm[1],
                  point[2] - (dot_val / norm_sq) * norm[2])

    # 3. Calculate areas of sub-triangles
    a1 = calculate_triangle_area(point_proj, p2, p3)
    a2 = calculate_triangle_area(p1, point_proj, p3)
    a3 = calculate_triangle_area(p1, p2, point_proj)

    # 4. Normalize
    total_area = a1 + a2 + a3
    if total_area == 0:
        raise ValueError("Total sub-triangle area is zero.")

    return (a1 / total_area, a2 / total_area, a3 / total_area)
```
## Subtopics

### Barycentric Coordinate Rasterization Derivatives
A method to calculate rasterization derivatives using barycentric coordinates to overcome occlusion discontinuity by treating shapes as locally planar [S2, unofficial]. It calculates barycentric coordinates of each triangle with respect to each pixel, introducing negative values for the barycentric coordinates of pixels lying outside the triangle border [S2, unofficial]. This approximation allows shapes to be treated as locally planar to approximate occlusion boundaries [S2, unofficial]. However, this approximation poses problems when optimizing for translation or occlusion [S2, unofficial].

## Operating characteristics
Time complexity: $O(1)$ per point-triangle pair [G]. Memory complexity: $O(1)$ [G]. Strengths: Simple geometric interpretation; handles off-plane points via projection [G]. Limitations: Fails on degenerate triangles; projection step adds computational cost compared to planar-only methods [S1, unofficial].
## Background
Origin: Standard computational geometry technique [G]. History: Used in graphics for interpolation, texture mapping, and point-in-triangle tests [G]. Usage: Found in [Point-Triangle Distance](2776-point-triangle-distance.md) and differentiable rendering pipelines [S1, unofficial][S2, unofficial].
## Sources
[S1] xandergos/unity-sparse-voxel-octrees, https://github.com/xandergos/unity-sparse-voxel-octrees, fetched 2026-10-26 (unofficial)
[S2] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
