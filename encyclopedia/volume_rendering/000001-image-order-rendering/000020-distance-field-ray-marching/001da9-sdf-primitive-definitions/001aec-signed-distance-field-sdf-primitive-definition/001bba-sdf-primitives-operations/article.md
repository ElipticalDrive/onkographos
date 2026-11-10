# SDF Primitives & Operations

Also called: Implicit Shape Functions, Signed Distance Function (SDF) Primitives and Operators, Signed Distance Function Primitives

## Overview
SDF Primitives & Operations provide the foundational analytical functions and Boolean/combinatorial logic required to define complex geometry for distance-field ray marching [S1, S6, S15, unofficial]. They solve the problem of representing explicit mesh tessellation with continuous scalar fields, allowing for efficient ray marching and procedural composition of shapes [G]. What sets this method apart from its siblings is the distinction between **domain manipulation** (transforming the input coordinate $\mathbf{p}$ before evaluation, as in [SDF Transformation](1086-sdf-transformation.md)) and **value-domain operations** (combining the output distance values $d$ of primitives, as described here) [S1, S3, S15, unofficial]. Unlike [k-SDF](9530-k-sdf.md), which approximates distances for specific optimization purposes, these primitives provide exact or analytically derived distance metrics for canonical shapes and their combinations [S1, S6, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** A query point $\mathbf{p} \in \mathbb{R}^3$ (and potentially additional parameters like radius, normal, or transformation matrices).
*   **Output:** A scalar signed distance $d \in \mathbb{R}$, where $d < 0$ indicates the point is inside the shape, $d = 0$ is on the surface, and $d > 0$ is outside [G].

### Data Structures
The primitives are typically implemented as functions or structs that encapsulate parameters. Common structures include:
*   **Primitive Struct:** Contains parameters (e.g., `radius`, `height`, `position`) and a method `dist(p)`.
*   **Composite Struct:** Contains references to child SDFs (e.g., `sdf1`, `sdf2`) and an operation type (e.g., `union`, `intersection`) [S1, unofficial].
*   **Material Info:** Often paired with distance in a `DistInfo` struct containing `distance` (f32) and `material` (enum/struct) [S1, unofficial].

### Basic Primitive Definitions
The following formulas compute the exact Euclidean signed distance for canonical shapes [S1, S5, S6, S8, unofficial]. All vectors are 3D unless noted.

1.  **Sphere:** Centered at origin, radius $r$.
    $$d(\mathbf{p}) = \|\mathbf{p}\| - r$$
    [S1, S3, unofficial]

2.  **Box (AABB):** Half-extents $\mathbf{b} = (b_x, b_y, b_z)$.
    $$\mathbf{q} = |\mathbf{p}| - \mathbf{b}$$
    $$d(\mathbf{p}) = \|\max(\mathbf{q}, 0)\| + \min(\max(q_x, \max(q_y, q_z)), 0)$$
    [S1, S3, unofficial]

3.  **Torus:** Major radius $R$, minor radius $r$. Centered at origin, aligned with Y-axis.
    $$d(\mathbf{p}) = \|\vec{2}(\|\mathbf{p}_{xy}\| - R, p_z)\| - r$$
    Where $\mathbf{p}_{xy} = (p_x, p_y)$ and $\vec{2}(u, v) = (u, v)$ [S1, unofficial].

4.  **Plane:** Defined by normal $\mathbf{n}$ (unit vector) and distance $h$ from origin along normal.
    $$d(\mathbf{p}) = \mathbf{p} \cdot \mathbf{n} - h$$
    [S1, S7, unofficial]

5.  **Cylinder:** Radius $r$, height $h$, aligned with Y-axis, centered at origin.
    $$\mathbf{q} = \vec{2}(\|\mathbf{p}_{xz}\| - r, |p_y| - h/2)$$
    $$d(\mathbf{p}) = \|\max(\mathbf{q}, 0)\| + \min(\max(q_x, q_y), 0)$$
    [S6, G, unofficial]

6.  **Cone:** Apex at origin, axis along $\mathbf{c}$ (unit vector), height $h$, base radius $r$.
    Let $\mathbf{p}'$ be $\mathbf{p}$ projected onto the cone axis.
    $$d(\mathbf{p}) = \text{distance to segment between apex and base center}$$
    [S2, S6, unofficial]

7.  **Ellipsoid:** Radii $\mathbf{r} = (r_x, r_y, r_z)$.
    $$d(\mathbf{p}) = \|\mathbf{p} \oslash \mathbf{r}\| - 1$$
    Where $\oslash$ is component-wise division. Note: This is an approximation for general ellipsoids, exact SDFs are complex [S6, G, unofficial].

8.  **Pyramid:** Height $h$, base size $s$.
    Complex analytical formula involving projection onto base and side planes [S5, unofficial].

9.  **Octahedron:** Size $s$.
    $$d(\mathbf{p}) = \frac{|p_x| + |p_y| + |p_z| - s}{\sqrt{3}}$$
    [S5, G, unofficial]

### Boolean and Combinatorial Operations
These operations combine the distance values of two SDFs, $f_A(\mathbf{p})$ and $f_B(\mathbf{p})$ [S1, S7, S8, unofficial].

1.  **Union (A $\cup$ B):** The point is inside if it is inside A OR inside B. The distance is the minimum of the two.
    $$d_{union}(\mathbf{p}) = \min(f_A(\mathbf{p}), f_B(\mathbf{p}))$$
    [S1, S3, S7, S8, unofficial]

2.  **Intersection (A $\cap$ B):** The point is inside if it is inside A AND inside B. The distance is the maximum of the two.
    $$d_{intersect}(\mathbf{p}) = \max(f_A(\mathbf{p}), f_B(\mathbf{p}))$$
    [S1, S3, S7, S8, unofficial]

3.  **Difference (A - B):** The point is inside A but NOT inside B. This is equivalent to $A \cap (\mathbb{R}^3 \setminus B)$. The distance to the complement of B is $-f_B(\mathbf{p})$.
    $$d_{diff}(\mathbf{p}) = \max(f_A(\mathbf{p}), -f_B(\mathbf{p}))$$
    [S1, S3, S7, S8, unofficial]

4.  **Smooth Union:** To avoid sharp discontinuities from min/max, a smooth blend can be used. A common approximation (Quilez) with blend radius $k$:
    $$a = f_A(\mathbf{p}), b = f_B(\mathbf{p})$$
    $$h = \max(k - \max(a, b), 0.0) / k$$
    $$d_{smooth} = \min(a, b) - h \cdot h \cdot k \cdot 0.25$$
    [S8, S14, unofficial]

### Domain Manipulation (Transformations)
Transformations modify the input coordinate $\mathbf{p}$ before evaluation [S1, S3, S15, unofficial].

1.  **Translation:** Offset by $\mathbf{t}$.
    $$d(\mathbf{p}) = f(\mathbf{p} - \mathbf{t})$$
    [S1, S3, unofficial]

2.  **Rotation:** Rotate by quaternion $\mathbf{q}$ or matrix $M$.
    $$d(\mathbf{p}) = f(\mathbf{q} \times \mathbf{p}) \quad \text{or} \quad f(M^{-1}\mathbf{p})$$
    [S1, S3, unofficial]

3.  **Scaling:** Scale by factor $s$.
    $$d(\mathbf{p}) = f(\mathbf{p}/s) / s$$
    [G]

4.  **Repetition (Tiling):** Fold space into a fundamental domain.
    $$\mathbf{p}' = \mathbf{p} - \text{period} \cdot \lfloor \mathbf{p} / \text{period} + 0.5 \rfloor$$
    $$d(\mathbf{p}) = f(\mathbf{p}')$$
    [S1, unofficial]

### Specialized Primitives

#### Solid Noise Primitive
A sphere-based primitive that windows a noise function [S9, unofficial].
*   **Formula:**
    $$\text{noiseDensity}(\mathbf{P}) = m(\mathbf{P}) + (1 - \|\mathbf{P}/r\|)$$
    Where $\mathbf{P}$ is in local space, $r$ is radius, and $m(\mathbf{P})$ is the fractal noise function (e.g., fbm) [S9, unofficial].
*   **Pitfall:** The fractal function $m(\mathbf{P})$ can be positive outside the sphere's radius. If the maximal amplitude of the fractal is $A$, the function has non-negative values up to $A$ units away from the radius. Bounds padding is required [S9, unofficial].

#### Inverted Torus (Tube Cavity)
Used to render the interior of a torus as empty space, enabling front-to-back occlusion and eliminating donut-hole artifacts [S10, unofficial].
*   **Formula:**
    $$d(\mathbf{p}) = -(d_{ring}(\mathbf{p}) - r)$$
    Where $d_{ring}$ is the standard torus SDF. Negative values (inside the tube wall) are treated as empty space, positive values (donut hole, outside) as solid [S10, unofficial].

#### Butterfly SDF
Composed of `sdWing`, `sdBody`, and `sdAntenna` [S11, unofficial].
*   **Wing:** Un-bends point to wing space, checks outline (rounded triangles), calculates thickness and camber [S11, unofficial].
*   **Body:** Parametric radius function based on normalized Z coordinate [S11, unofficial].
*   **Antenna:** Quadratic bezier-like curve spine with varying radius [S11, unofficial].
*   **Combination:**
    ```python
    def map_butterfly(p, t, phase):
        m = vec3(abs(p.x), p.y, p.z)
        wing = sd_wing(m, t, phase)
        body = min(sd_body(p), sd_antenna(p))
        if wing < body:
            return vec2(wing, 0.0) # 0 = wing material
        else:
            return vec2(body, 1.0) # 1 = body material
    ```
    [S11, unofficial]

#### Procedural Noise SDF
Perlin or Simplex noise can be treated as an SDF because it returns a real number at every point and is locally smooth [S12, unofficial].
*   **Heightmap:** $z = \text{Perlin}(x, y, 0)$. Surface is where $z - \text{Perlin}(x, y, 0) = 0$ [S12, unofficial].
*   **3D Noise:** For overhangs/caves, use 3D noise. Subtract from surface for caves, add for overhangs. Requires fall-off values to reduce computation area [S12, unofficial].

#### Backface Distance Field (BDF)
Improves sphere tracing convergence by using closest signed distances to backfacing surface points [S13, abstract only, unofficial].
*   **External Points:** Step size is determined by backface distance, which is the largest direction-independent step that does not pass through the volume boundary more than once [S13, abstract only, unofficial].
*   **Internal Points:** Coincides with standard SDF [S13, abstract only, unofficial].
*   **Performance:** Generally outperforms basic, relaxed, and enhanced sphere tracing [S13, abstract only, unofficial].

#### Smooth Minimum Function (Dreams)
Used for blending primitive shapes over a fixed radius [S14, unofficial].
*   **Difference from Standard:** Standard smooth min has infinite range. Dreams uses a modified formula blending over a fixed radius [S14, unofficial].

#### KIFS Recursive Fractal
Uses KIFS schema with smooth-min and smooth-abs operators to blend spheres into organic shapes [S15, unofficial].

### Pseudocode: Ray Marching with SDF Primitives
This pseudocode integrates primitive evaluation, operations, and the ray marching loop [S1, S3, S8, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def length(v):
    return math.sqrt(v.x**2 + v.y**2 + v.z**2)

def max_vec(v):
    return max(v.x, max(v.y, v.z))

def min_vec(v):
    return min(v.x, min(v.y, v.z))

def vec3(x, y, z):
    return {'x': x, 'y': y, 'z': z}

def abs_vec(v):
    return vec3(abs(v['x']), abs(v['y']), abs(v['z']))

def add_vec(a, b):
    return vec3(a['x'] + b['x'], a['y'] + b['y'], a['z'] + b['z'])

def sub_vec(a, b):
    return vec3(a['x'] - b['x'], a['y'] - b['y'], a['z'] - b['z'])

def scale_vec(v, s):
    return vec3(v['x'] * s, v['y'] * s, v['z'] * s)

def dot_vec(a, b):
    return a['x']*b['x'] + a['y']*b['y'] + a['z']*b['z']

def normalize(v):
    l = length(v)
    if l == 0: return vec3(0,0,0)
    return vec3(v['x']/l, v['y']/l, v['z']/l)

def max_scalar(a, b):
    return a if a > b else b

def min_scalar(a, b):
    return a if a < b else b

def clamp(v, min_val, max_val):
    return max(min_val, min(max_val, v))

# --- Primitives ---

def sphere(p, r):
    return length(p) - r

def box(p, b):
    q = abs_vec(sub_vec(p, vec3(0,0,0))) # Assuming box centered at 0, b is half-extent
    # If box is not centered at 0, p should be transformed first
    q = abs_vec(p)
    q_max = vec3(max(q['x'], 0), max(q['y'], 0), max(q['z'], 0))
    q_min = vec3(min(q['x'], max(q['y'], q['z'])), 0, 0) # Simplified min logic for scalar
    # Correct formula from [S3, unofficial]:
    # q = abs(p) - b
    # qMax = max(q, 0)
    # return length(qMax) + min(max(q.x, max(q.y, q.z)), 0)
    q = vec3(max(q['x'] - b['x'], 0), max(q['y'] - b['y'], 0), max(q['z'] - b['z'], 0))
    inside = min_scalar(max(q['x'] - b['x'], max(q['y'] - b['y'], q['z'] - b['z'])), 0)
    return length(q) + inside

def torus(p, R, r):
    # R = major radius, r = minor radius
    # dist(p) = length(vec2(length(p.xy) - R, p.z)) - r
    px = p['x']
    py = p['y']
    pz = p['z']
    dist_ring = math.sqrt((math.sqrt(px**2 + py**2) - R)**2 + pz**2)
    return dist_ring - r

def plane(p, n, h):
    # n is normal, h is distance from origin
    return dot_vec(p, n) - h

# --- Operations ---

def union_sdf(p, sdf1, sdf2):
    return min_scalar(sdf1(p), sdf2(p))

def intersection_sdf(p, sdf1, sdf2):
    return max_scalar(sdf1(p), sdf2(p))

def difference_sdf(p, sdf1, sdf2):
    return max_scalar(sdf1(p), -sdf2(p))

def translate_sdf(sdf, offset):
    def wrapped(p):
        return sdf(sub_vec(p, offset))
    return wrapped

def rotate_sdf_y(sdf, angle):
    cos_a = math.cos(angle)
    sin_a = math.sin(angle)
    def wrapped(p):
        # Rotate around Y axis
        x = p['x'] * cos_a + p['z'] * sin_a
        z = -p['x'] * sin_a + p['z'] * cos_a
        return sdf(vec3(x, p['y'], z))
    return wrapped

# --- Ray Marching ---

def cast_ray(origin, direction, sdf_func, max_dist=30.0, max_steps=1000):
    p = origin
    for i in range(max_steps):
        d = sdf_func(p)
        if d < 0.001: # Surface hit
            return True, p, i
        if d > max_dist: # Missed
            return False, p, i
        p = add_vec(p, scale_vec(direction, d))
    return False, p, max_steps

def compute_normal(p, sdf_func, eps=0.001):
    # Central difference
    dx = sdf_func(vec3(p['x'] + eps, p['y'], p['z'])) - sdf_func(vec3(p['x'] - eps, p['y'], p['z']))
    dy = sdf_func(vec3(p['x'], p['y'] + eps, p['z'])) - sdf_func(vec3(p['x'], p['y'] - eps, p['z']))
    dz = sdf_func(vec3(p['x'], p['y'], p['z'] + eps)) - sdf_func(vec3(p['x'], p['y'], p['z'] - eps))
    return normalize(vec3(dx, dy, dz))
```

### Edge Cases and Pitfalls
1.  **Discontinuities:** Using `min()` or `max()` directly creates sharp edges. For smooth blending, use smooth union/difference [S8, S14, unofficial].
2.  **Division by Zero:** In Mandelbulb or polar-based SDFs, avoid division by zero when $r=0$ or $z=0$ (e.g., in `atan`) [S1, unofficial].
3.  **Precision:** Floating-point precision issues can cause artifacts in ray marching near the surface. Use a small epsilon for normal computation and surface detection [S3, unofficial].
4.  **Bounds Padding:** For noise-based primitives (Solid Noise), the fractal function can extend the effective bounds beyond the geometric primitive. Padding is required for correct culling [S9, unofficial].
5.  **Inverted SDFs:** Negating an SDF (e.g., for torus tube interior) changes the interpretation of inside/outside. Ensure ray marching logic handles this correctly (e.g., treating negative as empty) [S10, unofficial].

## Subtopics

### [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md)
Modifies input position $\mathbf{p}$ to create periodic, mirrored, or stretched structures. Acts on spatial coordinates rather than distance output.

### [Quadric SDF (First-Order Approximation)](2682-quadric-sdf-first-order-approximation.md)
Approximates distance to quadric surfaces using first-order Taylor expansion, trading precision for speed.

### [Repeating SDF Geometry](4713-repeating-sdf-geometry.md)
Creates periodic tiling by folding 3D coordinates into a fundamental domain.

### [Basic Primitive Definitions](5346-basic-primitive-definitions.md)
Foundational analytical functions for canonical shapes (spheres, boxes, tori).

### [Higher Order Algebraic Signed Distance Fields](8500-higher-order-algebraic-signed-distance-fields.md)
Generalizes SDFs using local Taylor approximations or least squares fit polynomials for higher accuracy.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per primitive evaluation. Ray marching complexity depends on the number of steps, which is proportional to the distance to the surface divided by the minimum step size (SDF value) [S8, unofficial].
*   **Memory Complexity:** $O(1)$ for procedural primitives. No mesh storage required [G].
*   **Performance:** Highly efficient for simple geometry and boolean combinations. Performance degrades with complex fractals (e.g., Mandelbulb) or deep nesting of operations [S1, S15, unofficial].
*   **Strengths:** Infinite resolution, easy boolean operations, compact representation, smooth shading via normals [S1, S6, unofficial].
*   **Limitations:** Sharp edges from min/max operations, potential for artifacts in complex boolean combinations, difficulty in representing highly detailed organic geometry without noise [S8, S12, unofficial].
*   **When to Use:** Procedural geometry, soft shadows, ambient occlusion, smooth blending [S1, S15, unofficial].
*   **When Not to Use:** Highly detailed static geometry where mesh is more efficient, or when exact geometric precision is critical and approximations (like quadric SDFs) are used [S1, S2682].

## Background
SDFs were popularized by Inigo Quilez in 2007, who demonstrated techniques for efficient soft shadows, smooth blending, and domain repetition [S15, unofficial]. The concept of using distance functions for ray marching has roots in earlier work on implicit surfaces [S7, unofficial]. The method has become a staple in real-time rendering for its ability to create complex geometry with minimal code and memory overhead [S1, S15, unofficial].

## Sources
[S1] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
[S2] agill-dev-0/SDF-ray-marcher, https://github.com/agill-dev-0/SDF-ray-marcher, fetched 2026-10-26 (unofficial)
[S3] HeartofTheForce/Isosurface, https://github.com/HeartofTheForce/Isosurface, fetched 2026-10-26 (unofficial)
[S4] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S5] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S6] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S7] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S8] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S9] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S10] Add OpenGL GPU ray marching renderers for Intel HD Graphics 630 with Jump Flood Algorithm optimization, https://github.com/rhythmatician/phyllotaxis-torus/pull/1, fetched 2026-10-26 (unofficial)
[S11] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S12] Implicit Functions and extracting an isosurface, https://gamedev.stackexchange.com/questions/85873/implicit-functions-and-extracting-an-isosurface, fetched 2026-10-26 (unofficial)
[S13] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
[S14] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S15] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
