# SDF Primitives & Operations

Also called: Basic Shapes, Boolean operations on implicit surfaces, CSG Intersection, Implicit Function CSG, SDF Operations, Subtraction, Union

## Overview
SDF Primitives & Operations define the fundamental geometric building blocks and Boolean compositing rules for Signed Distance Fields (SDFs) [S3, S4, unofficial]. This method solves the problem of representing complex 3D geometry as a single continuous scalar field by combining simple analytic shapes (primitives) using arithmetic operators [S4, unofficial]. It is distinguished from mesh-based rendering by replacing explicit topology with implicit distance evaluations, allowing complex scenes to be constructed via simple arithmetic on scalar values [S4, unofficial]. Unlike sibling [SDF Transformation](1086-sdf-transformation.md) which modifies the input coordinate space, this method operates on the output distance values to define the final surface [G].

## How it works

### Inputs and Outputs
**Inputs:**
- Point $\mathbf{p}$ in 3D space [S3, unofficial].
- Primitive parameters (e.g., radius $r$, box size $\mathbf{b}$, normal $\mathbf{n}$) [S3, unofficial].
- For composite operations: two or more SDF functions $f_A, f_B$ and their respective parameters [S4, unofficial].

**Outputs:**
- A scalar float representing the signed distance from $\mathbf{p}$ to the nearest surface [S3, unofficial].
- Positive values indicate outside, zero on the surface, negative inside [S3, unofficial].

### Data Structures
No complex data structures are required for basic primitives [G]. The state is defined by the function parameters [G]. For CSG operations, the scene graph is typically a tree of function pointers or lambda expressions evaluating child SDFs [G].

### Primitives
The following analytic primitives are defined by [S3, unofficial] and [S4, unofficial]. All vectors are 3D.

1.  **Sphere**: Centered at origin, radius $r$ [S3, unofficial].
    $$f_{sphere}(\mathbf{p}) = \|\mathbf{p}\| - r$$
2.  **Box**: Axis-aligned, centered at origin, half-extents $\mathbf{b} = (b_x, b_y, b_z)$ where $b_i > 0$ [S3, unofficial].
    $$q = |\mathbf{p}| - \mathbf{b}$$
    $$q_{max} = \max(q, 0)$$
    $$f_{box}(\mathbf{p}) = \|q_{max}\| + \min(\max(q_x, \max(q_y, q_z)), 0)$$
3.  **Round Box**: Box with rounded edges of radius $r_{round}$ [S3, unofficial].
    $$f_{roundbox}(\mathbf{p}) = f_{box}(\mathbf{p}) - r_{round}$$
4.  **Cylinder**: Axis-aligned along Y, height $h$, radius $r$ [S3, unofficial].
    $$d = \sqrt{p_x^2 + p_z^2} - r$$
    $$h_{half} = h / 2$$
    $$f_{cyl}(\mathbf{p}) = \max(\max(d, |p_y| - h_{half}), 0)$$
5.  **Round Cylinder**: Cylinder with rounded edges [S3, unofficial].
    $$f_{roundcyl}(\mathbf{p}) = f_{cyl}(\mathbf{p}) - r_{round}$$
6.  **Plane**: Defined by unit normal $\mathbf{n}$ and distance $h$ from origin along $\mathbf{n}$ [S4, unofficial].
    $$f_{plane}(\mathbf{p}) = \mathbf{p} \cdot \mathbf{n} - h$$
7.  **Torus**: Centered at origin, major radius $R$, minor radius $r$ [S3, unofficial].
    $$q = \sqrt{p_x^2 + p_z^2} - R$$
    $$f_{torus}(\mathbf{p}) = \sqrt{q^2 + p_y^2} - r$$
8.  **Ellipsoid**: Axis-aligned, radii $\mathbf{r} = (r_x, r_y, r_z)$ [S3, unofficial].
    $$\mathbf{p}' = \mathbf{p} / \mathbf{r}$$
    $$f_{ellipsoid}(\mathbf{p}) = \|\mathbf{p}'\| - 1$$
9.  **Cone**: Axis-aligned along Y, height $h$, radius $r$ at base [S3, unofficial].
    $$d = \sqrt{p_x^2 + p_z^2}$$
    $$h_{half} = h / 2$$
    $$f_{cone}(\mathbf{p}) = \max(\max(d, |p_y| - h_{half}), 0)$$
10. **Pyramid**: Height $h$ [S3, unofficial].
    *Note: Exact formula not provided in sources [G].*
11. **Link**: Half of a chain-link [S3, unofficial].
    *Note: Parametrized by position, major/minor radii, and link length [S3, unofficial]. Exact formula not provided [G].*
12. **Elongated Torus**: Torus extended in x, y, z [S3, unofficial].
    *Note: Exact formula not provided [G].*

