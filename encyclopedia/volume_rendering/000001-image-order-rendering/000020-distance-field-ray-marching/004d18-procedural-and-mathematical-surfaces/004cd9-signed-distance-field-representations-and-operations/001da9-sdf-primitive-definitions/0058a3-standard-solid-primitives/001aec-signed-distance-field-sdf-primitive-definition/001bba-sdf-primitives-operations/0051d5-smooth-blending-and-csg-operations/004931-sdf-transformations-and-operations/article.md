# SDF Transformations and Operations

Also called: SDF Boolean and Transform Operations, SDF Primitives & Operations > Smooth Blending and CSG Operations > SDF Transformations and Operations, Smooth Blending and Domain Repetition, Smooth Minimum, Smooth Union

## Overview
This method defines the mathematical operators and coordinate transformations required to construct complex procedural geometry from primitive Signed Distance Fields (SDFs). It solves the problem of combining simple shapes into complex models and positioning them in world space. It is distinguished from exact Boolean operations by its support for smooth blending approximations, which allow for continuous gradients and organic transitions between shapes, at the cost of distance field accuracy [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{x} \in \mathbb{R}^3$ in world space, primitive SDF functions $S_i(\mathbf{p})$, transformation parameters (translation $\mathbf{t}$, rotation $\mathbf{R}$, scale $s$), and blend parameters $k$.
*   **Output:** A scalar distance value $d$ representing the distance to the nearest surface.

**Data Structures:**
Operations are often stored in a tree structure within GPU buffers. The specific layout for an SDF operation block is defined as [S3, unofficial]:

**SdfOpSpecificBlock**
*   `mat4s`: Array of 2 `Mat4` matrices.
*   `vec4s`: Array of 3 `Vec4` vectors.
*   `floats`: Array of 2 `f32` scalars.

**SdfBoundingBoxBlock**
*   `matrix`: `Mat4`.
*   `scale`: `Vec4`.
*   `full_inverse`: `Mat4`.
*   `trans_inverse`: `Mat4`.

**SdfOpBlock** (Parent container)
*   `op_code`: `u32` (Operation identifier).
*   `is_primitive`: `bool`.
*   `parent_is_union`: `bool`.
*   `len`: `u32`.
*   `level`: `u32`.
*   `op_specific`: `SdfOpSpecificBlock`.
*   `bounding_box`: `SdfBoundingBoxBlock`.
*   `other_box`: `SdfBoundingBoxBlock`.

**Numbered Steps:**

1.  **Coordinate Transformation:** Transform the query point $\mathbf{x}$ into the primitive's local space.
    *   Subtract translation: $\mathbf{p} = \mathbf{x} - \mathbf{t}$.
    *   Apply rotation: $\mathbf{p} = \mathbf{R}^T \mathbf{p}$ (assuming $\mathbf{R}$ is orthonormal).
    *   Apply scale: $\mathbf{p} = \mathbf{p} / s$.
    *   The resulting distance $d_i$ must be divided by the scale factor $|s|$ to maintain correct distance metric properties [S2, G, unofficial].

2.  **Domain Repetition (Tiling):** For periodic structures, apply a modulo operation to the local coordinates before evaluation [S2, unofficial]:
    $$ \mathbf{p}_{mod} = \mathbf{p} - \lfloor \mathbf{p} / \mathbf{period} + 0.5 \rfloor \cdot \mathbf{period} $$
    Evaluate the primitive at $\mathbf{p}_{mod}$.

3.  **Exact Boolean Operations:** Combine distances $d_1$ and $d_2$:
    *   **Union:** $d_{union} = \min(d_1, d_2)$. This returns the distance to the closest shape exterior edge.
    *   **Intersection:** $d_{intersect} = \max(d_1, d_2)$.
    *   **Subtraction:** $d_{sub} = \max(d_1, -d_2)$ (subtracting shape 2 from shape 1) [S2, unofficial].

4.  **Smooth Blending:** Approximate unions/subtractions with a transition controlled by parameter $k$. These are no longer exact distance fields but distorted approximations [S2, unofficial].

    **Exponential Smooth Minimum:**
    $$ \text{smin}(a, b, k) = -k \cdot \log_2(2^{-a/k} + 2^{-b/k}) $$
    Alternatively using natural base $e$:
    $$ \text{smin}(a, b, k) = -k \cdot \ln(e^{-a/k} + e^{-b/k}) $$

    **Polynomial (Hermite) Smooth Minimum:**
    $$ h = \text{clamp}\left(\frac{0.5 + 0.5(b - a)}{k}, 0.0, 1.0\right) $$
    $$ \text{smin}(a, b, k) = \text{lerp}(b, a, h) - k \cdot h \cdot (1.0 - h) $$
    Where $\text{lerp}(b, a, h) = (1.0 - h)a + hb$ [S2, G, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def transform_point(x, t, R, s):
    """
    Transform world point x to local space.
    t: translation vector
    R: rotation matrix (orthonormal)
    s: scale factor (scalar)
    """
    p = x - t
    p = R.transpose() @ p
    p = p / s
    return p

def domain_repetition(p, period):
    """
    Apply modulo tiling to local coordinates.
    p: local point vector
    period: period vector
    """
    # floor(p / period + 0.5) * period
    shift = (p / period + 0.5).floor() * period
    return p - shift

def exact_union(d1, d2):
    return min(d1, d2)

def exact_intersection(d1, d2):
    return max(d1, d2)

def exact_subtraction(d1, d2):
    """Subtract shape 2 from shape 1."""
    return max(d1, -d2)

def smooth_min_exponential(a, b, k):
    """
    Exponential smooth minimum.
    k: blend factor (smaller k = sharper transition)
    """
    if k == 0:
        return min(a, b)
    r = math.exp2(-a / k) + math.exp2(-b / k)
    return -k * math.log2(r)

def smooth_min_polynomial(a, b, k):
    """
    Polynomial (Hermite) smooth minimum.
    k: blend factor
    """
    if k == 0:
        return min(a, b)
    h = max(0.0, min(1.0, (0.5 + 0.5 * (b - a)) / k))
    lerp_val = (1.0 - h) * a + h * b
    return lerp_val - k * h * (1.0 - h)

def evaluate_sdf(x, primitives, transforms, blend_func, k):
    """
    Evaluate combined SDF.
    primitives: list of SDF functions
    transforms: list of (t, R, s) tuples
    blend_func: function for combining distances
    k: blend parameter
    """
    # Assuming binary tree structure or sequential combination
    # This is a simplified example for two primitives
    p1_local = transform_point(x, *transforms[0])
    d1 = primitives[0](p1_local) / transforms[0][2] # Divide by scale
    
    p2_local = transform_point(x, *transforms[1])
    d2 = primitives[1](p2_local) / transforms[1][2] # Divide by scale
    
    return blend_func(d1, d2, k)
```

**Parameters:**
*   $k$: The smoothness parameter. Smaller values yield sharper transitions closer to exact Boolean operations. Typical values depend on the scale of the scene.

**Edge Cases and Pitfalls:**
*   **Interior Distance Breaks:** Union breaks the interior distance field, returning a greater distance than it should. This can break outline thickness or require more raymarching steps [S2, unofficial].
*   **Exterior Distance Breaks:** Subtraction and intersection break the exterior distance field [S2, unofficial].
*   **Approximation Errors:** Smooth blending functions are no longer exact distance fields but distorted approximations. The more distorted they become, the more difficult they may be to work with [S2, unofficial].

## Subtopics

### SDF Operation Block
A data structure for storing SDF operations in a buffer, typically used in GPU compute shaders. It contains operation codes, primitive flags, and specific transformation data. Layout (repr C): `op_code` (u32), `is_primitive` (bool), `parent_is_union` (bool), `len` (u32), `level` (u32), `op_specific` (SdfOpSpecificBlock), `bounding_box` (SdfBoundingBoxBlock), `other_box` (SdfBoundingBoxBlock) [S3, unofficial].

### SDF Union Operation
The distance to the closest shape exterior edge is the lesser of the two distances. Can be repeated for 3 or more shapes by chaining the operation. Pitfall: Union breaks the interior distance field, returning a greater distance than it should. This can break outline thickness or require more raymarching steps [S2, unofficial].

### SDF Subtraction
To subtract shape 2 from shape 1: take the maximum of shape_dist1 and the negation of shape_dist2. To subtract shape 1 from shape 2: take the maximum of shape_dist2 and the negation of shape_dist1. Pitfall: Subtraction breaks the exterior distance field [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per primitive evaluation. Complex scenes require traversing the operation tree, so complexity is proportional to the number of primitives.
*   **Memory Complexity:** Depends on the storage of the operation tree. The `SdfOpBlock` structure requires fixed-size buffers.
*   **Strengths:** Allows for infinite resolution and compact storage of complex geometry. Smooth blending enables organic shapes.
*   **Limitations:** Smooth blending is not an exact distance field, which can affect shading and normal calculation accuracy. Exact Boolean operations can break distance field properties (interior/exterior), requiring more raymarching steps or specialized handling [S2, unofficial].
*   **When to use:** When procedural generation, infinite resolution, or smooth blending is required.
*   **When not to use:** When exact distance field properties are critical for all operations, or when the scene complexity exceeds the capacity of the operation tree structure.

## Background
Inigo Quilez produced the first results recognizable as modern SDF in 2007, demonstrating techniques for efficient soft shadows, smooth blending, and domain repetition [S1, unofficial]. This work built upon earlier foundations by Alex Evan (2006) and Keenan Crane (2005) [S1, unofficial]. The method has since become a standard technique in procedural rendering and raymarching.

## Sources
[S1] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
[S2] Signed Distance Fields - by Xor - GM Shaders, https://mini.gmshaders.com/p/sdf, fetched 2026-10-27 (unofficial)
[S3] holger-kibur/sdf, https://github.com/holger-kibur/sdf, fetched 2026-10-27 (unofficial)
