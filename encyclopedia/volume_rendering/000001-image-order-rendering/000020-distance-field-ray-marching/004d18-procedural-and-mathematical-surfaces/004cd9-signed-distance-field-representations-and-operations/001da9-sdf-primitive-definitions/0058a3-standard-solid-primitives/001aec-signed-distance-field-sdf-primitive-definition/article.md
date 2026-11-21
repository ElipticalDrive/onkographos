# Signed Distance Field (SDF) Primitive Definition

Also called: SDF Function, SDF Shapes, Signed Distance Field (SDF), Signed Distance Function (SDF), Signed Distance Function Primitives

## Overview
A Signed Distance Field (SDF) Primitive Definition is a method for representing 3D geometry using analytical scalar functions that return the shortest Euclidean distance from a query point $\mathbf{p}$ to the surface of a canonical shape, with a sign indicating whether the point is inside (negative) or outside (positive) the shape [S2, S3, S8, S9, S15, unofficial]. It solves the problem of defining complex scene geometry without explicit mesh tessellation, replacing discrete triangle lists with continuous, closed-form algebraic solutions for primitives like spheres, boxes, and tori [S1, S15, unofficial]. Unlike [Distance to Natural Quadrics](1559-distance-to-natural-quadrics.md), which focuses on the geometric properties of conic sections, SDF primitives provide exact distance bounds required for safe sphere tracing in ray marching [S1, S8, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D query point $\mathbf{p} = (p_x, p_y, p_z)$ in world or local space, and primitive-specific parameters (e.g., radius, center, half-extents).
*   **Output:** A scalar float $d = \text{SDF}(\mathbf{p})$ representing the signed distance.

**Data Structures:**
Primitives are typically defined by simple structs or parameter sets. Common layouts include [S14, unofficial]:
*   **Sphere:** `struct sph { float r; vec3 c; }` where `r` is radius and `c` is center.
*   **Box:** `struct box { vec3 p; vec3 size; }` where `p` is position and `size` is full extent (halved in calculation).
*   **Cylinder:** `struct cylinder { vec2 p; float r; }` (infinite) or `struct cylinderCapped { float r; float h; }` (capped, `h` is half-height).
*   **Cone:** `struct cone { float r; float h; vec3 c; }` where `c` is center/offset.
*   **Plane:** `struct plane { vec4 k; }` where `k.xyz` is the normal vector $\mathbf{n}$ and `k.w` is the offset $d_{offset}$.
*   **Torus:** `struct tor { float inR; float outR; }` where `inR` is tube radius and `outR` is major radius.

**Numbered Steps:**
1.  **Transform Point:** If the primitive is transformed (rotated, translated, scaled), map the query point $\mathbf{p}$ into the primitive's local coordinate space using the inverse transformation matrix $M^{-1}$ [S11, S13, unofficial]. For uniform scaling, the distance value must be divided by the scale factor to preserve the Lipschitz condition [S2, unofficial].
2.  **Evaluate Local Distance:** Compute the distance to the nearest point on the untransformed primitive surface using the specific analytical formula for the shape.
3.  **Apply Sign:** The result is positive if $\mathbf{p}$ is outside the surface, negative if inside, and zero if on the surface [S2, S15, unofficial].
4.  **Return Value:** Return the signed distance $d$.

**Formulas:**
The general definition for a shape centered at the origin with radius $r$ is $d(\mathbf{p}) = \|\mathbf{p}\| - r$ [S9, unofficial]. Specific primitives are defined as follows [S6, S11, S13, S14, unofficial]:

*   **Sphere:** Center $\mathbf{c}$, radius $r$.
    $$ d(\mathbf{p}) = \|\mathbf{p} - \mathbf{c}\| - r $$
*   **Box (AABB):** Half-extents $\mathbf{b} = (b_x, b_y, b_z)$, centered at origin.
    $$ \mathbf{d} = |\mathbf{p}| - \mathbf{b} $$
    $$ d(\mathbf{p}) = \|\max(\mathbf{d}, 0)\| + \min(\max(\mathbf{d}_x, \max(\mathbf{d}_y, \mathbf{d}_z)), 0) $$
    *Note: A cheaper, non-exact approximation is $\max(|\mathbf{p}| - \mathbf{b})$ [S6, unofficial].*
*   **Cylinder (Infinite):** Axis-aligned with Y, radius $r$.
    $$ d(\mathbf{p}) = \sqrt{p_x^2 + p_z^2} - r $$
*   **Cylinder (Capped):** Radius $r$, half-height $h$.
    $$ d_x = \sqrt{p_x^2 + p_z^2} - r, \quad d_y = |p_y| - h $$
    $$ d(\mathbf{p}) = \min(\max(d_x, d_y), 0) + \|\max(d_x, d_y, 0)\| $$
*   **Cone:** Radius $r$, height $h$, axis-aligned with Y. Let $\phi = \arctan(r/h)$.
    $$ d(\mathbf{p}) = \max(\sqrt{p_x^2 + p_z^2}\cos(\phi) - |p_y|\sin(\phi), \max(p_y - h, -p_y)) $$
    *Implementation detail: Requires checking distance to tip and base ring if the projection falls outside the mantle [S13, unofficial].*
*   **Plane:** Normal $\mathbf{n}$ (unit length), offset $d_{offset}$.
    $$ d(\mathbf{p}) = \mathbf{p} \cdot \mathbf{n} + d_{offset} $$
*   **Torus:** Major radius $R_{out}$, tube radius $R_{in}$.
    $$ d(\mathbf{p}) = \sqrt{(\sqrt{p_x^2 + p_z^2} - R_{out})^2 + p_y^2} - R_{in} $$
*   **Capsule:** Endpoints $\mathbf{a}, \mathbf{b}$, radius $r$.
    $$ \mathbf{ab} = \mathbf{b} - \mathbf{a} $$
    $$ t = \text{clamp}\left(\frac{(\mathbf{p} - \mathbf{a}) \cdot \mathbf{ab}}{\|\mathbf{ab}\|^2}, 0, 1\right) $$
    $$ d(\mathbf{p}) = \|\mathbf{a} + t\mathbf{ab} - \mathbf{p}\| - r $$

**Parameters:**
*   **Smoothing Constant ($k$):** Used in smooth minimum blending (e.g., $k=0.2$) [S11, unofficial].
*   **Gradient Step ($\epsilon$):** Used for normal approximation, typically $0.0001$ [S11, unofficial].
*   **Lipschitz Bound:** The SDF guarantees that $|f(\mathbf{p}_1) - f(\mathbf{p}_2)| \le \|\mathbf{p}_1 - \mathbf{p}_2\|$, ensuring $d(\mathbf{p})$ is a safe lower bound for ray marching [S15, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def vec3_sub(v1, v2):
    return (v1[0]-v2[0], v1[1]-v2[1], v1[2]-v2[2])

def vec3_add(v1, v2):
    return (v1[0]+v2[0], v1[1]+v2[1], v1[2]+v2[2])

def vec3_scale(v, s):
    return (v[0]*s, v[1]*s, v[2]*s)

def vec3_abs(v):
    return (abs(v[0]), abs(v[1]), abs(v[2]))

def vec3_max(v, s):
    return (max(v[0], s), max(v[1], s), max(v[2], s))

def vec3_min(v, s):
    return (min(v[0], s), min(v[1], s), min(v[2], s))

def vec3_max_component(v):
    return max(v[0], v[1], v[2])

def vec3_min_component(v):
    return min(v[0], v[1], v[2])

def sdf_sphere(p, center, radius):
    """
    Returns signed distance from p to sphere.
    Positive outside, negative inside.
    """
    return length(vec3_sub(p, center)) - radius

def sdf_box(p, center, half_extents):
    """
    Returns signed distance from p to AABB.
    p: query point (vec3)
    center: box center (vec3)
    half_extents: half-size of box (vec3)
    """
    # Translate point to box local space
    p_local = vec3_sub(p, center)
    
    # Distance to surface
    d = vec3_abs(p_local) - half_extents
    
    # Calculate exact distance to corners/edges
    # max(d, 0) gives distance to the infinite box (positive if outside)
    # min(d, 0) gives distance to the interior (negative if inside)
    dist_outside = length(vec3_max(d, 0))
    dist_inside = vec3_min_component(vec3_min(d, 0))
    
    return dist_outside + dist_inside

def sdf_torus(p, major_radius, tube_radius):
    """
    Torus centered at origin, aligned with Y axis.
    """
    # Distance to the center ring of the torus
    q = (length((p[0], p[2])) - major_radius, p[1])
    return length(q) - tube_radius

def sdf_cylinder_capped(p, radius, half_height):
    """
    Cylinder aligned with Y axis.
    """
    dx = length((p[0], p[2])) - radius
    dy = abs(p[1]) - half_height
    
    # Distance to the side surface or caps
    dist_outside = length(vec3_max((dx, dy, 0), 0))
    dist_inside = vec3_min_component(vec3_min((dx, dy, 0), 0))
    
    return dist_outside + dist_inside

def sdf_union(d1, d2):
    """
    Union of two SDFs.
    """
    return min(d1, d2)

def sdf_intersection(d1, d2):
    """
    Intersection of two SDFs.
    """
    return max(d1, d2)

def sdf_subtraction(d1, d2):
    """
    Subtraction of d2 from d1.
    """
    return max(d1, -d2)

def sdf_smooth_union(d1, d2, k):
    """
    Smooth union using Quilez's approximation.
    """
    h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0)
    return mix(d2, d1, h) - k * h * (1.0 - h)

def clamp(val, min_val, max_val):
    return max(min_val, min(val, max_val))

def mix(a, b, t):
    return a + t * (b - a)
```

**Edge Cases and Pitfalls:**
*   **Non-Exact SDFs:** Not all shapes have closed-form SDFs. Arbitrary meshes require approximate SDFs (e.g., baked into 3D textures or BVH traversal) [S15, unofficial].
*   **Lipschitz Violation:** Non-uniform scaling or certain deformations can violate the Lipschitz condition, causing the SDF to no longer be a safe lower bound, leading to ray tunneling [S2, S11, unofficial].
*   **Gradient Noise:** The GDF (Generalized Distance Function) implementation with variable exponent `e` is slow and may not produce correct distances, though it allows for bulging [S13, unofficial].
*   **Driver Issues:** Loop unrolling issues on some hardware (e.g., Nvidia 350.12) can cause slowdowns for complex GDF primitives like `fIcosahedron` [S6, S13, unofficial].
*   **Sharp Discontinuities:** Taking the `min()` or `max()` of two SDFs yields sharp corners; smooth blending requires specific approximation functions like `smin` [S11, unofficial].

## Subtopics

### [SDF Primitives & Operations](1075-sdf-primitives-operations.md)
SDF Primitives & Operations provide the foundational analytical functions and geometric operators used to define scene geometry implicitly for distance-field ray marching, replacing explicit mesh tessellation with continuous scalar fields [G]. They solve the problem of representing complex 3D shapes and their compositions using closed-form algebraic solutions for canonical shapes (spheres, boxes, tori) and boolean/domain operators [G]. Unlike [SDF Transformation](1086-sdf-transformation.md) which modifies coordinates, or [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) which combines distinct shapes, this method focuses on the definition of base primitives and the immediate application of min/max operators for union/intersection.

### [SDF Transformation](1086-sdf-transformation.md)
SDF Transformation is a technique for applying geometric transformations (rotation, translation, scaling, and deformation) to Signed Distance Field primitives by manipulating the query point coordinates before evaluating the base distance function. It solves the problem of positioning and animating implicit surfaces in world space without modifying the underlying primitive definition, preserving the mathematical integrity of the distance field for ray marching. Unlike [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md), which handles exact distance compensation for uniform factors, or [Lipschitz Transformation Bounds](1557-lipschitz-transformation-bounds.md), which derives safety bounds for non-linear deformations, standard SDF Transformation relies on the inverse mapping of isometries (which preserve distance) or explicit scaling factors.

### [k-SDF](9530-k-sdf.md)
k-SDF is a representation for real-time view synthesis of fuzzy geometries using $k$ lightweight, semi-transparent mesh shells rendered via rasterization in a fixed order, bounding the number of sampling locations per ray to a small range ($3 \le k \le 9$). It models implicit geometry as $k$ distinct signed distance fields combined with a view-dependent transparency field and a color field. Unlike [3D Gaussian Splatting](52-3d-gaussian-splatting.md) or [SMERF](53-smurf.md), it is sorting-free and bounds samples per ray, enabling real-time rendering on low-cost smartphones. Unlike [AdaptiveShells](54-adaptive-shells.md), it learns multiple SDFs and finds sampling locations via rasterization using 2D textures rather than volume rendering.

### [SDF Representation DOP](12996-sdf-representation-dop.md)
The SDF Representation DOP creates a volumetric signed distance field (SDF) from geometry to enable fast inside/outside testing for collision detection. It differs from general volume rendering methods by focusing on the topological correctness of the distance field for rigid body dynamics rather than visual appearance, utilizing specific modes like Laser Scanning to handle non-watertight geometry.

### SDF Grid Modeling
A float grid where the active values describe the distance to a surface. Negative values are on the inside and positive on the outside, and the surface is where the values are zero [S16, unofficial]. Boolean operations on SDF grids allow for efficient boolean operations with smooth transitions, which are hard to perform on meshes. Smoothing is often applied using an SDF Grid Mean node [S16, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per primitive evaluation. The cost scales linearly with the number of primitives in the scene if evaluated naively, or logarithmically if accelerated with a BVH or grid structure.
*   **Memory Complexity:** $O(1)$ for analytical primitives (only parameters stored). $O(N)$ for grid-based SDFs where $N$ is the number of voxels.
*   **Measured Performance:** SDF ray marching is significantly faster than traditional ray-triangle intersection for scenes with high geometric complexity or procedural detail, as it avoids traversal overhead and leverages the distance bound for large steps [S1, S8, unofficial].
*   **Strengths:** Exact representation of smooth surfaces; easy composition via CSG; efficient rendering of complex procedural geometry; robust inside/outside testing.
*   **Limitations:** Limited to shapes with known analytical SDFs; sharp corners from boolean operations can cause aliasing or require smoothing; non-uniform scaling breaks the Lipschitz condition unless compensated.
*   **When to Use:** Real-time rendering of procedural scenes, ray marching, collision detection, and implicit modeling.
*   **When Not to Use:** Scenes requiring exact triangle-level detail (e.g., texturing with high-frequency UV maps), or when the geometry cannot be approximated by analytical functions.

## Background
The concept of Signed Distance Fields originates from computational geometry and level set methods, where they are used to track interfaces and represent orientable manifolds [S10, unofficial]. In computer graphics, SDFs were popularized for real-time rendering by Inigo Quilez's work on "Modeling with Distance Functions" and the subsequent adoption of sphere tracing (ray marching) techniques [S1, S7, S11, unofficial]. They have become a standard tool in procedural generation, ray marching shaders, and implicit modeling due to their efficiency and composability [S1, S8, unofficial].

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
[S16] Volume Grids in Geometry Nodes — Blender Developers Blog, https://code.blender.org/2025/10/volume-grids-in-geometry-nodes/, fetched 2026-10-27 (unofficial)
