# Ray-primitive intersection


## Overview
Ray-primitive intersection is the family of geometric algorithms that determine the intersection between a parametric ray and a specific geometric primitive (such as a box, sphere, triangle, or implicit surface) [S1, S3, unofficial]. It solves the problem of finding the closest intersection point along a ray to enable correct rendering of opaque surfaces, volume traversal bounds, and occlusion testing [S1, S2, unofficial]. This family is distinguished from [Ray-marching](52-ray-marching.md) by its reliance on exact analytical solutions for primitive shapes, whereas marching uses fixed steps [S1, unofficial]. It serves as the core evaluation step within a broader ray-scene traversal, where the test is executed for multiple objects to find the global minimum $t$ [S1, S2, unofficial].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Ray-box intersection](776-ray-box-intersection.md)
The Ray-box intersection method (specifically the Slab Test) computes the parametric entry ($t_{near}$) and exit ($t_{far}$) distances of a ray intersecting an Axis-Aligned Bounding Box (AABB). It solves the problem of determining the valid sampling interval along a ray for volume rendering or ray tracing, enabling early rejection of rays that miss the volume or primitives. It sets itself apart from [Ray-box intersection via OpenGL Geometry Shaders](1768-ray-box-intersection-via-opengl-geometry-shaders.md) by using an analytical arithmetic approach rather than geometric shader primitives, and from [Ray-Volume Intersection (Face Distance Method)](2381-ray-volume-intersection-face-distance-method.md) by operating on the bounding box of the volume rather than the volume's internal structure directly.

### [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md)
This method determines if a ray intersects a triangle in 3D space using Plücker coordinates to evaluate the winding order of the triangle edges relative to the ray. It handles arbitrary ray directions and is distinct from barycentric approaches like Möller-Trumbore by relying on cross-product-based Plücker line coordinates rather than solving a linear system for barycentric weights.

### [Ray Crossing Count (Winding Number)](915-ray-crossing-count-winding-number.md)
The Ray Crossing Count method determines whether a query point lies inside, outside, or on the boundary of a closed triangle mesh by casting rays and counting signed intersections with the mesh geometry. It solves the point-in-mesh containment problem required for interior testing in volume rendering algorithms like Marching Cubes. Unlike simple binary hit tests, this method computes a generalized winding number: it returns exactly 1 for points inside a closed, outward-oriented mesh, 0 for points outside, and a continuous value between 0 and 1 for points near holes or in non-manifold configurations. A point outside the object yields 0, no point yields values < 0, and points inside yield exactly 1.

### [Sphere Intersection](1620-sphere-intersection.md)
A method to render a volumetric sphere by intersecting camera rays with the sphere geometry to determine entry and exit points, then applying Beer's Law along the chord length. It calculates the parametric distances ($t$) where a ray enters and exits the sphere to compute the optical path length, which is used to determine transmittance and scattered light contribution. Unlike [Ray-box intersection](776-ray-box-intersection.md) which handles axis-aligned bounding volumes, this method provides an exact analytical solution for spherical primitives, serving as a foundational building block for simple volumetric scenes before advancing to complex density fields or [Ray-Marching](52-ray-marching.md) techniques.

### [Ray-Volume Intersection (Face Distance Method)](2381-ray-volume-intersection-face-distance-method.md)
The Ray-Volume Intersection (Face Distance Method) is a two-step procedure to determine if a viewing ray intersects the data volume and to find the entry point for volume traversal. It calculates distances from the ray origin to each of the six faces of the volume geometry, divides by the corresponding ray direction component, and selects the smallest valid distance as the nearest impact point.

### [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md)
This family covers algorithms for determining the intersection of a ray with a tetrahedron, a fundamental operation in volume rendering on unstructured grids and tetrahedral mesh rendering [G]. It solves the problem of finding the entry and exit points (and their parametric distances) of a ray as it traverses a tetrahedral cell [G]. The methods differ in their geometric representation (Plücker coordinates, barycentric coordinates, plane equations) and optimization strategies (early exits, GPU wave intrinsics, connectivity-based traversal) [G]. Key distinctions include the use of Plücker coordinates to avoid explicit plane intersections [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md), the use of linear algebra for point containment [Point-in-Tetrahedron Test via Barycentric Coordinates](2591-point-in-tetrahedron-test-via-barycentric-coordinates.md), and the use of mesh shaders for parallel evaluation [Mesh Shader Tetrahedron Intersection Optimization](5174-mesh-shader-tetrahedron-intersection-optimization.md).

