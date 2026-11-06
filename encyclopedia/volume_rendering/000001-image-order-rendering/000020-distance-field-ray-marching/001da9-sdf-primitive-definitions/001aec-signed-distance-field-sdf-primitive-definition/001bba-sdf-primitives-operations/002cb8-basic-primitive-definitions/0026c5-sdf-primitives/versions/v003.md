# SDF Primitives

Also called: Signed Distance Field (SDF) Primitives, Signed Distance Function Primitives, Signed Distance Primitives, SDF Primitive Functions, Basic Signed Distance Functions, Basic SDFs, Analytical SDFs

## Overview
SDF Primitives are analytical functions that compute the exact Euclidean signed distance from a query point in 3D space to the surface of a geometric shape [S1, S5, S7, unofficial]. They serve as the fundamental building blocks for distance-field ray marching, replacing explicit mesh geometry with implicit mathematical definitions [S5, S8, unofficial]. Unlike polygonal primitives, SDF primitives provide a continuous scalar field where the sign indicates interior ($d < 0$) or exterior ($d > 0$) status, enabling safe, large step sizes along rays during rendering [S1, S9, unofficial]. They are distinguished from their siblings, such as [Polyhedral Primitives](2529-polyhedral-primitives.md) or [Cone Signed Distance Function](2594-cone-signed-distance-function.md), by their reliance on closed-form algebraic solutions for standard shapes like spheres, boxes, and tori, rather than iterative or piecewise linear approximations [S1, S10, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $p \in \mathbb{R}^3$ and primitive-specific parameters (e.g., radius, half-extents).
*   **Output:** A scalar signed distance $d \in \mathbb{R}$. $d < 0$ indicates the point is inside the primitive, $d > 0$ indicates it is outside, and $d = 0$ lies on the surface [S1, S9, unofficial].

**Data Structures:**
Primitives are defined in a canonical orientation centered at the origin [SDF Primitives](1731-sdf-primitives.md). To place a primitive at a different location, orientation, or scale, the query point $p$ is transformed before evaluation:
1.  **Translate:** Replace $p$ with $p - c$, where $c$ is the center [SDF Primitives](1731-sdf-primitives.md).
2.  **Rotate:** Multiply $p$ by the inverse rotation matrix $R^{-1}$: $p' = R^{-1} p$ [SDF Primitives](1731-sdf-primitives.md).
3.  **Scale:** Uniformly scale by $s$ using $d(p/s) \cdot s$ [SDF Primitives](1731-sdf-primitives.md).

**Formulas and Steps:**
The following implementations assume the primitive is centered at the origin and axis-aligned where applicable.

1.  **Sphere:**
    *   Parameters: Radius $r$.
    *   Formula: $d(p) = \|p\| - r$ [S1, S2, S5, S9, unofficial].
    *   Implementation: `return length(p) - r;` [S5, unofficial].

2.  **Axis-Aligned Box (Cube):**
    *   Parameters: Half-extents vector $b = (b_x, b_y, b_z)$ [S5, S10, unofficial].
    *   Steps:
        1.  Compute $q = |p| - b$ [S10, unofficial].
        2.  Compute outside distance: $d_{out} = \| \max(q, 0) \|$ [S10, unofficial].
        3.  Compute inside distance: $d_{in} = \min(\max(q_x, \max(q_y, q_z)), 0)$ [S10, unofficial].
        4.  Result: $d(p) = d_{out} + d_{in}$ [S10, unofficial].
    *   Formula: $d(p, b) = \|\max(q, 0)\| + \min(\max(q_x, \max(q_y, q_z)), 0)$ where $q = |p| - b$ [S1, S5, unofficial].

3.  **Torus:**
    *   Parameters: Major radius $R$ (hole radius), minor radius $r$ (tube radius) [S2, S8, unofficial].
    *   Steps:
        1.  Compute $q = (\sqrt{p_x^2 + p_z^2} - R, p_y)$ [S1, S8, unofficial].
        2.  Result: $d(p) = \|q\| - r$ [S1, S8, unofficial].
    *   Implementation: `vec2 q = vec2(length(p.xz) - R, p.y); return length(q) - r;` [S10, unofficial].

4.  **Infinite Plane:**
    *   Parameters: Normalized normal vector $n$, offset $d$ (distance from origin) [S3, S10, unofficial].
    *   Formula: $d(p) = \text{dot}(p, n) + d$ [S3, S10, unofficial].
    *   Note: $n$ must be normalized [S3, unofficial].

5.  **Cylinder (Capped):**
    *   Parameters: Radius $r$, half-height $h$ [S10, unofficial].
    *   Steps:
        1.  Compute $d = (\text{abs}(\sqrt{p_x^2 + p_z^2}) - r, \text{abs}(p_y) - h)$ [S10, unofficial].
        2.  Result: $d(p) = \min(\max(d_x, d_y), 0) + \|\max(d, 0)\|$ [S10, unofficial].

6.  **Capsule:**
    *   Parameters: Endpoints $a, b$, radius $r$ [S10, unofficial].
    *   Steps:
        1.  Compute vector $ab = b - a$ and $ap = p - a$ [S10, unofficial].
        2.  Compute projection parameter $t = \text{clamp}(\frac{\text{dot}(ap, ab)}{\text{dot}(ab, ab)}, 0, 1)$ [S10, unofficial].
        3.  Result: $d(p) = \|ap - ab \cdot t\| - r$ [S10, unofficial].

7.  **Octahedron:**
    *   Parameters: Size parameter $s$ [S5, unofficial].
    *   Formula: $d(p) = \frac{|p_x| + |p_y| + |p_z| - s}{\sqrt{3}}$ [S5, unofficial].
    *   Implementation: `p = abs(p); return (p.x + p.y + p.z - s) * 0.57735027;` [S5, unofficial].

8.  **L-Norm Shapes (Generalized):**
    *   Helper: $len(v, n) = (|v_x|^n + |v_y|^n + |v_z|^n)^{1/n}$ [S7, unofficial].
    *   Used to approximate shapes with sharp corners (e.g., N-norm Torus) [S7, unofficial].

**Edge Cases and Pitfalls:**
*   **Normalization:** The plane normal $n$ must be normalized for the distance calculation to be correct [S3, unofficial].
*   **Precision:** Standard floating-point precision is sufficient for most primitives, but extreme scales may require careful handling of the offset in plane equations [S3, unofficial].
*   **Degenerate Input:** For capsules, if endpoints $a$ and $b$ are identical, the denominator $\text{dot}(ab, ab)$ becomes zero; a guard clause should handle this case (e.g., treating it as a sphere) [G].

## Subtopics

### [Signed Box Distance Function](1092-signed-box-distance-function.md)
Computes the exact Euclidean signed distance to an axis-aligned box. Preferred over naive half-space intersections because it provides true Euclidean distance to corners and edges, ensuring correct step sizes in sphere tracing.

### [Plane Signed Distance Function](1102-plane-signed-distance-function.md)
Computes distance to an infinite plane. Distinguished by computational simplicity (single dot product) and unbounded nature, unlike bounded siblings like Sphere or Box.

### [Sphere Signed Distance Function](1237-sphere-signed-distance-function.md)
Computes distance to a sphere. Closed-form and rotationally invariant, unlike the [Ellipsoid SDF](2397-ellipsoid-signed-distance-function.md) which requires solving higher-degree polynomials.

### [Ellipsoid Signed Distance Function](2397-ellipsoid-signed-distance-function.md)
Computes distance to an ellipsoid. Provides fast approximations (V1) or iterative-free approximations (V2).

### [Capsule Signed Distance Function](2439-capsule-signed-distance-function.md)
Computes distance to a capsule (finite segment with radius). Distinguished from Sphere by its linear axis and from Box by its smooth, rounded geometry.

### [Arc Signed Distance Function](3922-arc-signed-distance-function.md)
Computes distance to a 3D arc (tube along a circular arc). Distinguished from Capsule by its curved trajectory and from Torus by its limited angular aperture [G].

### Rhombic Triacontahedron SDF
A Signed Distance Function for a Rhombic Triacontahedron crystal shape, constructed by folding 3D space using specific magic numbers and directions multiple times [S11, unofficial].
*   **Magic Numbers:** $c = \cos(\pi/5)$, $s = \sqrt{0.75 - c^2}$ [S11, unofficial].
*   **Magic Direction:** $n = (-0.5, -c, s)$ [S11, unofficial].
*   **Steps:**
1.  $p = |p|$ [S11, unofficial].
2.  Fold along $n$: $p \leftarrow p - 2 \cdot \min(0, \text{dot}(p, n)) \cdot n$ [S11, unofficial].
3.  $p_{xy} = |p_{xy}|$ [S11, unofficial].
4.  Fold along $n$ again [S11, unofficial].
5.  $p_{xy} = |p_{xy}|$ [S11, unofficial].
6.  Fold along $n$ one more time [S11, unofficial].
7.  Return $p_z - 1$ [S11, unofficial].

### Death Star SDF Primitive
A composite primitive constructed using standard SDF operations (union, difference) with specific radii ($r_a=5.0, r_b=2.0$) and depth ($d=6.0$), often animated with rotation [S12, unofficial].

### Spherical Signed Distance Field
A structure used to represent visibility by encoding the signed distance to the boundary of the occluded region in a sphere, distinct from geometric primitives [S13, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per primitive evaluation, as all basic primitives use closed-form algebraic expressions [S1, S10, unofficial].
*   **Memory Complexity:** $O(1)$, requiring only the storage of primitive parameters (e.g., radius, center) [S4, S5, unofficial].
*   **Performance:** Extremely efficient for simple scenes or when combined with CSG operations, as they avoid mesh traversal or rasterization overhead [S9, S10, unofficial].
*   **Strengths:** Exact distance estimation allows for large, safe step sizes in ray marching (sphere tracing), reducing the number of iterations required for convergence [S1, S9, unofficial]. Smooth gradients facilitate anti-aliasing and ambient occlusion calculations [S12, S13, S14, unofficial].
*   **Limitations:** Complex scenes require many primitive evaluations and CSG operations, which can become computationally expensive compared to rasterized meshes for highly detailed geometry [S8, unofficial]. Not suitable for non-convex or highly irregular shapes without decomposition into simpler primitives [S10, unofficial].
*   **When to Use:** Ideal for procedural geometry, soft-body simulation, and real-time ray marching where simplicity and exact distance bounds are prioritized [S1, S5, unofficial].
*   **When Not to Use:** Unsuitable for complex, organic, or highly detailed meshes that cannot be easily approximated by a combination of basic primitives [S8, unofficial].

## Background
SDF primitives were popularized in computer graphics by Inigo Quilez, who published analytical distance functions for common geometric shapes in his article "Distances" [S8, S10, unofficial]. The technique of using these functions for ray marching, known as "sphere tracing," was introduced by John C. Williams in 1990 [S9, unofficial]. The method has since become a standard tool in creative coding, real-time rendering, and procedural generation due to its simplicity and visual fidelity [S8, S11, unofficial].

## Sources
[S1] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S2] AustinGoodman/non-linear-sphere-tracing, https://github.com/AustinGoodman/non-linear-sphere-tracing, fetched 2026-10-26 (unofficial)
[S3] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
[S4] ChadMcKinney/Dirac-Sea-Engine, https://github.com/ChadMcKinney/Dirac-Sea-Engine, fetched 2026-10-26 (unofficial)
[S5] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S6] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S7] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S8] jk9g13/LaFortuna-SDF-Ray-marching, https://github.com/jk9g13/LaFortuna-SDF-Ray-marching, fetched 2026-10-27 (unofficial)
[S9] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S10] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S11] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S12] agill-dev-0/SDF-ray-marcher, https://github.com/agill-dev-0/SDF-ray-marcher, fetched 2026-10-26 (unofficial)
[S13] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
