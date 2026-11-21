# Basic Primitive Definitions

Also called: Analytical SDF Primitives, Closed-Form SDF Primitives

## Overview
Basic Primitive Definitions provide the foundational analytical functions that compute the exact Euclidean signed distance from a query point to standard geometric shapes, serving as the atomic building blocks for distance-field ray marching [S1, unofficial]. They replace explicit mesh geometry with implicit mathematical definitions, enabling safe, large step sizes along rays by leveraging the continuous scalar field where the sign indicates interior ($d < 0$) or exterior ($d > 0$) status . Unlike polygonal primitives or iterative approximations, these primitives rely on closed-form algebraic solutions for standard shapes like spheres, boxes, and tori [S1, unofficial].

## Shared principles
All primitives in this family adhere to a common computational framework:
1.  **Canonical Placement:** Primitives are defined in a local coordinate system centered at the origin $(0,0,0)$ and axis-aligned where applicable .
2.  **Transformation:** To render a primitive at a different position, rotation, or scale, the query point $p$ is transformed into the primitive's local space (inverse transformation) before evaluation, or the primitive parameters are adjusted accordingly [S1, unofficial].
3.  **Signed Output:** The function returns a scalar $d \in \mathbb{R}$. $d < 0$ indicates the point is inside the primitive, $d > 0$ indicates it is outside, and $d = 0$ lies on the surface [S1, unofficial].
4.  **Gradient Availability:** The functions are differentiable almost everywhere, allowing for the computation of surface normals via the gradient $\nabla d$ [S1, unofficial].

## Subtopics

### [SDF Primitives](1731-sdf-primitives.md)
Standard analytical functions for spheres, boxes, tori, planes, cylinders, capsules, and octahedra. Chosen for their simplicity and exact closed-form solutions for common shapes.

### [Polyhedral Primitives](2529-polyhedral-primitives.md)
Exact SDFs for convex polyhedra and prisms. Chosen when complex angular geometries are required without discretization artifacts, relying on piecewise-linear face constraints.

### [Cone Signed Distance Function](2594-cone-signed-distance-function.md)
Exact distance to finite cones, including lateral mantle and base cap. Chosen for tapered geometries, distinguishing itself from infinite cones and cylinders by handling base boundaries.

### [Segment Signed Distance Function](2679-segment-signed-distance-function.md)
Distance to line segments defined by endpoints. Chosen as the geometric core for 1D primitives and extruded shapes, differing from infinite lines by clamping the projection parameter.

### Inverted Torus Rendering
- **Definition:** Renders the torus interior tube as empty space by negating the torus SDF, enabling natural front-to-back occlusion and eliminating donut-hole artifacts [S1, unofficial].
- **Step:** Negate torus SDF: `return -(d_ring - r)`. Treat negative SDF as empty space (tube cavity), positive SDF as solid (donut hole, outside). Ray march naturally finds tube interior [S1, unofficial].
- **Pitfall:** Prevents donut-hole artifacts by ensuring ray marching only operates inside the tube cavity [S1, unofficial].

### Butterfly SDF Primitives
- **Definition:** Signed Distance Field functions for the butterfly's components: `sdWing` for the flapping wings, `sdBody` for the thorax/abdomen, and `sdAntenna` for the antennae. These are combined in `mapButterfly` to return the minimum distance and material ID [S2, unofficial].
- **Code:** Wing SDF logic un-bends the point to wing space, checks the outline, and calculates thickness and camber to form a slab. Body SDF logic uses a parametric radius function based on normalized Z coordinate `u`. Antenna SDF logic uses a quadratic bezier-like curve `c` for the spine and varying radius. Wing Outline is defined as two rounded triangles (forewing and hindwing) using `sdTriangle`, with a notch between them [S2, unofficial].

### Superellipsoid Primitives
- **Definition:** Uses superellipsoids and a 'slice-27 grid' (3D equivalent of Flash's slice-9 grids) for geometry primitives, allowing rounded 3D shapes with customizable corners [S3, unofficial].
- **Step:** Specify primitive properties: corner distance (independently for xyz), wall thickness, or bounding box for visible region. Combine primitives using Voronoi style distance (typically clamped to xy or ground plane) or advanced boolean intersections with material priority. Apply volumetric texturing in object space with automatically generated texture coordinates measured in meters to scale independently of voxel size. Generate UVW coordinates along length, height, and depth (into walls). Assign special material IDs for distinct normal generation (e.g., distinct boards) [S3, unofficial].

## Background
The concept of using analytical signed distance functions for ray marching was popularized by Inigo Quilez in his online articles and books on procedural modeling [S1, unofficial]. The specific implementations for basic primitives (sphere, box, torus) are standard in the computer graphics community for implicit modeling and collision detection. The inverted torus technique for rendering tube-like structures without artifacts is documented in specific GPU implementation guides [S1, unofficial]. Butterfly and superellipsoid variants represent specialized applications in procedural generation and voxel-based world generation [S2, S3, unofficial].

## Sources
[S1] Add OpenGL GPU ray marching renderers for Intel HD Graphics 630 with Jump Flood Algorithm optimization, https://github.com/rhythmatician/phyllotaxis-torus/pull/1, fetched 2026-10-26 (unofficial)
[S2] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S3] How Does It Work?, https://www.voxelquest.com/news/how-does-it-work, fetched 2026-10-29 (unofficial)
