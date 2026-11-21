# Boundary Geometry Classification

Also called: Boundary Polygon Classification, Boundary Geometry Check

## Overview
Boundary Geometry Classification is a geometric filtering step used during octree construction and neighbor-based culling to distinguish surface geometry located on voxel cell boundary planes from internal geometry within complex blocks [S1, unofficial]. It solves the problem where neighbor occlusion tests would incorrectly prune or interact with internal fragments of non-manifold or complex shapes by ensuring that culling logic only applies to polygons strictly lying on the six faces of the current voxel cell [S1, unofficial]. It sets itself apart from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating at the individual polygon level to identify *which* geometry is eligible for neighbor interaction, rather than determining *how* to cull based on coverage masks.

## How it works
**Inputs and Outputs:**
- Input: A set of vertices defining a single polygon (triangle or quad), the axis of the cell boundary plane being tested, the coordinate value of that plane, and a tolerance epsilon.
- Output: A boolean indicating whether the polygon lies on the boundary plane.

**Data Structures:**
- `vertices`: An array of 3D vectors (`Vec3`), where each vector contains `(x, y, z)` coordinates.
- `axis`: An integer index (0, 1, or 2) corresponding to the x, y, or z axis.
- `plane`: A floating-point scalar representing the coordinate value of the cell boundary (e.g., $0.0$ or $1.0$ in local cell space).
- `eps`: A small floating-point tolerance value.

**Steps:**
1. Iterate through each vertex $v$ in the polygon's vertex set.
2. Extract the coordinate of $v$ along the specified `axis`.
3. Calculate the absolute difference between this coordinate and the `plane` value.
4. Check if this difference is less than or equal to `eps`.
5. If all vertices satisfy this condition, the polygon is classified as a boundary polygon. Otherwise, it is internal.

**Formulas:**
For a vertex $v = (v_x, v_y, v_z)$ and axis $a \in \{0, 1, 2\}$:
$$ |v_{axis} - plane| \le \epsilon $$
The polygon is a boundary polygon if:
$$ \forall v \in \text{vertices}, \quad |v_{axis} - plane| \le \epsilon $$

**Parameters:**
- `eps`: A small tolerance value used to account for floating-point precision errors. Typical values are not explicitly given in the sources, but it must be small enough to distinguish boundary geometry from internal geometry while robust to numerical noise [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point inaccuracies may cause vertices that are mathematically on the boundary to appear slightly off. The `eps` parameter must be chosen carefully to avoid false negatives (internal geometry misclassified as boundary) or false positives (internal geometry misclassified as boundary).
- **Degenerate Input:** If the polygon has coincident vertices, the check still holds, but the geometric significance may be null.
- **Chunk Borders:** The method treats chunk borders semantically identical to internal voxel neighbors; the only difference is the source of the neighbor data, not the classification logic [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lies_on_boundary(vertices: list, axis: int, plane: float, eps: float) -> bool:
    """
    Classify a polygon as a boundary polygon by checking if all its vertices 
    lie on the same cell boundary plane within a small tolerance.
    
    Args:
        vertices: List of Vec3 objects representing polygon vertices.
        axis: Integer index (0, 1, or 2) for x, y, or z axis.
        plane: Float value of the boundary plane coordinate.
        eps: Tolerance for boundary detection.
        
    Returns:
        True if all vertices lie on the boundary plane, False otherwise.
    """
    for v in vertices:
        # Extract coordinate along the specified axis
        coord = v[axis]
        # Check if within tolerance of the plane
        if abs(coord - plane) > eps:
            return False
    return True
```

## Subtopics

### [Complex Face and Edge Criterion](6790-complex-face-and-edge-criterion.md)
A method to identify cells containing surface fragments smaller than the current subdivision level, ensuring neighbor-based culling does not incorrectly prune internal geometry of complex blocks. It is applied after basic inhomogeneity checks to decide if a cell must be subdivided further.

## Operating characteristics
- **Time Complexity:** $O(V)$ per polygon, where $V$ is the number of vertices in the polygon (typically 3 or 4). This is a constant-time operation per polygon relative to the scene size.
- **Memory Complexity:** $O(1)$ auxiliary space, as it only requires local vertex data.
- **Strengths:** Allows arbitrary internal geometry to remain untouched while enabling efficient neighbor occlusion tests only on relevant boundary geometry [S1, unofficial].
- **Limitations:** Relies on a tolerance parameter `eps` which must be tuned for the specific precision and scale of the mesh.
- **When to use:** When performing neighbor-based culling in voxel-based meshing to avoid false positives on internal geometry of complex shapes.
- **When not to use:** When geometry is guaranteed to be manifold and fully contained within voxel cells without internal fragments.

## Background
Originating from techniques for face culling in voxel cube and partial shape rendering, this method was developed to optimize the rendering of complex boolean solids by separating boundary-relevant geometry from internal geometry [S1, unofficial]. It addresses the limitation of simple boolean solid flags, which are insufficient for partial shapes with mixed visible/hidden areas on a single face, by providing a geometric criterion for neighbor interaction eligibility.

## Sources
[S1] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
