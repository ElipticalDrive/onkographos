# Conservative Rasterization via Triangle Expansion

Also called: Hole Repair via Conservative Rasterization, Triangle Expansion Method

## Overview
This method eliminates voxelization holes caused by standard rasterization, which only covers pixels whose centers lie inside a triangle, by expanding the triangle in clip space and culling the expanded area in the fragment shader [S1, unofficial]. It differs from sibling methods like [Conservative Voxelization](3247-conservative-voxelization.md) by implementing the expansion logic manually in the geometry shader using homogeneous plane mathematics, rather than relying on hardware-specific conservative rasterization extensions or post-processing hole-filling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Triangle vertices in clip space ($P_0, P_1, P_2$), each a `vec4` $(x, y, z, w)$ [S1, unofficial].
- Output: Expanded triangle vertices in clip space, which are then rasterized by the hardware [S1, unofficial].

**Data Structures:**
- `vec4 pos[3]`: Array of three clip-space vertices [S1, unofficial].
- `vec3 edgePlanes[3]`: Array of three homogeneous plane normals [S1, unofficial].
- `vec4 aabb`: Axis-Aligned Bounding Box storing min $(x, y)$ and max $(z, w)$ in clip space [S1, unofficial].
- `vec2 pixelDiagonal`: Half-pixel diagonal distance in clip space [S1, unofficial].

**Numbered Steps:**

1.  **Calculate Clip-Space AABB:** Compute the 2D bounding box of the triangle in clip space $(x, y)$, ignoring $z$. Expand this box by the half-pixel diagonal distance to ensure coverage of edge pixels [S1, unofficial].
    $$ AABB_{expanded} = AABB_{original} \pm pixelDiagonal $$
    where $pixelDiagonal$ is half the diagonal length of a pixel in clip space [S1, unofficial].

2.  **Compute Edge Planes:** For each edge of the triangle, calculate a homogeneous plane normal in 2D homogeneous space (using $x, y, w$ components). The plane normal $N$ for an edge from $P_i$ to $P_j$ is the cross product of the edge vector and the vertex position [S1, unofficial]:
    $$ N = (P_i.xyw - P_j.xyw) \times P_j.xyw $$
    This defines a plane passing through the origin in homogeneous space, representing the line segment in 2D [S1, unofficial].

3.  **Expand Edge Planes:** Offset each plane normal's $z$ component (the constant term in the homogeneous line equation) outward [S1, unofficial]. The offset distance is determined by the projection of the half-pixel size onto the plane's normal [S1, unofficial]:
    $$ N.z \leftarrow N.z - \text{dot}(pixelDiagonal, |N.xy|) $$
    This moves the edge outward, effectively expanding the triangle boundary [S1, unofficial].

4.  **Compute New Vertices:** Calculate the intersection lines of the expanded planes [S1, unofficial]. The intersection line vector of two planes is the cross product of their normals [G]. Find the intersection of these lines with the original triangle plane to determine the new vertex positions [S1, unofficial].

