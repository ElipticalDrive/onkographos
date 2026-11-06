# Boundary Geometry Classification

Also called: Boundary Polygon Classification, Boundary Geometry Check

## Overview
Boundary Geometry Classification is a filtering step in blocky meshing that ensures neighbor-based face culling is applied exclusively to geometry lying on voxel cell boundary planes, while leaving internal geometry of complex blocks untouched [S1, unofficial]. It solves the problem where culling logic intended for inter-block occlusion would incorrectly suppress valid internal triangles within a single complex shape [S1, unofficial]. It differs from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating on the geometric position of vertices rather than precomputed coverage bitmasks, and from [Face-Culling Geometry Shader](3928-face-culling-geometry-shader.md) by being a CPU-side classification logic rather than a GPU shader stage.

## How it works
**Inputs and Outputs:**
- Input: A list of generated polygons (triangles), each defined by a set of 3D vertices.
- Output: A boolean classification for each polygon indicating whether it is a "boundary polygon" eligible for neighbor culling.

**Data Structures:**
- Polygon: A collection of vertices. Each vertex is a 3D vector $(x, y, z)$.
- Cell Boundary Plane: Defined by an axis index $a \in \{0, 1, 2\}$ (corresponding to $x, y, z$) and a plane coordinate $P$.

**Steps:**
1. For each generated polygon, determine if it qualifies as a boundary polygon.
2. A polygon is classified as a boundary polygon if all its vertices lie on the same cell boundary plane within a specified tolerance $\epsilon$.
3. The check is performed for each of the six possible boundary planes of the voxel cell containing the polygon. If the condition holds for any one plane, the polygon is marked as boundary geometry.

**Formulas:**
A polygon with vertices $V = \{v_0, v_1, v_2\}$ lies on the boundary plane defined by axis $a$ and coordinate $P$ if:
$$ \forall v \in V, \quad |v[a] - P| \le \epsilon $$
Where:
- $v[a]$ is the coordinate of vertex $v$ along axis $a$.
- $P$ is the coordinate of the cell boundary (e.g., integer voxel coordinate).
- $\epsilon$ is a small tolerance value.

**Parameters:**
- $\epsilon$: A small tolerance value used to determine if a vertex lies on the boundary plane. [S1, unofficial] Typical values are not specified in the sources.

**Edge Cases and Pitfalls:**
- **Internal Geometry:** Complex blocks may contain internal triangles that do not lie on any cell boundary plane. These must not be tested against neighboring blocks [S1, unofficial].
- **Tolerance:** The choice of $\epsilon$ is critical. Too small a value may fail to classify vertices that are numerically on the boundary due to floating-point precision; too large a value may incorrectly classify internal geometry as boundary geometry. [S1, unofficial]
- **Chunk Borders:** Faces on the edge of a chunk are not semantically special; they still have neighbor voxels, just retrieved from a different source. The classification logic remains the same [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lies_on_boundary(vertices: list, axis: int, plane: float, eps: float) -> bool:
    """
    Check if all vertices lie on the boundary plane defined by axis and plane.
    
    Args:
        vertices: List of 3D vectors (x, y, z).
        axis: Index of the axis (0, 1, or 2) corresponding to the boundary plane.
        plane: The coordinate of the boundary plane.
        eps: Tolerance for the boundary check.
        
    Returns:
        True if all vertices are within eps of the plane, False otherwise.
    """
    for v in vertices:
        if abs(v[axis] - plane) > eps:
            return False
    return True

def classify_boundary_polygon(vertices: list, cell_bounds: dict, eps: float) -> bool:
    """
    Classify a polygon as a boundary polygon if it lies on any of the 6 cell boundary planes.
    
    Args:
        vertices: List of 3D vectors defining the polygon.
        cell_bounds: Dictionary containing the min and max coordinates for the cell in each dimension.
                     Example: {'min': [x_min, y_min, z_min], 'max': [x_max, y_max, z_max]}
        eps: Tolerance for the boundary check.
        
    Returns:
        True if the polygon is a boundary polygon, False otherwise.
    """
    # Check the 6 faces of the voxel cell
    # Faces are defined by axis and the min/max coordinate of that axis
    boundaries = [
        (0, cell_bounds['min'][0]), (0, cell_bounds['max'][0]),
        (1, cell_bounds['min'][1]), (1, cell_bounds['max'][1]),
        (2, cell_bounds['min'][2]), (2, cell_bounds['max'][2])
    ]
    
    for axis, plane in boundaries:
        if lies_on_boundary(vertices, axis, plane, eps):
            return True
            
    return False
```

## Subtopics

### Complex Face and Edge Criterion
This variant extends boundary classification to detect cells containing surface pieces smaller than the current subdivision level, specifically for octree-based adaptive meshing [S2].
- **Complex Face Criterion:** Detects connected components of voxels on homogeneous cell faces (where all vertices are inside or outside the surface $\partial V$) [S2]. It ensures that all cells intersected by $\partial V$ are processed by only checking voxels belonging to cell faces [S2].
- **Complex Edge Criterion:** Detects configurations where a cell edge intersects $\partial V$ more than once, which can induce non-manifold surfaces [S2].
- **Usage:** Used in octree construction to decide if a cell must be subdivided based on topological and geometrical characteristics of $\partial V$ [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of polygons, as each polygon is checked against a constant number of boundary planes (6).
- **Memory Complexity:** $O(1)$ auxiliary space per polygon, as the check is stateless.
- **Strengths:** Allows arbitrary internal geometry to remain untouched while efficiently applying neighbor occlusion only where spatially relevant [S1, unofficial]. Simplifies chunk transition logic by treating chunk borders as having normal voxel neighbors [S1, unofficial].
- **Limitations:** Requires precise knowledge of cell boundaries and tolerance $\epsilon$ to avoid misclassification. Does not handle complex internal topology detection (handled by variants like Complex Face Criterion) [S2].
- **When to use:** When using neighbor-based culling in blocky meshing to avoid culling internal triangles of complex shapes [S1, unofficial].
- **When not to use:** When using methods that do not rely on neighbor occlusion or when internal geometry culling is desired [S1, unofficial].

## Background
Boundary Geometry Classification was introduced in the context of face culling for voxel cubes and partial shapes [S1, unofficial]. It addresses the need to separate boundary geometry from internal geometry in complex blocks to optimize neighbor-based culling [S1, unofficial]. The Complex Face and Edge Criterion variant was developed for adaptive surface extraction from anisotropic volumetric data using generalized octrees [S2].

## Sources
[S1] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
[S2] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
