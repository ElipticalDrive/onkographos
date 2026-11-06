# Signed Box Distance Function

Also called: Box SDF, Box SDF v2, Rectangular (Axis-Aligned Box) SDF, SDF Box, SDF Cube, Signed Box, sdBox

## Overview
The Signed Box Distance Function computes the exact Euclidean signed distance from a query point to the surface of an axis-aligned box [S5, S9, S11, unofficial]. It distinguishes interior from exterior space by returning negative values inside the box and positive values outside [S9, unofficial]. It is preferred over the naive intersection of six half-spaces because it provides the true Euclidean distance to corners and edges, ensuring correct step sizes in sphere tracing and accurate ambient occlusion [S12, S13, S14, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `p`, a 3D point in space (vector).
- Input: `b`, the half-extents of the box (vector), defining the box as centered at the origin with dimensions $2b_x \times 2b_y \times 2b_z$. If the box is not centered at the origin, `p` is transformed to local coordinates: $p_{local} = p - c$, where $c$ is the center [S6, S7, S9, unofficial].
- Output: A scalar float representing the signed distance.

**Data Structures:**
- `p`: `vec3` (x, y, z).
- `b`: `vec3` (x, y, z).
- `c`: `vec3` (x, y, z), optional center offset.

**Algorithm:**
The method exploits symmetry by taking the absolute value of the point coordinates, effectively folding the space into the first octant [S2, S9, unofficial]. It then calculates the distance to the box boundaries in this folded space.

1.  **Local Coordinate Transformation:** If the box has a center $c$, compute $p' = p - c$. Otherwise, $p' = p$.
2.  **Symmetry Folding:** Compute $q = |p'| - b$. This vector $q$ represents the distance from the point to the box faces, with negative values indicating the point is inside the box along that axis [S9, S11, unofficial].
3.  **Inside Distance:** Compute the maximum component of $q$: $c_{in} = \max(q_x, \max(q_y, q_z))$. This value is negative if the point is strictly inside the box (all components of $q$ are negative) [S9, S11, S12, unofficial].
4.  **Outside Distance:** Compute the Euclidean length of the positive components of $q$: $d_{out} = \| \max(q, 0) \|$. This handles points outside the box, correctly calculating distance to faces, edges, or corners [S9, S11, S12, unofficial].
5.  **Combine:** The final signed distance is $d = c_{in} + d_{out}$ [S9, S11, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def signed_box_distance(p: Vec3, b: Vec3, c: Vec3 = None) -> float:
    """
    Computes the signed distance to an axis-aligned box.
    
    Args:
        p: Query point (vec3).
        b: Half-extents of the box (vec3).
        c: Center of the box (vec3). Default is (0,0,0).
        
    Returns:
        Signed distance (float). Negative inside, positive outside.
    """
    # 1. Transform to local coordinates if center is provided
    if c is not None:
        p_local = p - c
    else:
        p_local = p
        
    # 2. Symmetry folding: absolute value minus half-extents
    # q.x = |p.x| - b.x, etc.
    q = abs(p_local) - b
    
    # 3. Calculate inside distance (max component)
    # This is negative if the point is inside the box
    inside_dist = max(q.x, max(q.y, q.z))
    
    # 4. Calculate outside distance (L2 norm of positive components)
    # Clamps negative components to 0, then takes length
    outside_dist = length(max(q, 0.0))
    
    # 5. Combine
    return inside_dist + outside_dist
```

**Parameters:**
- `b` (half-extents): Typical values depend on scene scale. For a unit box centered at origin, $b = (0.5, 0.5, 0.5)$ [S11, S12, unofficial].
- `c` (center): Default $(0, 0, 0)$ [S9, unofficial].

**Edge Cases and Pitfalls:**
- **Underestimation (Version 1):** An earlier approximation used $d = \min(c_{in}, \| \max(q, 0) \|)$ [S12, S13, unofficial]. This underestimates the distance in corner regions, leading to smaller ray steps, slower convergence, and incorrect ambient occlusion (aliasing/black dots) [S12, S13, unofficial]. The correct formula uses addition ($c_{in} + d_{out}$) [S9, S11, unofficial].
- **Overestimation:** Overestimating distance can cause the ray to pass through the surface or inaccurately estimate the surface point, negatively impacting ambient occlusion [S12, unofficial].
- **Cheap Approximation:** A cheaper, less precise approximation uses $d = \max(|p-c| - s)$ where $s$ is the full side length [S14, unofficial]. This is not a true Euclidean distance for points outside corners and should only be used if performance is critical and visual artifacts are acceptable [S14, unofficial].

## Subtopics

### [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md)
The Unsigned Box Distance Function computes the Euclidean distance from a query point to the surface of an axis-aligned box centered at the origin, returning only non-negative values regardless of whether the point lies inside or outside the volume. Unlike its sibling, the Signed Box Distance Function, which distinguishes interior from exterior space, this method treats the box as a solid boundary where internal points are measured by their distance to the nearest face.

### [Box with Edge Rounding SDF](4004-box-with-edge-rounding-sdf.md)
Computes the signed distance to an axis-aligned box with rounded edges and corners, defined by half-extents $b$ and edge radius $e$. It extends the basic box distance by replacing sharp corners with spherical fillets, allowing for smooth geometric transitions in ray marching. It differs from the [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md) by providing signed interior/exterior information and from the [Rectangle Signed Distance Function](4031-rectangle-signed-distance-function.md) by supporting 3D volumes with uniform edge rounding rather than planar rectangles.

### [Rectangle Signed Distance Function](4031-rectangle-signed-distance-function.md)
Computes the exact Euclidean distance from a query point to the boundary of an axis-aligned rectangle centered at the origin. It is the 2D analog of the [Signed Box Distance Function](1092-signed-box-distance-function.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ per query. Involves a few vector subtractions, absolute values, max/min operations, and one square root (for the L2 norm).
- **Memory Complexity:** $O(1)$. No auxiliary data structures required.
- **Performance:** Extremely fast, suitable for real-time ray marching. The square root is the most expensive operation, but it is only computed for the outside distance.
- **Strengths:** Exact Euclidean distance ensures correct sphere tracing steps and accurate ambient occlusion. Simple implementation.
- **Limitations:** Only applies to axis-aligned boxes. For oriented boxes, a transformation is required.
- **When to use:** When exact distance is required for sphere tracing, especially for ambient occlusion or collision detection.
- **When not to use:** When performance is critical and approximate distance is acceptable (use the cheap approximation [S14, unofficial]). When the box is rotated (use oriented box variants or transform coordinates).

## Background
The Signed Box Distance Function is a fundamental primitive in Signed Distance Field (SDF) rendering. The exact formulation using the sum of inside and outside distances is widely attributed to Inigo Quilez [S9, S12, S13, unofficial]. Earlier approximations (Version 1) were found to cause artifacts in ambient occlusion [S12, S13, unofficial]. The function is a standard component in ray marching engines and implicit surface rendering [S2, S14, unofficial].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S2] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S3] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S4] AlerianEmperor/SDF-Sphere-Tracing, https://github.com/AlerianEmperor/SDF-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S5] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S6] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S7] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S8] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S9] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S10] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S11] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S12] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S13] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S14] Volumetric Rendering: Signed Distance Functions - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
