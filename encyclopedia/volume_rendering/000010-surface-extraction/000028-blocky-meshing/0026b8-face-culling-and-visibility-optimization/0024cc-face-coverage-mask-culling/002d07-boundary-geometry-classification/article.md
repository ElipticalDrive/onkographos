# Boundary Geometry Classification

Also called: Boundary Polygon Classification, Boundary Geometry Check

## Overview
Boundary Geometry Classification is a method used in blocky meshing to distinguish between geometry located on the voxel cell boundary planes and geometry located in the interior of a complex block [S1, unofficial]. It solves the problem where neighbor-based face culling must be restricted to external surfaces to avoid incorrectly culling internal geometry that is occluded by the block's own internal structure [S1, unofficial]. It sets itself apart from siblings like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by providing the geometric predicate required to determine *which* faces are eligible for neighbor occlusion tests, rather than performing the culling logic itself.

## How it works
**Inputs and Outputs:**
- Input: A polygon defined by a list of 3D vertices (`vertices`), the axis of the face being tested (`axis`), the coordinate of the cell boundary plane (`plane`), and a small tolerance value (`eps`).
- Output: A boolean indicating whether the polygon lies on the boundary.

**Data Structures:**
- `vertices`: An array of 3D vectors (`Vec3`), where each vector contains `(x, y, z)` coordinates.
- `axis`: An integer index (0, 1, or 2) corresponding to the x, y, or z axis perpendicular to the face.
- `plane`: A floating-point scalar representing the coordinate value of the voxel cell boundary (e.g., $x=0$, $x=1$).
- `eps`: A floating-point tolerance value for numerical stability.

**Steps:**
1. Iterate through each vertex $v$ in the polygon's vertex list.
2. For each vertex, extract the coordinate corresponding to the specified `axis`.
3. Calculate the absolute difference between this coordinate and the `plane` value.
4. Check if this difference is less than or equal to `eps`.
5. If all vertices satisfy this condition, the polygon is classified as a boundary polygon. Otherwise, it is internal.

**Formula:**
A polygon with vertices $V = \{v_0, v_1, \dots, v_n\}$ is on the boundary if:
$$ \forall v \in V, \quad |v_{\text{axis}} - \text{plane}| \le \text{eps} $$
Where $v_{\text{axis}}$ is the component of vertex $v$ along the specified axis.

**Parameters:**
- `eps`: A small tolerance value. The exact value is not specified in the sources [S1, unofficial]. It must be small enough to distinguish boundary geometry from internal geometry but large enough to handle floating-point precision errors.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point inaccuracies in vertex generation may cause vertices that are theoretically on the boundary to fail the check. A non-zero `eps` is required [S1, unofficial].
- **Degenerate Input:** If a polygon has no vertices, the function should return false or handle the empty case explicitly [G].
- **Internal Geometry:** Geometry strictly inside the voxel (not on any of the 6 boundary planes) will fail this check for all axes and should be excluded from neighbor culling tests to prevent incorrect occlusion [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lies_on_boundary(vertices: list, axis: int, plane: float, eps: float) -> bool:
    """
    Classifies a polygon as a boundary polygon if all its vertices lie on the same cell boundary plane.
    
    Args:
        vertices: List of 3D vectors (x, y, z).
        axis: Index of the axis (0=x, 1=y, 2=z) perpendicular to the face.
        plane: The coordinate value of the boundary plane.
        eps: Tolerance for boundary detection.
        
    Returns:
        True if all vertices are within eps of the plane, False otherwise.
    """
    # Handle empty vertex list
    if not vertices:
        return False
        
    for v in vertices:
        # Extract coordinate along the specified axis
        coord = v[axis]
        # Check if within tolerance of the boundary plane
        if abs(coord - plane) > eps:
            return False
            
    return True
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(V)$, where $V$ is the number of vertices in the polygon. Since polygons in blocky meshing are typically triangles ($V=3$), this is effectively $O(1)$.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Simple, fast, and robust for separating internal from external geometry. It allows the mesher to keep arbitrary internal geometry while applying neighbor occlusion only where it makes spatial sense [S1, unofficial].
- **Limitations:** Relies on a tolerance value `eps` which must be chosen carefully. It does not handle topological complexity within homogeneous regions; that is handled by the [Complex Face and Edge Criterion](6790-complex-face-and-edge-criterion.md) [S1, unofficial].

## Background
This method is part of the face culling pipeline for voxel cubes and partial shapes, specifically designed to support neighbor-based occlusion culling in blocky meshing [S1, unofficial]. It ensures that internal triangles inside a complex block are not tested against neighboring blocks, which would be incorrect [S1, unofficial].

## Sources
[S1] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
