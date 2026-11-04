# Linear Intersection Approximation

Also called: Midpoint Intersection

## Overview
Linear Intersection Approximation (also known as Midpoint Intersection) is a method for estimating the intersection point of an isosurface with a grid edge. It solves the problem of determining where a scalar field crosses a threshold value by calculating the geometric midpoint of the edge, rather than performing linear interpolation based on the scalar values at the endpoints. It is distinguished from its parent, [Linear Edge Intersection](868-linear-edge-intersection.md), by ignoring the scalar field values entirely, and from its sibling, [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md), by using a single fixed step rather than recursive refinement.

## How it works
**Inputs:**
- Two edge endpoints in 3D space: $P_1 = (x_1, y_1, z_1)$ and $P_2 = (x_2, y_2, z_2)$.
- The edge index or configuration to determine the specific edge of a voxel cube (if starting from a voxel corner).

**Outputs:**
- The intersection point $P_{mid} = (x_{mid}, y_{mid}, z_{mid})$, which is the geometric midpoint of the edge.

**Data Structures:**
- Point coordinates are typically stored as floating-point values (e.g., `float` or `double`).
- Edge indices are integers mapping to specific edges of a voxel (0–12 for a cube).

**Steps:**
1. **Initialize Endpoints:** Define the coordinates of the two endpoints of the edge. If the edge is part of a voxel grid, these are often derived from the voxel corner $(x, y, z)$ and the edge number.
2. **Adjust Endpoints:** Based on the edge number, adjust the coordinates of $P_1$ and $P_2$ to represent the specific edge of the unit cube. For example:
   - Edge 0: From $(x, y, z)$ to $(x, y+1, z)$.
   - Edge 1: From $(x, y+1, z)$ to $(x+1, y+1, z)$.
   - (Other edges follow similar patterns, incrementing $x, y,$ or $z$ by 1 for the second endpoint).
3. **Calculate Midpoint:** Compute the midpoint of the edge using the arithmetic mean of the endpoints.
   $$ P_{mid} = \frac{P_1 + P_2}{2} $$
   $$ x_{mid} = \frac{x_1 + x_2}{2}, \quad y_{mid} = \frac{y_1 + y_2}{2}, \quad z_{mid} = \frac{z_1 + z_2}{2} $$
4. **Return Result:** Return $P_{mid}$ as the intersection point.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_intersection_approximation(p1, p2):
    """
    Calculates the midpoint of an edge defined by two 3D points.
    
    Args:
        p1: Tuple or array of 3 floats (x1, y1, z1)
        p2: Tuple or array of 3 floats (x2, y2, z2)
        
    Returns:
        Tuple or array of 3 floats representing the midpoint (x_mid, y_mid, z_mid)
    """
    x_mid = (p1[0] + p2[0]) / 2.0
    y_mid = (p1[1] + p2[1]) / 2.0
    z_mid = (p1[2] + p2[2]) / 2.0
    return (x_mid, y_mid, z_mid)

def get_edge_endpoints(voxel_origin, edge_index):
    """
    Returns the endpoints of a specific edge of a unit voxel cube.
    
    Args:
        voxel_origin: Tuple (x, y, z) of the voxel's minimum corner.
        edge_index: Integer 0-12 representing the edge.
        
    Returns:
        Tuple of two points (p1, p2)
    """
    x, y, z = voxel_origin
    # Define all 12 edges of a unit cube relative to the origin
    edges = [
        ((x, y, z), (x, y + 1, z)),  # Edge 0
        ((x, y + 1, z), (x + 1, y + 1, z)),  # Edge 1
        ((x + 1, y + 1, z), (x + 1, y, z)),  # Edge 2
        ((x + 1, y, z), (x, y, z)),  # Edge 3
        ((x, y, z + 1), (x, y + 1, z + 1)),  # Edge 4
        ((x, y + 1, z + 1), (x + 1, y + 1, z + 1)),  # Edge 5
        ((x + 1, y + 1, z + 1), (x + 1, y, z + 1)),  # Edge 6
        ((x + 1, y, z + 1), (x, y, z + 1)),  # Edge 7
        ((x, y, z), (x, y, z + 1)),  # Edge 8
        ((x, y + 1, z), (x, y + 1, z + 1)),  # Edge 9
        ((x + 1, y + 1, z), (x + 1, y + 1, z + 1)),  # Edge 10
        ((x + 1, y, z), (x + 1, y, z + 1))  # Edge 11
    ]
    return edges[edge_index]
```

**Parameters:**
- None. The method is deterministic and requires no tuning parameters.

**Edge Cases and Pitfalls:**
- **Precision:** Standard floating-point arithmetic is sufficient. No special handling is needed for degenerate edges (zero-length) as the midpoint will simply be the endpoint.
- **Boundary Conditions:** The method assumes the edge is a straight line segment. It does not account for curved edges or non-linear scalar fields.
- **Accuracy:** This method is an approximation. It is only accurate if the scalar field varies linearly and the threshold is exactly at the midpoint, or if the scalar values at the endpoints are symmetric around the threshold. In general, it can introduce significant geometric errors compared to linear interpolation.

## Subtopics

### Recursive Bisection Interpolation
A variant that refines the intersection point by recursively bisecting the edge until the midpoint crosses the isovalue, reducing geometric holes between different LODs [S2, unofficial]. It uses the linear interpolation formula $p = a + \frac{iso - value_a}{value_b - value_a}(b - a)$ for the final result if the bisection does not find a crossing, or returns the midpoint if a crossing is detected [S2, unofficial]. The condition for bisection is $(value_a - iso)(value_c - iso) < 0$ [S2, unofficial]. It is controlled by a boolean flag `surface_shift` (0 or 1) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per edge. It involves only basic arithmetic operations (addition and division).
- **Memory Complexity:** $O(1)$. No additional data structures are required.
- **Performance:** Extremely fast due to the lack of scalar field lookups and complex calculations.
- **Strengths:** Simplicity and speed. Useful for quick approximations or when scalar field data is unavailable.
- **Limitations:** Low accuracy. It ignores the scalar field values, leading to incorrect intersection points unless the field is perfectly symmetric around the midpoint. It is not suitable for high-quality isosurface extraction.
- **When to use:** When performance is critical and accuracy is secondary, or as a fallback when scalar values are not available.
- **When not to use:** When high-fidelity isosurfaces are required, or when the scalar field varies significantly across the edge.

## Background
The method is described in the context of marching cubes implementations, specifically in the `ilastik/marching_cubes` repository [S1, unofficial]. It is presented as a simple alternative to linear interpolation for edge intersections. The term "Midpoint Intersection" is used interchangeably. The method is part of a broader set of techniques for isosurface extraction, where accurate edge intersection is crucial for generating correct triangle meshes.

## Sources
[S1] ilastik/marching_cubes, https://github.com/ilastik/marching_cubes, fetched 2026-10-26 (unofficial)
[S2] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
