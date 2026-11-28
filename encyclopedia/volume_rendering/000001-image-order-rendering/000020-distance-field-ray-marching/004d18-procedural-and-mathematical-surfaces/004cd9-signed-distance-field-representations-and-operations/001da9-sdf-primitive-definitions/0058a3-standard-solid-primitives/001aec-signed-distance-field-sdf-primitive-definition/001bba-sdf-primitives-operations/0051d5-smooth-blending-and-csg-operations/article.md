# Smooth Blending and CSG Operations

Also called: Smooth Minimum, SDF Boolean Operations, Constructive Solid Geometry with SDFs

## Overview
This family defines the mathematical operators and coordinate transformations required to construct complex procedural geometry from primitive Signed Distance Fields (SDFs). It solves the problem of combining simple shapes into complex models and positioning them in world space. It is distinguished from exact Boolean operations by its support for smooth blending approximations, which allow for continuous gradients and organic transitions between shapes, at the cost of distance field accuracy [S2].

## Shared principles
The core mechanism involves evaluating primitive SDF functions $S_i(\mathbf{p})$ at query points $\mathbf{x}$, applying coordinate transformations, and combining the resulting scalar distances using specific binary operators.

**Coordinate Transformation:**
To evaluate a primitive in world space, the query point $\mathbf{x}$ is transformed into the primitive's local space. This involves:
1.  Subtracting translation $\mathbf{t}$: $\mathbf{p} = \mathbf{x} - \mathbf{t}$.
2.  Applying rotation $\mathbf{R}$: $\mathbf{p} = \mathbf{R}^T \mathbf{p}$ (assuming $\mathbf{R}$ is orthonormal).
3.  Applying scale $s$: $\mathbf{p} = \mathbf{p} / s$.
The resulting distance $d_i$ must be divided by the scale factor $|s|$ to maintain correct distance metric properties [S2, G].

**Domain Repetition (Tiling):**
For periodic structures, a modulo operation is applied to local coordinates before evaluation [S2]:
$$ \mathbf{p}_{mod} = \mathbf{p} - \lfloor \mathbf{p} / \mathbf{period} + 0.5 \rfloor \cdot \mathbf{period} $$

**Exact Boolean Operations:**
These operators combine distances $d_1$ and $d_2$ to form unions, intersections, and subtractions:
*   **Union:** $d_{union} = \min(d_1, d_2)$. This returns the distance to the closest shape exterior edge.
*   **Intersection:** $d_{intersect} = \max(d_1, d_2)$.
*   **Subtraction:** $d_{sub} = \max(d_1, -d_2)$ (subtracting shape 2 from shape 1) [S2].

**Smooth Blending:**
Approximate unions/subtractions are achieved with a transition controlled by parameter $k$. These are no longer exact distance fields but distorted approximations [S2]. Two primary formulations are used:

1.  **Exponential Smooth Minimum:**
    $$ \text{smin}(a, b, k) = -k \cdot \log_2(2^{-a/k} + 2^{-b/k}) $$
    Alternatively using natural base $e$:
    $$ \text{smin}(a, b, k) = -k \cdot \ln(e^{-a/k} + e^{-b/k}) $$

2.  **Polynomial (Hermite) Smooth Minimum:**
    $$ h = \text{clamp}\left(\frac{0.5 + 0.5(b - a)}{k}, 0.0, 1.0\right) $$
    $$ \text{smin}(a, b, k) = \text{lerp}(b, a, h) - k \cdot h \cdot (1.0 - h) $$
    Where $\text{lerp}(b, a, h) = (1.0 - h)a + hb$ [S2, G].

**Data Structures:**
Operations are often stored in a tree structure within GPU buffers. The specific layout for an SDF operation block is defined as :

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

## Subtopics

### [SDF Transformations and Operations](12454-sdf-transformations-and-operations.md)
This method defines the mathematical operators and coordinate transformations required to construct complex procedural geometry from primitive Signed Distance Fields (SDFs). It solves the problem of combining simple shapes into complex models and positioning them in world space. It is distinguished from exact Boolean operations by its support for smooth blending approximations, which allow for continuous gradients and organic transitions between shapes, at the cost of distance field accuracy.

### Smooth Minimum Function
#### definition
- [S1, unofficial] A function used in Dreams to blend primitive shapes over a fixed radius, allowing for smoothed addition and subtraction in CSG operations.
#### formula
- [S1, unofficial] The specific formula is not provided in the text, but it is noted that the standard smooth minimum has an infinite range (reaching zero at infinity). Dreams uses a modified formula that blends over a fixed radius.
#### parameter
- [S1, unofficial] Blend radius: A specified radius over which shapes blend. This replaces the infinite range of the standard smooth minimum.
#### usage
- [S1, unofficial] Used in Dreams for creating intuitive sculpting tools for user-generated content.

### Composite SDF via Union and Subtraction
#### definition
- [S2] The SDF for bubbles is defined as the union of N ellipsoids. Since ellipsoid SDFs lack a simple closed form, a lower bound of the signed distance field is used. Each bubble $i$ has position $\mathbf{x}_i$, semi-axis lengths $\mathbf{r}_i$, and orientation angles $\theta_i, \phi_i$ in the $\{X, Z\}$ and $\{Y, Z\}$ planes.
#### formula
- [S2] The full bubble field SDF is the union of all N individual distance functions: $S_{bubbles}(\mathbf{x}) = \bigcup_{i=1}^N B_i(\mathbf{x} - \mathbf{x}_i, \mathbf{r}_i, \theta_i, \phi_i)$, where the union operator $\mathcal{S}$ is defined by $\mathcal{S}(S_1, S_2) = \min(S_1, S_2)$.
- [S2] The bubbles are subtracted from the displaced panel using the intersection operator $\bigcap$, defined by $\bigcap(S_1, S_2) = \max(S_1, -S_2)$. The final distance function is $S_{final} = \bigcap(S_{panel}, S_{bubbles})$.
#### layout
- [S2] To manage a high number of bubbles efficiently, a regular voxel grid is defined inside the panel. Each voxel contains a maximum of one bubble to prevent overlapping. This allows $S_{bubbles}$ to be interpreted as a single SDF and traversed using a fast voxel traversal algorithm.

## Background
The application of SDFs to Constructive Solid Geometry (CSG) allows for the construction of 3D shapes through unions and subtractions of primitives, enabling simple and intuitive sculpting tools [S1, unofficial]. This approach is particularly favorable in user-generated content games like *Dreams*, where objects are constructed from lists of edits (primitive shapes) applied in a linear, right-leaning CSG tree [S1, unofficial]. The use of smooth blending functions, such as the smooth minimum, facilitates organic transitions between shapes, although these functions traditionally have an infinite range, requiring modification for fixed-radius blending in specific applications [S1, unofficial].

In scientific and industrial contexts, such as the modeling of old glass, SDFs are used to represent complex features like bubbles. These are modeled as unions of ellipsoids (using lower-bound approximations due to the lack of closed-form ellipsoid SDFs) and subtracted from base panels [S2]. To handle high densities of such features, spatial partitioning structures like regular voxel grids are employed to enable efficient traversal and evaluation [S2].

## Sources
[S1] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S2] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
