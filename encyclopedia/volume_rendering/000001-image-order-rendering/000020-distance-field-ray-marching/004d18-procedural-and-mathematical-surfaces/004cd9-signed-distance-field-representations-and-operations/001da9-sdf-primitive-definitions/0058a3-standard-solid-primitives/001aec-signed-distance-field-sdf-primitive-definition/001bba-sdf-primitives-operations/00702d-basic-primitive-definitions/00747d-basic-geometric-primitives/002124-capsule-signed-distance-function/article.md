# Capsule Signed Distance Function

Also called: 3D Capsule SDF, Capsule Rendering via Distance Check, Capsule SDF, Capsule3D SDF, Signed Capsule, sdCapsule

## Overview
The Capsule Signed Distance Function (SDF) computes the shortest Euclidean distance from a query point $p$ to the surface of a capsule, defined as a cylinder of radius $r$ capped by hemispheres at its endpoints. It solves the problem of efficiently determining spatial containment and proximity for volumetric rendering or collision detection of elongated, rounded shapes. It differs from the [Sphere SDF](1237-sphere-signed-distance-function.md) by supporting an arbitrary axis and length, and from the [Cone SDF](2594-cone-signed-function.md) by maintaining a constant radius along its central axis.

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $p \in \mathbb{R}^3$; Capsule endpoints $a, b \in \mathbb{R}^3$; Radius $r \in \mathbb{R}^+$.
*   **Output:** Signed distance $d \in \mathbb{R}$.

**Data Structures:**
No persistent data structures are required. The computation uses temporary vectors and scalars.

**Parameters:**
*   $r$: Radius of the capsule. [S4, unofficial] uses $0.12$ as a hardcoded example, but $r$ is a general parameter.
*   $a, b$: The two endpoints defining the capsule's central axis.

**Algorithm:**
The function calculates the distance by projecting the query point onto the line segment defined by $a$ and $b$, clamping the projection to the segment bounds, and computing the distance from the point to this closest point on the segment, then subtracting the radius.

1.  Compute the vector from endpoint $a$ to the query point $p$:
    $$ pa = p - a $$
2.  Compute the vector representing the capsule's axis:
    $$ ba = b - a $$
3.  Compute the squared length of the axis vector:
    $$ len\_sq = ba \cdot ba $$
4.  Determine the projection parameter $h$ along the axis. If the axis has negligible length ($len\_sq < \epsilon$), $h$ is 0. Otherwise, project $pa$ onto $ba$ and clamp the result to $[0, 1]$:
    $$ h = \text{clamp}\left(\frac{pa \cdot ba}{len\_sq}, 0.0, 1.0\right) $$
    where $\text{clamp}(v, min, max)$ returns $min$ if $v < min$, $max$ if $v > max$, and $v$ otherwise.
5.  Compute the closest point on the segment to $p$:
    $$ closest = a + ba \cdot h $$
    Alternatively, using the precomputed $pa$ and $ba$:
    $$ closest\_relative = pa - ba \cdot h $$
6.  Compute the Euclidean distance from $p$ to the closest point:
    $$ dist = \| closest\_relative \| $$
7.  Subtract the radius to get the signed distance:
    $$ d = dist - r $$

**Edge Cases and Pitfalls:**
*   **Zero-Length Capsule:** If $a = b$, $len\_sq = 0$. Division by zero must be avoided. The sources handle this by clamping $h$ to 0.0 if $len\_sq$ is below a threshold (e.g., $1e-12$) [S5, unofficial]. In this case, the capsule degenerates to a sphere centered at $a$ with radius $r$.
*   **Precision:** Using squared lengths avoids square roots in the projection step, improving performance and stability.
*   **Sign Convention:** The sign is determined by the final subtraction. $d < 0$ implies the point is inside the capsule volume.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sd_capsule(p, a, b, r):
    """
    Computes the signed distance from point p to a capsule defined by endpoints a, b and radius r.
    
    Args:
        p: tuple or list of 3 floats (x, y, z)
        a: tuple or list of 3 floats (x, y, z)
        b: tuple or list of 3 floats (x, y, z)
        r: float, radius of the capsule
        
    Returns:
        float: signed distance
    """
    # Vector from a to p
    pa_x = p[0] - a[0]
    pa_y = p[1] - a[1]
    pa_z = p[2] - a[2]
    
    # Vector from a to b
    ba_x = b[0] - a[0]
    ba_y = b[1] - a[1]
    ba_z = b[2] - a[2]
    
    # Squared length of the capsule axis
    len_sq = ba_x * ba_x + ba_y * ba_y + ba_z * ba_z
    
    # Projection parameter h
    if len_sq < 1e-12:
        h = 0.0
    else:
        # Dot product of pa and ba
        dot_pa_ba = pa_x * ba_x + pa_y * ba_y + pa_z * ba_z
        # Clamp h to [0, 1]
        h = dot_pa_ba / len_sq
        if h < 0.0:
            h = 0.0
        elif h > 1.0:
            h = 1.0
            
    # Closest point on the segment relative to a
    # closest_relative = pa - h * ba
    closest_rel_x = pa_x - h * ba_x
    closest_rel_y = pa_y - h * ba_y
    closest_rel_z = pa_z - h * ba_z
    
    # Distance from p to the closest point on the segment
    dist = math.sqrt(closest_rel_x * closest_rel_x + 
                     closest_rel_y * closest_rel_y + 
                     closest_rel_z * closest_rel_z)
                     
    # Signed distance
    return dist - r
```

**Subtopics:**
*   [Segment Signed Distance Function](2679-segment-signed-distance-function.md): Computes distance to a 1D line segment. The capsule SDF is essentially the segment SDF with an offset by radius $r$. Choose the segment SDF when radius is zero or for pure geometric line queries.
*   [Cone Signed Distance Function](2594-cone-signed-distance-function.md): Computes distance to a cone. Choose the cone SDF when the shape tapers to a point or has a varying radius.

## Operating characteristics
*   **Time Complexity:** $O(1)$. The calculation involves a constant number of arithmetic operations (subtractions, multiplications, dot products, one square root).
*   **Memory Complexity:** $O(1)$. No auxiliary data structures are needed.
*   **Strengths:** Extremely fast evaluation, making it suitable for real-time ray marching and GPU-based volume rendering. It provides smooth, differentiable surfaces (except at the junction of the cylinder and hemispheres, where the normal is continuous but the derivative might have discontinuities depending on implementation, though standard SDFs are generally considered smooth enough for shading).
*   **Limitations:** Only represents a specific shape. Cannot represent arbitrary meshes. Precision issues may arise with very small radii or very long capsules if not handled carefully, though the clamping logic mitigates most numerical instability.
*   **When to use:** When rendering or detecting collisions with capsule-shaped objects (e.g., character skeletons, pipes, cables).
*   **When not to use:** For complex, non-convex, or highly detailed geometries where mesh-based or voxel-based representations are more appropriate.

## Background
The capsule SDF is a standard primitive in Signed Distance Field rendering, popularized by Inigo Quilez in his online articles and GLSL code snippets [S3, S5, unofficial]. It is widely used in ray marching demos and real-time graphics engines due to its simplicity and efficiency. The function is often implemented as `sdCapsule` in GLSL libraries [S3, S4, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
[S3] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S4] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S5] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
