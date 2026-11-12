# Signed Distance Field (SDF) Primitive Definition

Also called: SDF Function, SDF Shapes, Signed Distance Field (SDF), Signed Distance Function (SDF), Signed Distance Function Primitives

## Overview
A Signed Distance Field (SDF) Primitive Definition is an analytical function that returns the shortest Euclidean distance from a query point $\mathbf{p}$ to the surface of a geometric primitive, with a sign indicating whether the point is inside (negative), outside (positive), or on (zero) the surface [S2, S3, S14, unofficial]. It solves the problem of representing explicit mesh tessellation with continuous scalar fields, allowing for efficient ray marching and procedural composition of shapes [G]. What sets this method apart from its siblings is the distinction between **domain manipulation** (transforming the input coordinate $\mathbf{p}$ before evaluation, as in [SDF Transformation](1086-sdf-transformation.md)) and **value-domain operations** (combining the output distance values $d$ of primitives, as described here) [S1, S3, S15, unofficial]. Unlike [k-SDF](9530-k-sdf.md), which approximates distances for specific optimization purposes, these primitives provide exact or analytically derived distance metrics for canonical shapes and their combinations [S1, S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D position vector $\mathbf{p} = (p_x, p_y, p_z)$ in the primitive's local coordinate space, and primitive-specific parameters (e.g., radius $r$, center $\mathbf{c}$, half-extents $\mathbf{b}$).
*   **Output:** A scalar float $d = \text{sdf}(\mathbf{p})$.

**Data Structures:**
Primitives are typically defined by simple structs containing their geometric parameters. Common layouts include [S14, unofficial]:
*   `struct sph`: `float r` (radius), `vec3 c` (center).
*   `struct box`: `vec3 p` (position), `vec3 size` (full size).
*   `struct tor`: `float inR` (tube radius), `float outR` (major radius).
*   `struct plane`: `vec4 k` (normal xyz, offset w).

**Numbered Steps:**
1.  **Local Space Evaluation:** The function evaluates the distance from $\mathbf{p}$ to the primitive's surface in its canonical local space (usually centered at the origin, aligned with axes) [S6, S13, unofficial].
2.  **Distance Calculation:** The Euclidean distance is calculated using the $L_2$ norm, unless a specific approximation (like $L_8$ norm) is used for rounded corners [S2, S4, S14, unofficial].
3.  **Sign Application:** The sign is determined by the position relative to the surface:
    *   $d < 0$: Inside [S15, unofficial].
    *   $d = 0$: On surface [S15, unofficial].
    *   $d > 0$: Outside [S15, unofficial].
4.  **Transformation (Optional):** If the primitive is in world space, the query point $\mathbf{p}$ is transformed into local space using the inverse of the primitive's world transformation matrix before evaluation [S10, G, unofficial].

**Formulas:**
*   **Sphere:** Centered at origin with radius $r$:
    $$ d(\mathbf{p}) = \|\mathbf{p}\| - r = \sqrt{p_x^2 + p_y^2 + p_z^2} - r $$
    For a sphere centered at $\mathbf{c}$:
    $$ d(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - r $$ [S2, S3, S4, S6, S11, S14, unofficial]
*   **Box (AABB):** Centered at origin with half-extents $\mathbf{b} = (b_x, b_y, b_z)$:
    $$ \mathbf{d} = |\mathbf{p}| - \mathbf{b} $$
    $$ d(\mathbf{p}) = \|\max(\mathbf{d}, 0)\| + \min(\max(d_x, \max(d_y, d_z)), 0) $$
    This computes the exact distance to corners. A "cheap" box overestimates distance: $d_{cheap} = \max(|\mathbf{p}| - \mathbf{b})$ [S6, S14, unofficial].
*   **Plane:** Defined by normal $\mathbf{n}$ (unit length) and distance from origin $d_{offset}$:
    $$ d(\mathbf{p}) = \mathbf{p} \cdot \mathbf{n} + d_{offset} $$ [S6, S14, unofficial]
*   **Cylinder (Infinite):** Axis-aligned with Y, radius $r$:
    $$ d(\mathbf{p}) = \|\mathbf{p}_{xz}\| - r = \sqrt{p_x^2 + p_z^2} - r $$ [S14, unofficial]
*   **Cylinder (Capped):** Radius $r$, half-height $h$:
    $$ d_x = \|\mathbf{p}_{xz}\| - r, \quad d_y = |p_y| - h $$
    $$ d(\mathbf{p}) = \min(\max(d_x, d_y), 0) + \|\max(d_x, d_y, 0)\| $$ [S14, unofficial]
*   **Torus:** Major radius $R_{out}$, minor radius $R_{in}$, centered at origin, aligned with Y:
    $$ d(\mathbf{p}) = \|(\|\mathbf{p}_{xz}\| - R_{out}, p_y)\| - R_{in} $$ [S6, S13, S14, unofficial]
*   **Cone:** Radius $r$, height $h$, axis-aligned with Y:
    $$ \phi = \arctan(r/h) $$
    $$ d(\mathbf{p}) = \max(\sqrt{p_x^2 + p_z^2}\cos(\phi) - |p_y|\sin(\phi), \max(p_y - h, -p_y)) $$ [S14, unofficial]
*   **Capsule:** Endpoints $\mathbf{a}, \mathbf{b}$, radius $r$:
    $$ d(\mathbf{p}) = \text{distance}(\mathbf{p}, \text{segment } \mathbf{a}\mathbf{b}) - r $$ [S6, S13, unofficial]

**Parameters:**
*   **Radius ($r$):** Typically positive floats. Example: Sphere $r=1$ [S3, unofficial].
*   **Half-extents ($\mathbf{b}$):** Positive floats. Example: Box $\mathbf{b}=(20, 20, 5)$ [S4, unofficial].
*   **Smoothing Constant ($k$):** For smooth unions, $k \approx 0.2$ [S11, unofficial].
*   **Gradient Step ($\epsilon$):** For normal approximation, $\epsilon = 0.0001$ [S11, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Closed Form:** Not every shape has a closed-form SDF. Arbitrary meshes require approximate SDFs (baked into 3D textures or generated from BVH traversal) [S15, unofficial].
*   **GDF Performance:** Generalized Distance Functions (GDF) using loops for polyhedra (e.g., Icosahedron) may fail to unroll on certain drivers (e.g., Nvidia 350.12), causing slowdowns [S6, S13, unofficial].
*   **Incorrect Bounds:** Some illustrative functions (e.g., `fBlob`) are not correct distance bounds and should not be used for sphere tracing [S6, S13, unofficial].
*   **Sharp Discontinuities:** Taking the min/max of two SDFs yields a sharp discontinuity in the derivative, which may affect shading smoothness unless blended [S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def abs_vec(v):
    return (abs(v[0]), abs(v[1]), abs(v[2]))

def max_vec(a, b):
    return (max(a[0], b[0]), max(a[1], b[1]), max(a[2], b[2]))

def min_vec(a, b):
    return (min(a[0], b[0]), min(a[1], b[1]), min(a[2], b[2]))

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def normalize(v):
    l = length(v)
    if l == 0: return (0,0,0)
    return (v[0]/l, v[1]/l, v[2]/l)

def clamp(x, min_val, max_val):
    return max(min_val, min(max_val, x))

def mix(a, b, t):
    return a + t * (b - a)

def sdf_sphere(p, center, radius):
    """
    Returns signed distance from p to sphere.
    p: (x, y, z)
    center: (cx, cy, cz)
    radius: float
    """
    diff = (p[0]-center[0], p[1]-center[1], p[2]-center[2])
    return length(diff) - radius

def sdf_box(p, center, half_size):
    """
    Returns signed distance from p to AABB box.
    p: (x, y, z)
    center: (cx, cy, cz)
    half_size: (bx, by, bz)
    """
    # Transform p to local space relative to center
    local_p = (p[0]-center[0], p[1]-center[1], p[2]-center[2])
    d = abs_vec(local_p)
    d = (d[0]-half_size[0], d[1]-half_size[1], d[2]-half_size[2])
    
    outside = max_vec(d, (0,0,0))
    inside = min_vec(d, (0,0,0))
    
    dist_sq = outside[0]**2 + outside[1]**2 + outside[2]**2
    if dist_sq == 0:
        # Point is inside
        max_inside = max(inside[0], max(inside[1], inside[2]))
        return max_inside
    else:
        return math.sqrt(dist_sq) + max_inside

def sdf_torus(p, center, major_radius, minor_radius):
    """
    Returns signed distance from p to torus.
    Torus lies in XZ plane, centered at center.
    """
    # Transform to local
    local_p = (p[0]-center[0], p[1]-center[1], p[2]-center[2])
    
    # Distance to center of tube in XZ plane
    q = (length((local_p[0], local_p[2])) - major_radius, local_p[1])
    return length(q) - minor_radius

def sdf_plane(p, normal, offset):
    """
    Returns signed distance from p to plane.
    normal: unit vector (nx, ny, nz)
    offset: distance from origin along normal
    """
    return dot(p, normal) + offset

def sdf_capsule(p, a, b, radius):
    """
    Returns signed distance from p to capsule defined by segment ab and radius.
    """
    ab = (b[0]-a[0], b[1]-a[1], b[2]-a[2])
    ap = (p[0]-a[0], p[1]-a[1], p[2]-a[2])
    
    ab_sq = dot(ab, ab)
    if ab_sq == 0:
        # a and b are same point, treat as sphere
        return length(ap) - radius
        
    t = clamp(dot(ap, ab) / ab_sq, 0.0, 1.0)
    closest = (a[0] + t*ab[0], a[1] + t*ab[1], a[2] + t*ab[2])
    dist_to_closest = length((p[0]-closest[0], p[1]-closest[1], p[2]-closest[2]))
    
    return dist_to_closest - radius

def sdf_union(d1, d2):
    """
    Combines two SDFs using minimum (Union).
    """
    return min(d1, d2)

def sdf_intersection(d1, d2):
    """
    Combines two SDFs using maximum (Intersection).
    """
    return max(d1, d2)

def sdf_subtraction(d1, d2):
    """
    Subtracts shape 2 from shape 1.
    """
    return max(d1, -d2)

def smin(a, b, k=0.2):
    """
    Smooth minimum blending.
    """
    h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0)
    return mix(b, a, h) - k * h * (1.0 - h)
```

## Subtopics

### [SDF Primitives & Operations](1075-sdf-primitives-operations.md)
Provides foundational analytical functions and Boolean/combinatorial logic. It distinguishes between domain manipulation (transforming $\mathbf{p}$) and value-domain operations (combining $d$). Unlike [k-SDF](9530-k-sdf.md), it provides exact metrics for canonical shapes.

### [SDF Transformation](1086-sdf-transformation.md)
Applies geometric transformations (rotation, translation, scaling) by manipulating query point coordinates before evaluation. Unlike [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md), it relies on inverse mapping of isometries or explicit scaling factors.

### [k-SDF](9530-k-sdf.md)
Represents fuzzy geometry using $k$ lightweight mesh shells and view-dependent transparency. It is sorting-free and bounds samples per ray, enabling real-time rendering on low-cost smartphones.

### [SDF Representation DOP](12996-sdf-representation-dop.md)
Creates a volumetric SDF from geometry for fast inside/outside testing in collision detection, focusing on topological correctness rather than visual appearance.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per primitive evaluation. The cost is dominated by the number of ray march steps, which depends on the scene complexity and desired precision.
*   **Memory Complexity:** $O(1)$ for analytical primitives. No mesh or voxel storage is required for the geometry definition itself [S15, unofficial].
*   **Measured Performance:** Sphere tracing with SDFs is significantly faster than triangle intersection tests for complex procedural scenes because it skips empty space using the distance lower bound [S1, S8, unofficial].
*   **Strengths:**
    *   **Robustness:** The Lipschitz continuity guarantees that sphere tracing does not overshoot the surface [G, S14, unofficial].
    *   **Composition:** Easy to combine shapes using CSG operations (min/max) [S11, unofficial].
    *   **Smoothness:** Analytical normals can be approximated via finite differences, providing smooth shading for procedural geometry [S11, unofficial].
*   **Limitations:**
    *   **Approximation:** Complex or arbitrary meshes cannot be represented exactly without approximation (e.g., voxelization or neural networks) [S15, unofficial].
    *   **Sharp Corners:** Basic CSG operations (min/max) create sharp discontinuities in normals, which may require smoothing techniques for aesthetic purposes [S11, unofficial].
    *   **Precision:** Near the surface, floating-point precision errors can affect normal calculation and step size [S11, unofficial].

## Background
Signed Distance Fields originated in the field of computer graphics for representing implicit surfaces. The concept of using distance functions for ray marching was popularized by Inigo Quilez and others in the demoscene and real-time graphics community [S1, S7, S11, unofficial]. The term "SDF" is widely used in procedural rendering to replace explicit mesh tessellation with continuous scalar fields [G]. The mathematical foundation relies on the properties of Euclidean distance and Lipschitz continuity [S2, S14, unofficial].

## Sources
[S1] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S2] Implicit Functions and extracting an isosurface, https://gamedev.stackexchange.com/questions/85873/implicit-functions-and-extracting-an-isosurface, fetched 2026-10-26 (unofficial)
[S3] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S4] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S5] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S6] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S7] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S8] Ray Marching: Signed Distance Fields, Sphere Tracing, and SDF Shading, https://tutorials.tektite.studio/ray-marching/, fetched 2026-10-27 (unofficial)
[S9] AJKKinney/Mandelbulb, https://github.com/AJKKinney/Mandelbulb, fetched 2026-10-27 (unofficial)
[S10] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S11] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1819/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S12] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S13] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S14] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S15] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