### [Point-Triangle Distance](2776-point-triangle-distance.md)
Computes the closest point on a triangle to a query point and returns the squared distance, handling all edge and vertex cases via Voronoi-region decomposition. It differs from [Ray-primitive intersection](769-ray-primitive-intersection.md) by solving a static point-to-mesh query rather than a ray-scene traversal, and from [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md) by returning the geometric closest point rather than an intersection parameter [G].

### [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md)
This family of methods determines the intersection between a ray and a surface defined implicitly by a scalar field function $f(\mathbf{x}) = 0$ (or $f(\mathbf{x}) - v = 0$ for an isovalue $v$). Unlike explicit mesh intersection, these methods solve the root-finding problem $f_t(t) = 0$ for the smallest $t > 0$ along the ray $\vec{p}(t) = \vec{o} + t\vec{d}$ without extracting a proxy triangle mesh. They are distinguished from sibling geometric primitives (e.g., [Ray-box intersection](776-ray-box-intersection.md)) by their reliance on function evaluation and numerical root-finding rather than analytic geometric formulas, and from sibling SDF methods (e.g., [SDF Raycast Collision Detection](2417-sdf-raycast-collision-detection.md)) by their broader applicability to general implicit surfaces beyond simple collision detection.

### [Exact Rational Ray-Surfel Evaluation](2915-exact-rational-ray-surfel-evaluation.md)
Exact Rational Ray-Surfel Evaluation computes the intersection of a viewing ray with a flat surfel rectangle in its local $(u, v)$ coordinates using a homography approximation around the center of the screen-space bounding box. It sets itself apart from the reference 2DGS formulation by replacing the per-pixel cross-product-based inversion with a precomputed affine approximation, reducing computational cost while maintaining visual fidelity.