### CSG Operations
Boolean operations combine SDFs $f_A$ and $f_B$ to create new surfaces [S1, S4, unofficial].

1.  **Union (A $\cup$ B)**: The surface is the boundary of the combined volume.
    $$f_{union}(\mathbf{p}) = \min(f_A(\mathbf{p}), f_B(\mathbf{p}))$$
    This is the most common operation for combining disjoint or overlapping objects [S4, unofficial].
2.  **Intersection (A $\cap$ B)**: The surface is the boundary of the overlapping volume.
    $$f_{intersect}(\mathbf{p}) = \max(f_A(\mathbf{p}), f_B(\mathbf{p}))$$
    Source [S4, unofficial] states $d(\mathbf{x}, A \cap B) \geq \max(f_A, f_B)$ [S4, unofficial].
3.  **Subtraction (A \ B)**: The surface is A minus B.
    $$f_{sub}(\mathbf{p}) = \max(f_A(\mathbf{p}), -f_B(\mathbf{p}))$$

### Normal Computation
Normals are required for lighting and are computed via central differences [S1, unofficial].
$$\mathbf{n}(\mathbf{p}) = \text{normalize}\left( \begin{bmatrix} f(\mathbf{p} + \epsilon \mathbf{x}) - f(\mathbf{p} - \epsilon \mathbf{x}) \\ f(\mathbf{p} + \epsilon \mathbf{y}) - f(\mathbf{p} - \epsilon \mathbf{y}) \\ f(\mathbf{p} + \epsilon \mathbf{z}) - f(\mathbf{p} - \epsilon \mathbf{z}) \end{bmatrix} \right)$$
Where $\epsilon$ is a small epsilon value [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def max_vec(a, b):
    return [max(a[0], b[0]), max(a[1], b[1]), max(a[2], b[2])]

def min_vec(a, b):
    return [min(a[0], b[0]), min(a[1], b[1]), min(a[2], b[2])]

def abs_vec(v):
    return [abs(v[0]), abs(v[1]), abs(v[2])]

def sphere(p, r):
    return length(p) - r

def box(p, b):
    q = abs_vec(p) - b
    q_max = max_vec(q, [0, 0, 0])
    return length(q_max) + min(max(q[0], max(q[1], q[2])), 0)

def union(a_func, a_params, b_func, b_params, p):
    return min(a_func(p, a_params), b_func(p, b_params))

def intersection(a_func, a_params, b_func, b_params, p):
    return max(a_func(p, a_params), b_func(p, b_params))

def subtraction(a_func, a_params, b_func, b_params, p):
    return max(a_func(p, a_params), -b_func(p, b_params))

def plane(p, n, h):
    return sum(p[i] * n[i] for i in range(3)) - h

def normal_at(func, params, p, eps=1e-4):
    dx = func([p[0]+eps, p[1], p[2]], params) - func([p[0]-eps, p[1], p[2]], params)
    dy = func([p[0], p[1]+eps, p[2]], params) - func([p[0], p[1]-eps, p[2]], params)
    dz = func([p[0], p[1], p[2]+eps], params) - func([p[0], p[1], p[2]-eps], params)
    vec = [dx, dy, dz]
    l = length(vec)
    if l == 0: return [0, 1, 0] # Fallback
    return [vec[0]/l, vec[1]/l, vec[2]/l]
```

### Parameters
- **Epsilon**: For normal calculation, a small value [S1, unofficial].
- **Radius/Size**: Primitive-specific [S3, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Near the surface ($f(\mathbf{p}) \approx 0$), floating-point errors can cause jitter in ray marching steps [G].
- **Subtraction Artifacts**: Subtracting a large object from a small one can result in inverted normals if not handled correctly by the CSG logic [G].
- **Smooth Union**: Not covered in basic operations, but standard min/max creates sharp edges. Smooth unions require additional blending functions [S3, unofficial].

## Subtopics

### [Signed Box Distance Function](1092-signed-box-distance-function.md)
Operators: $|\cdot|$ is absolute value, $\|\cdot\|$ is length of vector, $\max/\min$ are maximum/minimum operators. $\max(d_i, 0)$ applies the maximum operator to each component of vector $d_i$ with 0. This primitive is the basis for axis-aligned bounding boxes and is computationally efficient due to the absence of square roots in the intermediate steps [1092-signed-box-distance-function.md].

### [Sphere Signed Distance Function](1237-sphere-signed-distance-function.md)
A signed distance function for a sphere centered at the origin, returning the distance from a point to the sphere surface. It is the simplest analytic primitive and serves as a building block for more complex shapes like ellipsoids and solid noise [1237-sphere-signed-distance-function.md].

### [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md)
Constructive Solid Geometry (CSG) on implicit functions composites multiple primitive shapes into a single implicit surface by applying Boolean logic to their distance fields. This method allows the rendering of complex geometries composed of multiple primitives using a single field evaluation, which is essential for ray marching and bisection tracers. Unlike explicit mesh operations, CSG on implicit functions reduces geometric composition to arithmetic operations on scalar values, enabling normals and shadows derived from the resulting field's gradient. It is distinguished from sibling primitives by its role as a compositing layer that combines the outputs of [Sphere Signed Distance Function](1237-sphere-signed-distance-function.md), [Signed Box Distance Function](1092-signed-box-distance-function.md), and other primitives into a unified scene representation [G].

### [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md)
A set of functions that modify the input position vector `p` in place to create periodic, mirrored, or reflected spatial structures for Signed Distance Functions (SDFs). They partition space into cells and return a cell index for optional use as a random seed. These operators allow for the creation of infinite repeating patterns (e.g., grids, tiles) without increasing geometric complexity [1730-sdf-domain-manipulation-operators.md].

### [SDF Primitives](1731-sdf-primitives.md)
Basic Signed Distance Functions for common geometric shapes: Sphere, Box, Torus. These are the atomic elements from which all other SDF scenes are constructed [1731-sdf-primitives.md].

### [SDF Rounding](1829-sdf-rounding.md)
Rounding an SDF by subtracting a constant distance value from the SDF result. This operation effectively thickens or thins the surface uniformly and is used to create rounded edges on boxes or to adjust collision boundaries [1829-sdf-rounding.md].

### [Plane Signed Distance Function](1102-plane-signed-distance-function.md)
**Definition**: Signed distance function for a plane P defined by unit normal $\mathbf{n}$ and height $h$.
**Formula**: $f(\mathbf{x}) = \mathbf{x} \cdot \mathbf{n} - h$, where $\mathbf{n}$ is the unit normal and $h$ defines the relative placement of the plane.
**Parameter**: $h$: Defines the relative placement of the plane with respect to the plane perpendicular to normal $\mathbf{n}$ intersecting the origin.

### Boolean Subtraction with Domain Repetition
**Definition**: Creating complex geometry by subtracting domain-repeated smaller objects from a non-repeated larger object (e.g., domain repeated cubes subtracted from a larger sphere) [S5, unofficial].
**Step**: Combine domain repetition with subtracting distance fields from each other to create complex objects [S5, unofficial].

### Solid Noise Primitive
**Definition**: A sphere-based primitive that uses the location and radius to 'window' a noise function, where density is the sum of a windowing function and a fractal function [S6, unofficial].
**Formula**: $noiseDensity(P) = m(P) + (1 - |P/radius|)$, where $P$ is in the local space of the primitive and $m(P)$ is the fractal function [S6, unofficial].
**Parameter**: $A$: The maximal amplitude of the fractal function fbm [S6, unofficial].
**Code**: Implementation of `getSample` for `SolidNoisePoint`: transforms to local space, evaluates fractal function with gamma and amplitude, computes distance function, and returns $\max(0, distance + fractal) \times density$ [S6, unofficial].
**Pitfall**: Due to the fractal function, density may be positive outside the sphere's radius. The function has non-negative values at most $A$ units away from the radius, requiring bounds padding [S6, unofficial].

## Operating characteristics

**Time Complexity**: $O(1)$ per primitive evaluation. CSG operations add a constant factor proportional to the number of children in the scene graph [G].
**Memory Complexity**: $O(1)$ for primitives. Scene graph size depends on the number of primitives [G].
**Measured Performance**: Highly efficient on GPUs due to lack of branching and simple arithmetic [G].
**Strengths**:
- Infinite resolution (analytic) [G].
- Easy to implement and debug [G].
- Compact memory footprint [G].
- Natural support for smooth blending and morphing [S3, unofficial].
**Limitations**:
- Cannot represent arbitrary topology (e.g., self-intersections in complex ways without CSG) [G].
- Normals are approximate if computed via finite differences [G].
**When to use**: Ideal for real-time rendering of stylized, geometric, or procedural scenes [G].
**When not to use**: When photorealistic detail from scanned meshes is required, or when the geometry is too complex for efficient CSG evaluation [G].

## Background
The concept of Signed Distance Fields for rendering was popularized by Inigo Quilez in his blog posts and books on ray marching [S1, S3, unofficial]. The mathematical foundations of CSG with distance fields were established by Hart [S4, unofficial]. The technique has become a staple in real-time graphics for creating procedural geometry, VFX, and stylized rendering [G].

## Sources
[S1] HeartofTheForce/Isosurface, https://github.com/HeartofTheForce/Isosurface, fetched 2026-10-26 (unofficial)
[S2] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S3] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S4] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S5] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S6] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
