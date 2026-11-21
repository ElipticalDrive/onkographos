# Signed Box Distance Function

Also called: Box SDF, Box SDF v2, Rectangular (Axis-Aligned Box) SDF, SDF Box, SDF Cube, Signed Box, sdBox

## Overview
The Signed Box Distance Function computes the exact Euclidean signed distance from a query point to the surface of an axis-aligned box centered at the origin [S9, unofficial]. It distinguishes interior from exterior space by returning negative values inside the box and positive values outside [S9, unofficial]. It sets itself apart from the [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md) by providing signed output and from the [Plane Signed Distance Function](1102-plane-signed-distance-function.md) by handling bounded 3D volumes with sharp corners rather than infinite half-spaces [S7, unofficial]. Unlike the cheaper half-space intersection approximation [S14, unofficial], this method correctly calculates the distance to edges and corners [S13, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `p`, a 3D point in space (vector).
- Input: `b`, the half-extents of the box (vector), defining the box as $[-b_x, b_x] \times [-b_y, b_y] \times [-b_z, b_z]$ [S12, unofficial].
- Output: `d`, a scalar representing the signed distance.

**Data Structures:**
- `p`: `vec3` (x, y, z).
- `b`: `vec3` (x, y, z).

**Steps:**
1. Compute the difference vector `di` by subtracting the half-extents from the absolute coordinates of the point: $di = |p| - b$. This exploits symmetry to handle all octants uniformly [S2, unofficial].
2. Calculate the "inside distance" component `res` as the maximum of the components of `di`: $res = \max(di_x, \max(di_y, di_z))$ [S12, unofficial].
3. Check if the point is outside. If $res > 0$, the point is outside the box [S12, unofficial]. In this case, calculate the Euclidean distance to the closest face, edge, or corner by taking the length of the positive components of `di`: $res = \| \max(di, 0) \|$ [S12, unofficial].
4. If the point is inside ($res \le 0$), `res` remains the negative distance to the closest face [S9, unofficial].
5. Return `res`.

**Formula:**
$$ d(p) = \begin{cases} \max(di_x, \max(di_y, di_z)) & \text{if } \max(di_x, \max(di_y, di_z)) \le 0 \\ \| \max(di, 0) \| & \text{if } \max(di_x, \max(di_y, di_z)) > 0 \end{cases} $$
where $di = |p| - b$.

Where:
- $|p|$ is the component-wise absolute value of $p$ [S12, unofficial].
- $\max(v)$ is the maximum component of vector $v$ [S12, unofficial].
- $\max(v, 0)$ is the component-wise maximum of $v$ and the zero vector [S12, unofficial].
- $\| \cdot \|$ is the Euclidean norm (L2 norm) [S12, unofficial].

**Parameters:**
- `b`: Half-extents. For a 1x1x1 box centered at the origin, $b = (0.5, 0.5, 0.5)$ [S12, unofficial].

**Edge Cases and Pitfalls:**
- **Underestimation (Version 1):** An earlier version (Algorithm 3.5 in [S12, unofficial]) returned $\min(\max(|p|-b), \| \max(|p|-b, 0) \|)$. This underestimates the distance in corner regions, leading to smaller ray steps, slower convergence, and incorrect ambient occlusion (black dots/aliasing) [S12, unofficial][S13, unofficial].
- **Overestimation:** Overestimating the distance can cause the ray to pass through the surface or inaccurately estimate the surface point position, negatively impacting ambient occlusion [S12, unofficial].
- **Approximation:** A cheaper approximation $d = \max(|p-c| - s)$ (where $s$ is side length) is not a true Euclidean distance for points outside corners and returns distance to the nearest face/edge rather than the vertex. It is sufficient for raymarching as a safe lower bound but lacks precision [S14, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def signed_box_distance(p: Vec3, b: Vec3) -> float:
    # Step 1: Compute difference vector using symmetry
    di_x = abs(p.x) - b.x
    di_y = abs(p.y) - b.y
    di_z = abs(p.z) - b.z
    
    # Step 2: Calculate inside distance (max component)
    res = max(di_x, max(di_y, di_z))
    
    # Step 3: If outside (res > 0), calculate Euclidean distance to surface
    if res > 0:
        pos_x = max(di_x, 0.0)
        pos_y = max(di_y, 0.0)
        pos_z = max(di_z, 0.0)
        res = sqrt(pos_x**2 + pos_y**2 + pos_z**2)
        
    # Step 4: If inside (res <= 0), res is already the negative distance
    return res
```

## Subtopics

### [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md)
Computes the Euclidean distance to the box surface, returning only non-negative values. Use when interior/exterior distinction is not needed.

### [Box with Edge Rounding SDF](4004-box-with-edge-rounding-sdf.md)
Computes distance to a box with rounded edges/corners. Use for smooth geometric transitions.

### [Rectangle Signed Distance Function](4031-rectangle-signed-distance-function.md)
The 2D analog of the Signed Box Distance Function. Use for planar rectangles.

### [Oriented Box2D SDF](4370-oriented-box2d-sdf.md)
Computes distance to a 2D oriented box (segment-based). Use for 2D shapes with specific orientation.

## Operating characteristics
- **Time Complexity:** O(1). The computation involves a fixed number of arithmetic operations, absolute values, and a square root [G].
- **Memory Complexity:** O(1). No auxiliary data structures are required [G].
- **Performance:** Extremely fast, suitable for real-time ray marching. The exact version (Version 2) is slightly more expensive than the approximation due to the square root, but necessary for correct AO [G].
- **Strengths:** Exact Euclidean distance; handles inside/outside correctly; robust for sphere tracing [G].
- **Limitations:** The square root operation is more expensive than simple max/min. The approximation is faster but inaccurate for AO [S12, unofficial][S13, unofficial].
- **When to use:** Standard primitive for ray marching axis-aligned boxes [G].
- **When not to use:** When performance is critical and AO is not required (use approximation); when rounded edges are needed (use Box with Edge Rounding) [G].

## Background
The Signed Box Distance Function is a fundamental primitive in Signed Distance Field (SDF) ray marching [S8, unofficial]. The "Version 1" approximation was popularized by Inigo Quilez [S2, unofficial][S12, unofficial]. The "Version 2" exact formulation was introduced to fix corner artifacts and incorrect ambient occlusion calculations, as detailed in [S12, unofficial] and [S13, unofficial].

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
[S15] holger-kibur/sdf, https://github.com/holger-kibur/sdf, fetched 2026-10-27 (unofficial)