### [LinPrim Gradient Computation](3038-linprim-gradient-computation.md)
LinPrim Gradient Computation computes the gradients of intersection depths $\mathbf{i}$ with respect to the ray-space triangle corners $\mathbf{v}^{\text{ray}}$ to enable differentiable rendering of linear primitives. It differs from standard 3D Gaussian Splatting (3DGS) by backpropagating gradients through a linear primitive rasterizer (MTIA) rather than a Gaussian splatting kernel, requiring specific handling of ray-space to world-space transformations and quaternion-based rotation updates. It builds on [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by replacing the Gaussian alpha-blending gradient flow with opacity calculations based on intersection depths.

### [Ray ID Caching](3110-ray-id-caching.md)
Ray ID Caching is an optimization technique used in volume rendering and ray tracing to eliminate redundant intersection tests when a single ray intersects the same geometric object multiple times across different voxels. It solves the problem of "multiple intersection" overhead by storing a unique identifier (`rayID`) on each ray and the last intersecting `rayID` on each object. Unlike methods that require precise voxel-object containment checks (such as Glassner's method), this technique allows for looser voxel lists because redundant intersections are filtered by the ID comparison rather than geometric precision.

### [IRIS Ray-Geometry Intersection](3250-iris-ray-geometry-intersection.md)
IRIS Ray-Geometry Intersection is an intersection-aware ray-based sampling method for implicit editable scenes that determines samples as the maximum response points of Gaussians intersected by each ray. It solves the problem of efficiently sampling 3D Gaussians in large-scale or bounded scenes by leveraging hardware-accelerated ray-triangle intersection tests via proxy geometry. Unlike direct Gaussian intersection methods, it maps each Gaussian to a convex polyhedral proxy (an affine-transformed icosahedron) to utilize RT cores, setting it apart from siblings like [Ray-Splat Intersection](6816-ray-splat-intersection.md) which may use different primitive approximations or software-based tests.

### [Ray-Sphere Intersection](3539-ray-sphere-intersection.md)
Ray-sphere intersection computes the entry and exit points (or the nearest hit distance) where a parametric ray intersects a sphere defined by a center and radius. In volume rendering, this operation is primarily used to determine the marching bounds for ray-marching algorithms, ensuring samples are only taken within the volume's extent, and to perform early-outs in hierarchical traversal. It is distinct from [Sphere Intersection](1620-sphere-intersection.md) in that it solves for the intersection of a line (the ray) with a surface, whereas the latter typically refers to finding points on a sphere's surface satisfying other constraints (such as the 3-sphere trilateration variant). It is analytically exact and generally faster than iterative methods like [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) for simple spherical primitives.

### [Ray-Object Intersection Test](3604-ray-object-intersection-test.md)
The Ray-Object Intersection Test is the fundamental geometric primitive used to determine the closest intersection between a ray and a specific scene object, returning the intersection distance $t$ or a failure indicator if no intersection exists. It serves as the core evaluation step within a broader ray-scene traversal, where the test is executed for multiple objects to find the global minimum $t$. What sets this method apart from sibling primitives like [Ray-box intersection](776-ray-box-intersection.md) or [Sphere Intersection](1620-sphere-intersection.md) is its role as the generic interface for arbitrary geometry; it typically handles coordinate space transformations and delegates the specific algebraic solution to the object's internal representation, rather than implementing the intersection algebra directly in the traversal loop.

### [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md)
Azimuthal Hit Detection determines whether a ray intersects an azimuthal section within a spherical voxel grid, where azimuthal sections are defined in the XZ plane. It serves as a geometric primitive for traversing spherical volume data, specifically handling the angular discretization along the azimuthal axis. It is distinct from [Ray-box intersection](776-ray-box-intersection.md) which handles axis-aligned bounding volumes, and [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md) which handles general 3D simplicial volumes; instead, it leverages the specific planar geometry of the azimuthal slices.

### [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md)
Ray-Polar Section Intersection determines whether a ray intersects a "polar section," a geometric primitive defined within the XY plane of a spherical voxel grid. It serves as the angular traversal step for rays passing through spherical volume data, identifying which radial angular bin (section) the ray occupies at a given depth. It is distinct from [Ray-box intersection](776-ray-box-intersection.md) by operating on angular sectors rather than Cartesian cubes, and from [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md) by using the Y-axis as the polar axis rather than the Z-axis.

### [Ray-Cylinder Intersection](5014-ray-cylinder-intersection.md)
Computes the entry and exit distances for a ray intersecting a cylinder defined by `geomSize`, composed of front/back rectangles, top/bottom faces, and a lateral curved surface. It differs from [Ray-box intersection](776-ray-box-intersection.md) by handling the curved lateral surface via quadratic solving and cylindrical coordinate checks, and from [Ray-box Intersection via OpenGL Geometry Shaders](1768-ray-box-intersection-via-opengl-geometry-shaders.md) by being a general-purpose CPU/GPU kernel implementation rather than a shader-specific primitive assembly.

### [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md)
Ray Tracing and Ray Intersection Methods determine the first object intersected by a camera ray by analytically computing intersection distances for simple geometries. The core problem is finding the closest intersection point along a ray to render opaque surfaces correctly. This family is distinguished from [Ray-marching](link-to-marching) by its reliance on exact analytical solutions for primitive shapes, whereas marching uses fixed steps. It differs from [Ray-box intersection](776-ray-box-intersection.md) and [Ray-Triangle Intersection](909-general-ray-triangle-intersection-pl-cker-coordinates.md) by encompassing the broader algorithmic strategies (such as packetization and hybrid implicit tracing) used to accelerate these primitive tests in volume rendering contexts.

### [Point-Plane Reflection](5920-point-plane-reflection.md)
Point-Plane Reflection computes the mirror image of a point $\mathbf{p}$ across a plane defined by a normal vector $\mathbf{n}$ and a scalar offset $c$. It is primarily used in volume rendering and ray tracing to determine specular reflection paths, calculate virtual camera positions for mirror surfaces, or perform spatial transformations in signed distance field (SDF) operations. Unlike general ray-plane intersection which finds a hit point, this method transforms a point in space. It differs from [Ray-box intersection](776-ray-box-intersection.md) and [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md) by operating on a single point relative to an infinite plane rather than intersecting a ray with a bounded primitive.

### [Ray-Primitive and Segment Intersection](6230-ray-primitive-and-segment-intersection.md)
This method optimizes the evaluation of multiple ray-primitive intersections along a single ray by establishing a strict processing order based on bounding box distances, thereby reducing the computational load of expensive primitive intersection tests. It distinguishes itself from standard intersection tests by focusing on the *sequence* of calculations rather than the geometric solution itself, ensuring that faster, coarser intersection points are resolved before slower, finer segment evaluations.

### [Ray-Plane/Rectangle Intersection](6498-ray-plane-rectangle-intersection.md)

### [Ray-Ray Intersection](7624-ray-ray-intersection.md)
Computes the least-squares intersection point of two rays in 3D space, finding the point that minimizes the squared distance to both rays when they do not intersect exactly due to noise or parallax. It differs from exact intersection methods by providing a robust solution for skew lines, which is essential in volume rendering pipelines where ray origins and directions may have minor misalignments.

### [Ray-Splat/Gaussian Intersection](7916-ray-splat-gaussian-intersection.md)
This family defines the geometric primitives and intersection algorithms required to determine the interaction between a viewing ray and 3D Gaussian splats, which are the fundamental primitives in Gaussian Splatting rendering. It solves the problem of efficiently identifying which splats contribute to a pixel and computing their depth and opacity for alpha blending. It is distinguished from sibling methods by its focus on ellipsoidal (Gaussian) volumes rather than polyhedral or spherical primitives, often leveraging hierarchical structures like BVHs or specific plane-intersection techniques for numerical stability.

### [Quadtree Raycasting](8208-quadtree-raycasting.md)
Quadtree Raycasting is a recursive algorithm that traverses a 2D quadtree data structure to find the first solid node intersected by a ray. It solves the problem of efficiently locating geometry in sparse or hierarchical 2D volumes by skipping empty subtrees. It differs from standard grid-based raycasting (like [Ray-box intersection](776-ray-box-intersection.md)) by adapting the traversal step size and order to the hierarchical node boundaries rather than a uniform grid, and it differs from [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md) by operating on axis-aligned square quadrants.

### Medium Stack Update
- **Definition**: Updates the ray's medium stack when crossing a surface, pushing a new medium if entering and popping if exiting [S4, unofficial].
- **Steps**: Check if transmission occurred: `isTransmitted(-ray.direction, wi, ns)`. If entering (`isEntered(wi, ns)` is true) and primitive has medium, push medium to ray. If exiting, pop medium from ray [S4, unofficial].
- **Formula**: Transmission Check: $(wo \cdot n < 0) \neq (wi \cdot n < 0)$. Entry Check: $wi \cdot n < 0$ [S4, unofficial].

### Ray Marching via C++ Extension
- **Definition**: Performs ray-sphere intersection or ray marching against the SDF grid using a C++ renderer extension to find the rough intersection position for each pixel [S5, unofficial].
- **Steps**: Initialize output tensors for intersection position, voxel index, and ray direction. Call the C++ ray matching function with camera parameters (eye position), grid bounds, resolution, and grid data. Extract the rough intersection position, the index of the minimum corner of the intersecting voxel, and the ray direction from the outputs [S5, unofficial].
- **Pitfall**: Pixels with no intersection are marked by voxel_min_point_index being -1. These must be handled to avoid invalid memory access during interpolation [S5, unofficial].

### Ray Occlusion Test
- **Definition**: Tests if rays intersect any geometry without returning hit details, used for simple occlusion checks [S2, unofficial].
- **Steps**: Initialize RTCRay/RTCRay8. Set origin, direction, tnear = minDist, tfar = InfinityDist. Init context. Call rtcOccluded1 or rtcOccluded8. Return ray.tfar < 0.0f [S2, unofficial].
- **Parameter**: minDist: Minimum ray distance for intersection [S2, unofficial].

### Clipping Cell-Level Slice to Active Rays
- **Definition**: An optimization to reduce cell primitives in a slice by intersecting individual rays with the world-space bounding box of the current u,v slice [S6, unofficial].
- **Steps**: Ignore rays that have already successfully hit a cell (inactive rays), even if they intersect the slice bounding box [S6, unofficial].

### BVH-based UDF evaluation with analytic gradient
- **Definition**: Computes the unsigned distance from query points to a triangle mesh using a BVH, and provides an analytic gradient for backpropagation [S7, unofficial].
- **Steps**: Flatten input x to [N, 3]. Query BVH for distance udf, triangle ID tri_id, and barycentric coordinates uvw. Clamp uvw to [1e-9, 1-1e-9]. Interpolate nearest point on triangle using uvw and triangle vertices. Return udf reshaped to input shape. In backward pass, retrieve saved x and nearest_points. Compute diff = x - nearest_points. Compute norm = diff.norm(dim=-1, keepdim=True) + 1e-8. Gradient is grad_output scaled by diff/norm [S7, unofficial].
- **Formula**: $\nabla_x \text{UDF}(x) = \frac{x - p_{\text{nearest}}}{\|x - p_{\text{nearest}}\|}$ where $p_{\text{nearest}}$ is the closest point on the mesh [S7, unofficial].
- **Layout**: Input x shape [N, 3]. Output udf shape [N, 1]. Saved tensors: x_flat, nearest_points [S7, unofficial].
- **Pitfall**: uvw is clamped to [1e-9, 1-1e-9] to avoid invalid barycentric coordinates at triangle edges/corners [S7, unofficial].

### Ray Cursor
- **Definition**: A selection technique where a ray is emitted from the user's hand, with start point and orientation controlled by the user, selecting the first intersected object [S8, unofficial].
- **Steps**: Emit ray from hand position and orientation. Select first object intersected by ray. Augmentation: Cycle through all ray-object intersection points to disambiguate [S8, unofficial].
- **Pitfall**: Difficult to select small and distant objects due to required angular accuracy. Ray or cone may intersect multiple objects. Disambiguation metric: select object based on distance from targets to the apex and central axis of the cone [S8, unofficial].

### Ray-Image Frame Intersection
- **Definition**: Computes the intersection of a camera-space ray with lines parallel to the image frame axes (x=0, x=1, y=0, y=1) in normalized image coordinates [S9, unofficial].
- **Steps**: Extract intrinsics components: focal lengths $f_s, f_o$ and principal points $c_s, c_o$ for the same and other dimensions. Extract ray origin $o_s, o_o, o_z$ and direction $d_s, d_o, d_z$ components. Construct the 2D intersection point `xy` and 3D point `xyz = origins + t * directions`. Validate the projection: check if `xy` is in bounds $[-\epsilon, 1+\epsilon]$, if `xyz` is in front of camera ($z > -\epsilon$), and if $t > -\epsilon$ [S9, unofficial].
- **Formula**: $c = \frac{v - c_s}{f_s}$, where $v$ is the target coordinate value (0.0 or 1.0). $t = \frac{c o_z - o_s}{d_s - c d_z}$. This is the ray parameter $t$ at the intersection. $y_{other} = c_o + \frac{f_o (o_o (c d_z - d_s) + d_o (o_s - c o_z))}{d_z o_s - d_s o_z}$. This is the other coordinate value at the intersection [S9, unofficial].

### API-Dependent Intersection Method Selection
- **Comparison**: Ray/box intersection test yields better performance with compute shader and OpenCL implementations. Raster intersection test yields better performance with CUDA implementations. No single intersection method is globally superior; optimal choice depends on the target parallel API [S10].
- **Usage**: Best overall results achieved by combining compute shader with ray/box intersection test [S10].

### Intersection using Rasterization
- **Definition**: An efficient way to obtain the intersection between view-rays and the volume using OpenGL rasterization [S10].

### Procedural Ray/Box Intersection
- **Definition**: A method for determining ray entry and exit within the volume bounding box entirely within the fragment shader. The ray is defined by the camera position and the volume entry position [S11, unofficial].
- **Parameter**: Requires an exit criterion to determine when the ray has left the volume [S11, unofficial].
- **Pitfall**: The entire computational load for intersection and marching is placed on the fragment shader, which is a disadvantage compared to rasterization-based setup [S11, unofficial].

### Scan Plane Beam Intersection
- **Definition**: An approximate method to determine if a camera ray intersects a caustic beam and the location of that intersection, used because exact computation is too costly for thousands of beams [S12].
- **Steps**: Define a scan plane, which is an auxiliary arbitrary plane containing the camera ray. Compute the intersections of the scan plane and the edges of the caustic beam to form a triangle. Minimize error from curved beam edges by choosing the scan plane normal vector to be close to the normal of the caustic triangle. Transform the view ray to the beam's local space. Select a representative point on the intersection segment ∆x (specifically the midpoint) and evaluate the area function A(y). Compute the fraction of light scattered towards the viewer based on the cross-section area (determining radiance scaling) and scattering in the participating medium [S12].
- **Formula**: np = rview × (rview × n4), where np is the scan plane normal, rview is the view ray direction, and n4 is the caustic triangle normal [S12].

## Background
The classification of ray-object intersections into five classes based on intersection type, data type, and early termination capability was introduced by Sobierajski et al. [S3, unofficial]. Class 1 covers geometric data intersections requiring a single point, while Classes 2-5 cover volumetric data intersections with varying requirements for early termination and value aggregation (point vs. segment, single value vs. all values) [S3, unofficial]. The use of Plücker coordinates for ray-triangle intersection was detailed in early graphics literature [S1, unofficial]. The distinction between rasterization-based and ray/box intersection-based volume entry determination was analyzed in comparisons of GPU-based volume ray casting implementations [S10]. The use of BVHs for unsigned distance field (UDF) evaluation with analytic gradients was explored in differentiable rendering contexts [S7, unofficial]. The "Ray Cursor" selection technique was introduced by Liang and Green [1994] for immersive 3D environments [S8, unofficial]. The scan plane beam intersection method for real-time volume caustics was proposed by [S12].

## Sources
[S1] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S2] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
[S3] Volumetric Ray Tracing, https://www.cs.drexel.edu/~david/Classes/Papers/p11-sobierajski.pdf, fetched 2026-10-28 (unofficial)
[S4] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S5] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S6] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S7] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S8] Interaction with volumetric displays, http://hdl.handle.net/1807/16733, fetched 2026-10-26 (unofficial)
[S9] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S10] (PDF) A Comparison between GPU-based Volume Ray Casting ..., https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S11] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S12] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