5.  **Fragment Shader Culling:** In the fragment shader, calculate the barycentric coordinates or distance to the original triangle edges [S1, unofficial]. Discard fragments that lie within the expanded region but outside the original triangle to prevent over-rasterization [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def conservative_rasterization_expand(pos: list[tuple[float, float, float, float]], pixel_diagonal: tuple[float, float]) -> list[tuple[float, float, float, float]]:
    """
    Expands a triangle in clip space to ensure conservative rasterization coverage.
    pos: List of 3 vec4 clip-space vertices (x, y, z, w).
    pixel_diagonal: Tuple (dx, dy) representing half-pixel diagonal in clip space.
    Returns: List of 3 expanded vec4 vertices.
    """
    # Step 1: Calculate AABB and expand
    xs = [v[0] for v in pos]
    ys = [v[1] for v in pos]
    aabb_min = (min(xs), min(ys))
    aabb_max = (max(xs), max(ys))
    aabb_min = (aabb_min[0] - pixel_diagonal[0], aabb_min[1] - pixel_diagonal[1])
    aabb_max = (aabb_max[0] + pixel_diagonal[0], aabb_max[1] + pixel_diagonal[1])

    # Helper functions for 3D vector operations
    def cross3d(a: tuple, b: tuple) -> tuple:
        return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])
    def dot3d(a: tuple, b: tuple) -> float:
        return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

    # Step 2: Compute homogeneous edge planes
    # Edge 0: P2 -> P0, Edge 1: P0 -> P1, Edge 2: P1 -> P2
    edge_planes = []
    edge_planes.append(cross3d((pos[0][0]-pos[2][0], pos[0][1]-pos[2][1], pos[0][3]-pos[2][3]), (pos[2][0], pos[2][1], pos[2][3])))
    edge_planes.append(cross3d((pos[1][0]-pos[0][0], pos[1][1]-pos[0][1], pos[1][3]-pos[0][3]), (pos[0][0], pos[0][1], pos[0][3])))
    edge_planes.append(cross3d((pos[2][0]-pos[1][0], pos[2][1]-pos[1][1], pos[2][3]-pos[1][3]), (pos[1][0], pos[1][1], pos[1][3])))

    # Step 3: Expand edge planes
    for i in range(3):
        # Compute dot(pixel_diagonal, abs(N.xy)) safely for 2D inputs
        offset = pixel_diagonal[0] * abs(edge_planes[i][0]) + pixel_diagonal[1] * abs(edge_planes[i][1])
        edge_planes[i] = (edge_planes[i][0], edge_planes[i][1], edge_planes[i][2] - offset)

    # Step 4: Compute new vertices
    # The intersection of two homogeneous lines L1 and L2 is L1 x L2
    new_pos = []
    for i in range(3):
        # Vertex i is the intersection of edge plane i and edge plane (i+1)%3
        intersect = cross3d(edge_planes[i], edge_planes[(i + 1) % 3])
        # Convert homogeneous point (x, y, w) to clip space
        if intersect[2] != 0:
            x = intersect[0] / intersect[2]
            y = intersect[1] / intersect[2]
            # z is preserved from the original vertex as the expansion is purely 2D in xyw [G]
            z = pos[i][2]
            w = 1.0
            new_pos.append((x, y, z, w))
        else:
            new_pos.append(pos[i])
    return new_pos
```

**Parameters:**
- `pixelDiagonal`: Half the diagonal length of a pixel in clip space [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision issues may arise if the triangle is very small or very large in clip space [G].
- Degenerate triangles (zero area) may produce undefined plane normals [G].
- The method assumes orthographic projection for the simplified 2D homogeneous line interpretation; perspective projection requires careful handling of the $w$ component [S1, unofficial].

## Subtopics

### [Conservative Voxelization](3247-conservative-voxelization.md)
Uses hardware extensions for conservative rasterization. Choose when hardware support is available and manual geometry shader implementation is undesirable.

### [3D Grid Rasterization](903-3d-grid-rasterization.md)
Rasterizes directly into a 3D grid. Choose when avoiding 2D projection artifacts is critical.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Strengths:** Effectively eliminates holes caused by standard rasterization without requiring post-processing [S1, unofficial].
- **Limitations:** Requires manual implementation in the geometry shader [S1, unofficial].
- **When to use:** When using standard rasterization pipelines and encountering voxelization holes, especially in multi-view projection scenarios [S1, unofficial].

## Background
The method was described by YangWC in a blog post on GPU-based 3D voxelization, addressing the issue of holes in voxelized models due to standard rasterization [S1, unofficial].

## Sources
[S1] 体素化Voxelization：基于GPU的三维体素化 - YangWC's Blog, https://yangwc.com/2019/06/11/Voxelization/, fetched 2026-10-30 (unofficial)
