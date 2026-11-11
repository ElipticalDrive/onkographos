# Boundary Geometry Classification


## Overview
Boundary Geometry Classification is a method used in blocky meshing to distinguish geometry located on the voxel cell boundary planes from geometry located in the interior of a complex block [S1, unofficial]. It solves the problem where neighbor-based face culling must be restricted to external surfaces to avoid incorrectly culling internal geometry [S1, unofficial]. It provides the geometric predicate to determine which faces are eligible for neighbor occlusion tests [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A polygon defined by a list of 3D vertices (`vertices`), the axis of the face being tested (`axis`), the coordinate of the cell boundary plane (`plane`), and a small tolerance value (`eps`) [S1, unofficial].
- Output: A boolean indicating whether the polygon lies on the boundary [S1, unofficial].

**Data Structures:**
- `vertices`: An array of 3D vectors (`Vec3`), where each vector contains `(x, y, z)` coordinates [S1, unofficial].
- `axis`: An integer index (0, 1, or 2) corresponding to the x, y, or z axis perpendicular to the face [S1, unofficial].
- `plane`: A floating-point scalar representing the coordinate value of the voxel cell boundary [S1, unofficial].
- `eps`: A floating-point tolerance value for numerical stability [S1, unofficial].

**Steps:**
1. Check if all vertices in the polygon lie on the same cell boundary plane within a small tolerance `eps` [S1, unofficial].
2. If the condition holds, classify the polygon as a boundary polygon; otherwise, classify it as internal [S1, unofficial].

**Formula:**
A polygon with vertices $V = \{v_0, v_1, \dots, v_n\}$ is on the boundary if:
$$ \forall v \in V, \quad |v_{\text{axis}} - \text{plane}| \le \text{eps} $$
where $v_{\text{axis}}$ is the component of vertex $v$ along the specified axis [S1, unofficial].

**Parameters:**
- `eps`: A small tolerance value used to determine if a vertex lies on the boundary plane [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point inaccuracies in vertex generation are handled by the tolerance `eps` [S1, unofficial].
- **Internal Geometry:** Geometry strictly inside the voxel (not on any of the 6 boundary planes) will fail this check for all axes and should be excluded from neighbor culling tests to prevent incorrect occlusion [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lies_on_boundary(vertices: list, axis: int, plane: float, eps: float) -> bool:
    """Classifies a polygon as a boundary polygon if all its vertices lie on the same cell boundary plane."""
    return all(abs(v[axis] - plane) <= eps for v in vertices)
```

## Subtopics

### [Complex Face and Edge Criterion](6790-complex-face-and-edge-criterion.md)
A subdivision control mechanism used during octree construction to detect surface pieces smaller than the current voxel resolution. It ensures that all cells intersected by the surface $\partial V$ are processed by identifying "complex" configurations, specifically homogeneous faces containing disconnected surface components and edges intersecting the surface multiple times. Choose it when standard inhomogeneity checks fail to capture topological complexity within homogeneous regions, preventing the loss of fine geometric details.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Simple and fast for separating internal from external geometry, allowing the mesher to keep arbitrary internal geometry while applying neighbor occlusion only where it makes spatial sense [S1, unofficial].
- **Limitations:** Relies on a tolerance value `eps` which must be chosen carefully [S1, unofficial].
- **When to use:** When neighbor-based culling must be restricted to external surfaces to avoid incorrectly culling internal geometry [S1, unofficial].
- **When not to use:** Not given in the sources.

## Background
This method is part of the face culling pipeline for voxel cubes and partial shapes, specifically designed to support neighbor-based occlusion culling in blocky meshing [S1, unofficial]. It ensures that internal triangles inside a complex block are not tested against neighboring blocks, which would be incorrect [S1, unofficial].

## Sources
[S1] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
