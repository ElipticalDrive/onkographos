# Signed Box Distance Function

Also called: Box SDF, Box SDF v2, Rectangular (Axis-Aligned Box) SDF, SDF Box, SDF Cube, Signed Box, sdBox

## Overview
The Signed Box Distance Function computes the exact Euclidean distance from a query point $p$ to the surface of an axis-aligned box, returning a signed scalar where the sign indicates spatial containment ($d < 0$ inside, $d > 0$ outside) [S9, S12, S13, unofficial]. It solves the problem of defining a robust, differentiable implicit volume primitive for ray marching and implicit modeling, ensuring that step sizes in sphere tracing are safe lower bounds to prevent ray overshooting [S2, S15, unofficial]. Unlike the cheaper approximation that intersects half-spaces, this method correctly calculates the distance to corners and edges, preventing the underestimation artifacts that cause incorrect ambient occlusion and slower convergence [S13, S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $p \in \mathbb{R}^3$; Box definition. Two common parameterizations exist:
    1.  Center $c \in \mathbb{R}^3$ and side lengths $s \in \mathbb{R}^3$ [S6, S7, S15, unofficial].
    2.  Center $c \in \mathbb{R}^3$ and half-extents (bounds) $b \in \mathbb{R}^3$, where $b_i = s_i / 2$ [S9, S12, S13, unofficial].
*   **Output:** Signed distance $d(p) \in \mathbb{R}$.

**Data Structures:**
No complex data structures are required. The function operates on scalar values and 3D vectors. If the box is part of a scene graph, it may be stored as a struct containing two corner vectors $c_1, c_2$ or center/side-length pairs [S4, S5, unofficial].

**Algorithm (Version 2 - Exact):**
This is the standard, correct implementation that handles inside/outside cases separately to ensure Euclidean accuracy at corners [S13, S14, unofficial].

1.  **Shift and Scale:** Compute the vector $q$ representing the distance from the point to the box boundary in each axis. Using half-extents $b$:
    $$ q = |p - c| - b $$
    Where $| \cdot |$ is the component-wise absolute value. [S9, S12, S13, unofficial]

2.  **Inside Distance:** Calculate the maximum component of $q$. If the point is inside, all components of $q$ are negative. The distance to the nearest face is the magnitude of the largest negative component (closest to zero).
    $$ d_{inside} = \max(q_x, \max(q_y, q_z)) $$
    Note: If the point is inside, $d_{inside}$ will be negative. [S11, S13, S14, unofficial]

3.  **Outside Distance:** If the point is outside ($d_{inside} > 0$), the distance is the Euclidean length of the positive components of $q$ (the parts of the vector extending beyond the box faces).
    $$ d_{outside} = \| \max(q, 0) \|_2 $$
    Where $\max(q, 0)$ is the component-wise maximum of $q$ and the zero vector, and $\| \cdot \|_2$ is the L2 norm (Euclidean length). [S11, S13, S14, unofficial]

4.  **Combine:**
    *   If $d_{inside} > 0$ (point is outside), return $d_{outside}$.
    *   Otherwise (point is inside or on surface), return $d_{inside}$.
    
    This can be written compactly as:
    $$ d(p) = \max(q_x, \max(q_y, q_z)) + \| \max(q, 0) \|_2 $$
    Wait, this compact form is actually **Version 1** and is incorrect for corners [S13, unofficial]. The correct logic is:
    $$ d(p) = \begin{cases} \| \max(q, 0) \|_2 & \text{if } \max(q) > 0 \\ \max(q) & \text{otherwise} \end{cases} $$
    Actually, the standard compact formula often cited as "correct" in GLSL libraries (like [S9, unofficial]) is:
    $$ d(p) = \text{length}(\max(q, 0)) + \min(\max(q_x, \max(q_y, q_z)), 0) $$
    Let's verify this compact formula.
    If inside: $\max(q) < 0$. $\max(q, 0) = 0$. Length is 0. Result is $\max(q)$, which is negative. Correct.
    If outside: $\max(q) > 0$. $\max(q, 0)$ has positive components. Length is $d_{outside}$. $\min(\max(q), 0) = 0$. Result is $d_{outside}$. Correct.
    This compact formula is mathematically equivalent to the if/else logic and is the standard implementation [S9, S12, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def signed_box_distance(p: Vector3, center: Vector3, half_extents: Vector3) -> float:
    """
    Computes the signed Euclidean distance from point p to an axis-aligned box.
    
    Args:
        p: Query point (Vector3).
        center: Center of the box (Vector3).
        half_extents: Half-lengths of the box along x, y, z axes (Vector3).
        
    Returns:
        Signed distance. Negative if inside, positive if outside.
    """
    # 1. Shift point to box-centered coordinates and take absolute value
    # q = |p - center| - half_extents
    q = Vector3(
        abs(p.x - center.x) - half_extents.x,
        abs(p.y - center.y) - half_extents.y,
        abs(p.z - center.z) - half_extents.z
    )
    
    # 2. Calculate the maximum component of q (distance to nearest face if inside)
    max_q = max(q.x, max(q.y, q.z))
    
    # 3. Calculate the Euclidean distance of the positive components (distance to edge/corner if outside)
    # max(q, 0) clamps negative components to 0
    positive_q = Vector3(
        max(q.x, 0.0),
        max(q.y, 0.0),
        max(q.z, 0.0)
    )
    outside_dist = positive_q.length()
    
    # 4. Combine
    # If inside (max_q < 0), outside_dist is 0, so result is max_q (negative).
    # If outside (max_q > 0), max_q term is clamped to 0 by min(..., 0) logic implicitly 
    # handled by the addition: max_q + outside_dist? 
    # No, the formula is: length(max(q,0)) + min(max(q), 0)
    
    inside_term = min(max_q, 0.0)
    
    return outside_dist + inside_term
```

**Parameters:**
*   `center`: The 3D coordinates of the box center. Default: $(0,0,0)$ [S9, unofficial].
*   `half_extents` (or `b`): The distance from the center to the faces along each axis. For a $1 \times 1 \times 1$ box, $b = (0.5, 0.5, 0.5)$ [S13, unofficial].

**Edge Cases and Pitfalls:**
*   **Version 1 Artifact:** A common incorrect implementation is $d = \max(q) + \| \max(q, 0) \|_2$? No, the incorrect one is often cited as $d = \min(\max(q), \| \max(q, 0) \|_2)$ or simply using the max of the two without the conditional logic correctly, or specifically the formula $d = \max(|p|-b)$ which underestimates distance at corners [S13, S14, S15, unofficial].
*   **Underestimation:** Using the "cheap" approximation $d = \max(|p-c| - s)$ (where $s$ is side length, not half-extent) or the uncorrected max/min logic leads to underestimating the distance in corner regions. This causes ray marching to take smaller steps than necessary (slower convergence) and produces incorrect ambient occlusion (dark spots) [S13, S14, unofficial].
*   **Overestimation:** Any function that overestimates the distance risks the ray overshooting the surface, leading to intersection errors [S13, unofficial]. The exact SDF guarantees no overestimation.

## Subtopics

### [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md)
Computes the Euclidean distance from a query point to the surface of an axis-aligned box centered at the origin, returning only non-negative values regardless of whether the point lies inside or outside the volume. Unlike its sibling, the Signed Box Distance Function, which distinguishes interior from exterior space, this method treats the box as a solid boundary where internal points are measured by their distance to the nearest face.

### [Box with Edge Rounding SDF](4004-box-with-edge-rounding-sdf.md)
Computes the signed Euclidean distance from a query point to the surface of an axis-aligned box whose edges and corners are rounded by a constant radius. It solves the problem of defining smooth, organic variations of standard rectangular primitives for ray marching and implicit modeling. It differs from the [Signed Box Distance Function](1092-signed-box-distance-function.md) by replacing sharp 90-degree corners with spherical fillets, allowing for continuous normals and softer visual interactions.

### [Rectangle Signed Distance Function](4031-rectangle-signed-distance-function.md)
Computes the exact Euclidean distance from a query point to the boundary of an axis-aligned rectangle centered at the origin. It is the 2D analog of the [Signed Box Distance Function](1092-signed-box-distance-function.md).

### [Oriented Box2D SDF](4370-oriented-box2d-sdf.md)
Computes the signed Euclidean distance from a 2D point to a line segment of arbitrary length and thickness, effectively representing a 2D box with rounded corners oriented along the segment axis. It differs from the [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md) by providing signed output and from the [Signed Box Distance Function](1092-signed-box-distance-function.md) by operating in 2D with a specific segment-based orientation rather than a general 3D axis-aligned or oriented box. It serves as a primitive for constructing 2D shapes in ray marching or implicit modeling.

### Oriented Box SDF
Computes the signed distance from a point to an Oriented Bounding Box (OBB) [S16, unofficial]. It transforms the input point into the box's local coordinate system using the inverse of the box's transformation matrix, computes the distance to the unit box scaled by the box's dimensions, and adjusts for being inside or outside [S16, unofficial].
**Steps:**
1.  Transform point `p` to local space: `trans = full_inverse * (p, 1.0)` [S16, unofficial].
2.  Compute local difference: `q_local = (abs(trans.xyz) - 1.0) * scale` [S16, unofficial].
3.  Compute distance using the same logic as the axis-aligned box: `dist = length(max(q_local, 0)) + min(max(q_local), 0)` [S16, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$. The function involves a constant number of arithmetic operations (subtractions, absolute values, max/min, and one square root for the L2 norm).
*   **Memory Complexity:** $O(1)$. No auxiliary data structures are needed.
*   **Performance:** Extremely fast, suitable for real-time ray marching. The square root operation is the most expensive part, but it is only computed for the outside distance.
*   **Strengths:** Exact Euclidean distance; safe for sphere tracing (no overshooting); differentiable almost everywhere (gradient provides surface normal); handles inside/outside correctly.
*   **Limitations:** The square root operation is slightly more expensive than the "cheap" approximation. The cheap approximation is sometimes used for performance when visual quality (AO) is not critical, but it is technically incorrect [S15, unofficial].
*   **When to use:** Standard choice for axis-aligned boxes in ray marching, implicit modeling, and collision detection.
*   **When not to use:** If the box is rotated, use the Oriented Box SDF. If performance is critical and visual artifacts are acceptable, the cheap approximation might be used, but it is generally discouraged for high-quality rendering.

## Background
The Signed Box Distance Function is a fundamental primitive in Signed Distance Field (SDF) ray marching. The exact formulation (Version 2) was popularized by Inigo Quilez in his online articles on distance functions [S2, S13, S14, unofficial]. Earlier or simpler approximations (Version 1) were used in early GPU ray marching implementations but were shown to produce artifacts in ambient occlusion due to distance underestimation at corners [S13, S14, unofficial]. The function is widely implemented in GLSL libraries and graphics engines [S9, S12, unofficial].

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
[S10] nschaievitch/sdf-editor, https://github.com/nschaievitch/sdf-editor, fetched 2026-10-26 (unofficial)
[S11] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S12] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S13] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S14] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S15] Volumetric Rendering: Signed Distance Functions - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
[S16] holger-kibur/sdf, https://github.com/holger-kibur/sdf, fetched 2026-10-27 (unofficial)
