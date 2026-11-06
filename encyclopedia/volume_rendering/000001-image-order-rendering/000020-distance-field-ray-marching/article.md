# Distance-field ray marching

Also called: Sphere tracing, SDF ray marching

## Overview
Distance-field ray marching is a rendering technique that finds surface intersections by marching along rays in steps, using a distance field (signed distance function) to determine step size, allowing for approximate intersection detection without explicit mesh geometry [S1, unofficial]. It differs from sibling methods like [Direct volume ray casting](19-direct-volume-ray-casting.md) by relying on implicit geometric primitives defined by analytical distance functions rather than sampled voxel grids, and from [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md) by using the distance value itself to drive the traversal step size rather than a fixed or grid-based step [S2, S7, unofficial].

## Shared principles
The method operates on the principle that a Signed Distance Function (SDF) $f(\mathbf{p})$ returns the shortest Euclidean distance from a query point $\mathbf{p} \in \mathbb{R}^3$ to the nearest surface of the scene geometry [S2, S5, S6, S12, unofficial]. The sign of the result indicates the point's position relative to the surface:
- $f(\mathbf{p}) < 0$: The point is inside the primitive.
- $f(\mathbf{p}) = 0$: The point lies exactly on the surface.
- $f(\mathbf{p}) > 0$: The point is outside the primitive.

The core algorithm iteratively advances a ray $r(t) = \mathbf{o} + t\mathbf{d}$ along its direction $\mathbf{d}$ by querying the SDF at the current position $\mathbf{p}_i$. The step size $\Delta t$ is set to the distance value returned by the SDF: $\Delta t = f(\mathbf{p}_i)$ [S2, S7, S14, unofficial][S26][S33, S48, unofficial]. This step size is "safe" because, by the definition of the distance field, no geometry exists closer than $f(\mathbf{p}_i)$ to $\mathbf{p}_i$; thus, the ray cannot overshoot or penetrate a surface [S8, S27, S49, unofficial]. The position is updated as $\mathbf{p}_{i+1} = \mathbf{p}_i + f(\mathbf{p}_i)\mathbf{d}$ [S14, S33, unofficial].

The loop terminates when one of the following conditions is met:
1.  **Surface Hit:** The distance value $f(\mathbf{p}_i)$ falls below a small convergence threshold $\epsilon$ (e.g., $0.001$), indicating the ray is sufficiently close to the surface [S1, S2, S14, S34, S39, unofficial].
2.  **Escape:** The accumulated distance $t$ exceeds a maximum travel distance $t_{max}$, indicating the ray has exited the scene or bounding volume without hitting geometry [S1, S2, S34, unofficial].
3.  **Max Steps:** A predefined maximum number of iterations is reached, preventing infinite loops in case of non-convergence [S1, S2, S39, unofficial].

Scene composition is achieved by combining primitive SDFs using Boolean operations. The union of two shapes with SDFs $f_1$ and $f_2$ is $\min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S1, S38, unofficial]. The intersection is $\max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S38, unofficial]. Subtraction (difference) is $\max(-f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S38, unofficial]. Smooth blending between shapes can be achieved using smooth minimum functions, such as Inigo Quilez's `smoothMin(a, b, k)`: $h = \text{clamp}(k - |a-b|, 0, k) / k$; $\text{result} = \min(a,b) - h \cdot h \cdot k / 6$ [S9, S24, unofficial].

Transformations are applied to the query point before evaluation. Translation by vector $\mathbf{t}$ uses $f'(\mathbf{p}) = f(\mathbf{p} - \mathbf{t})$ [S35, unofficial]. Scaling by factor $s$ uses $f'(\mathbf{p}) = s \cdot f(\mathbf{p}/s)$ [S35, unofficial]. Domain repetition (tiling) is achieved by taking the modulo of the point coordinates: $f'(\mathbf{p}) = f(\mathbf{p} \pmod 1)$ [S35, unofficial].

Surface normals $\mathbf{n}$ at the hit point $\mathbf{p}$ are estimated via the gradient of the SDF: $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [S1, S10, unofficial]. Since analytical gradients are often unavailable or expensive, they are typically approximated using central finite differences with a small offset $\epsilon$ (e.g., $0.001$):
$$ \nabla f(\mathbf{p}) \approx \begin{bmatrix} f(\mathbf{p} + \epsilon \hat{x}) - f(\mathbf{p} - \epsilon \hat{x}) \\ f(\mathbf{p} + \epsilon \hat{y}) - f(\mathbf{p} - \epsilon \hat{y}) \\ f(\mathbf{p} + \epsilon \hat{z}) - f(\mathbf{p} - \epsilon \hat{z}) \end{bmatrix} $$
[S1, S35, S39, S45, S46, unofficial].

## Subtopics

### [Woodcock Tracking with Macrocell Majorant](986-woodcock-tracking-with-macrocell-majorant.md)
This method performs volume rendering using Woodcock tracking (null-collision) within coarse mesh elements, utilizing macrocell densities to determine the majorant extinction value. It builds on [Distance-field ray marching](21-distance-field-ray-marching.md) by replacing deterministic distance-based steps with stochastic steps derived from a majorant function, suitable for participating media rather than solid surfaces.

### [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md)
Ray-marching on 3D Gaussians is a volumetric rendering method that evaluates the full Gaussian mixture along rays to accurately handle primitive overlap and compute the volumetric integral, serving as a ground-truth baseline for analyzing approximations in standard 3D Gaussian Splatting (3DGS). It differs from the parent by querying a density/opacity field defined by a mixture of Gaussians rather than a Signed Distance Function (SDF), and by using adaptive step sizes to resolve overlaps rather than fixed distance-based steps.

### [Bounded 3D Distance Texture Ray Marching](1115-bounded-3d-distance-texture-ray-marching.md)
Bounded 3D Distance Texture Ray Marching renders implicit surfaces defined by a pre-computed 3D signed distance texture by combining direct texture lookups with geometric bounding box checks to handle boundary conditions. It solves the problem of invalid or noisy distance values at the edges of a texture volume by clamping or reflecting queries.

### [SDF Primitive Definitions](1578-sdf-primitive-definitions.md)
SDF Primitive Definitions provide the fundamental analytical building blocks for distance-field ray marching by defining the signed Euclidean distance from any point in 3D space to the surface of a geometric primitive. This method solves the problem of representing complex geometry without explicit mesh data by providing closed-form distance estimators for shapes like spheres, boxes, and tori.

### [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md)
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by computing valid Sphere Distance Bounds (SDB) using a dynamic Bounding Volume Hierarchy (BVH) derived from the FBM structure itself. It enables early termination of octave summation and adaptive stepping, addressing the high computational cost of evaluating noise functions at every step.

### [Raymarching with Depth Buffer Occlusion](1880-raymarching-with-depth-buffer-occlusion.md)
Raymarching with Depth Buffer Occlusion is a technique that allows raymarched Signed Distance Field (SDF) objects to correctly occlude and be occluded by standard mesh-based geometry. It solves the problem where raymarched objects typically float on top of all other scene elements by integrating the standard sphere tracing loop with a depth buffer lookup to limit the maximum travel distance.

### [SDF Depth Buffer Integration](1955-sdf-depth-buffer-integration.md)
SDF Depth Buffer Integration is a technique that extends sphere tracing by explicitly writing the computed surface hit point's depth to the hardware depth buffer (`SV_Depth`). It solves the problem of incorrect occlusion between ray-marched SDF geometries and other rendered objects by ensuring the depth buffer is updated with the accurate Z-value of the SDF hit.

### [Ray Marching with Reflections](1963-ray-marching-with-reflections.md)
Ray Marching with Reflections extends distance-field ray marching by continuing the ray path after a surface intersection to simulate specular reflections, effectively transitioning from raycasting to raytracing. Unlike standard sphere tracing which terminates at the first hit, this method computes a new ray direction based on the surface normal and reflectivity, casting secondary rays recursively.

### [Lighting and Shading Models](2055-lighting-and-shading-models.md)
Lighting and Shading Models compute the color of a surface point identified by distance-field ray marching by combining geometric properties (normals, depth) with light source parameters. This family solves the problem of transforming raw geometric hits into visually plausible images by simulating diffuse, specular, and ambient lighting components using the estimated normals from the SDF gradient.

### [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md)
Differentiable Sphere Tracing (DIST) is a rendering algorithm for implicit Signed Distance Functions (SDFs) represented by neural networks, enabling efficient forward and backward passes for inverse graphics optimization. It solves the non-differentiability and computational prohibitive nature of standard sphere tracing when used with gradient-based optimization by providing a differentiable path through the marching loop.

### [Ray Marching with Neural SDF](2409-ray-marching-with-neural-sdf.md)
Ray Marching with Neural SDF approximates the Space Distance Field (SDF) using a deep neural network to render images by marching rays along the estimated distance field. Unlike traditional sphere tracing which relies on analytical primitive definitions, this method uses a learned implicit representation, allowing for high-fidelity reconstruction of complex shapes from sparse data.

### [Surface Point Sampling for SDFs](2708-surface-point-sampling-for-sdfs.md)
Surface Point Sampling for SDFs generates training or rendering data points distributed around the implicit surface defined by a Signed Distance Function (SDF). Unlike standard sphere tracing which samples along a ray to find a single intersection, this method actively constructs a point cloud $P$ to capture geometry accurately for neural network training or high-quality rendering.

### [Ray Marching Dither](3269-ray-marching-dither.md)
Ray Marching Dither is a technique to reduce banding and layering artifacts caused by low step counts or fixed step sizes in distance-field ray marching. It perturbs the ray's sampling position or step size using random values (often blue noise) to break up coherent aliasing artifacts, improving visual quality at low computational cost.

### [Recursive Ray Marching with Early Stopping](3546-recursive-ray-marching-with-early-stopping.md)
Recursive Ray Marching with Early Stopping is a parallelization technique for sphere tracing that prevents wasted computation on rays that have already converged to a surface or exited the scene. It differs from its sibling [Adaptive Ray Marching with Dynamic Step Count](3888-adaptive-ray-marching-with-dynamic-step-count.md) by using a boolean mask to track active rays and excluding converged rays from subsequent SDF queries.

### [Deformation and Transformation Handling](3850-deformation-and-transformation-handling.md)
This section covers techniques for handling dynamic geometry in distance-field ray marching. It involves applying time-varying transformation matrices to the query points before SDF evaluation, or using deformation fields that modify the distance function itself to simulate bending, twisting, or morphing shapes.

### [Sphere Tracing with Composite SDF and Adaptive Sampling](3867-sphere-tracing-with-composite-sdf-and-adaptive-sampling.md)
This method renders scenes containing primitive shapes, character segments (letters), and arcs by combining their Signed Distance Functions (SDFs) into a composite scene and tracing rays via sphere tracing. It extends standard sphere tracing by handling specific geometric primitives and using adaptive sampling to improve quality in complex regions.

### [Spherical Volume Voxel Traversal](3877-spherical-volume-voxel-traversal.md)
Spherical Volume Voxel Traversal is an algorithm that traces a ray with a unit direction over a spherical voxel grid, returning a vector of the spherical coordinate voxels traversed. It serves as the spatial acceleration structure for rendering volumetric data defined in spherical coordinates, optimizing traversal for non-Cartesian domains.

### [Lidar Sphere Tracing](3889-lidar-sphere-tracing.md)
Lidar Sphere Tracing is a rendering technique that combines distance-field ray marching with point cloud projection to visualize scene intersections as a sparse Lidar point cloud rather than a continuous surface. It differs from standard sphere tracing by discarding the full volumetric integration and instead recording the first hit point as a discrete sample, simulating Lidar sensor output.

### [Step Size and Termination Rules](4132-step-size-and-termination-rules.md)
Step Size and Termination Rules define the logic for determining how far a ray advances between samples and when the marching process halts, solving the trade-off between rendering accuracy (avoiding tunneling or missing thin features) and computational performance. This family is distinct from [Standard Sphere Tracing](6223-sphere-tracing-with-sdfs.md) by allowing variable step sizes based on local geometry or noise derivatives.

### [Neon Raymarcher](4157-neon-raymarcher.md)
Neon Raymarcher is a real-time WebGL2 renderer that generates an infinite procedural synthwave city using signed-distance fields (SDFs) within a single fullscreen fragment shader, requiring no meshes, textures, or scene graph. It distinguishes itself from general-purpose SDF renderers by focusing on a specific aesthetic style and optimizing the SDF composition for a single-pass, highly constrained environment.

### [Scale-Invariant Integral Surface Rendering](4266-scale-invariant-integral-surface-rendering.md)
Scale-Invariant Integral Surface Rendering (SCALIS) is a ray-marching algorithm designed to render integral surfaces defined by skeletons with varying radii, significantly reducing computation time compared to standard sphere tracing. It addresses the specific problem of excessive step counts in large-scale scenes by using scale-invariant properties to skip empty space more aggressively.

### [Primary Ray Integrator](4401-primary-ray-integrator.md)
The Primary Ray Integrator is a single-function ray integrator that resolves a complete ray by compositing terrain, water, sky, cloud volumes, and aerial perspective. It serves as the core rendering loop for complex natural scenes, handling the intersection logic for multiple surface and volume types within a unified marching framework.

### [SDF Normal Estimation via Finite Differences](4619-sdf-normal-estimation-via-finite-differences.md)
This method estimates the surface normal at a point $p$ on a Signed Distance Field (SDF) by numerically computing the gradient of the distance function $\nabla f(p)$ using finite differences, then normalizing the result. It is the standard approach for shading when analytical normals are not available.

### [Antialiasing (AA)](4802-antialiasing-aa.md)
Antialiasing techniques for distance-field ray marching address the jagged edges caused by discrete sampling. Methods include supersampling (casting multiple rays per pixel), temporal antialiasing (jittering ray origins over time), and analytical antialiasing using the derivative of the SDF to estimate the probability of intersection within a pixel footprint.

### [Volumetric and Atmospheric Rendering](4968-volumetric-and-atmospheric-rendering.md)
Volumetric and Atmospheric Rendering is a family of techniques that render participating media (clouds, fog, smoke) and planetary atmospheres by integrating light transport along rays. It differs from [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by solving the radiative transfer equation (or a simplified approximation) rather than treating the volume as an opaque or translucent surface defined by an isosurface.

### [Acceleration and Optimization](6222-acceleration-and-optimization.md)
This family covers techniques to reduce the computational cost of distance-field ray marching and volume traversal, addressing the bottleneck of excessive Signed Distance Function (SDF) evaluations or ray steps. It sets itself apart from sibling methods by focusing on spatial data structures (like BVHs or grids) and algorithmic optimizations (like adaptive stepping) to skip empty space.

### [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md)
Sphere tracing with SDFs is a technique for rendering implicit surfaces by casting rays and stepping along them using the distance field as a guide. It solves the problem of rendering complex, analytic geometry without explicit mesh data by replacing triangle intersection tests with SDF evaluations.

#### Sampler Abstraction
A trait-based abstraction for sampling distance fields defined in terms of specific Distance metrics. It allows samplers to abstract sampling across multiple different Distance metrics [S54, unofficial]. Implementation for `ScalarSource` calls `self.source.sample_scalar(p)`, for `VectorSource` calls `self.source.sample_vector(p)`, and for `HermiteSource` calls `self.source.sample_normal(p)` [S54, unofficial].

#### Cube-tracing
An algorithm using the chessboard metric (or Manhattan metric) for distance computation, potentially resulting in more efficiently computed distances and ray intersection [S26]. Left as further research.

#### Spatially Varying Threshold
A technique to use FBM for amplifying shapes (hypertextures) by defining an isosurface $SDF(p) + B_n(p) = 0$, which is equivalent to finding the root of $B_n(p) = v(p)$ where $v(p) = -SDF(p)$ [S55]. To handle a spatially varying threshold $v(p)$, approximate it non-safely using its first-order Taylor polynomial at the current ray position $t_0$: $v(t) = v(t_0) + v'(t_0) \cdot (t - t_0)$ [S55]. If the intersection with $v(p)$ is too hard to compute, use the forward inclusion function $[v](p)$ instead to safely approximate $v(p)$ [S55].

#### ST-Fallback Focus Sphere
An alternative focus+context visualization used when WebGL2 image2D/atomic_uint support is unavailable (e.g., SwiftShader, sandboxed environments) [S56, unofficial]. It modulates the per-sample alpha of a ray-march shader using a focus sphere kernel. The sphere center is initialized to the mesh bounding box center, and the radius to 25% of the smallest bounding box dimension [S56, unofficial]. The focus mask is computed as `focusMask = pow(clamp(distance(p, sphereCenter) / sphereRadius, 0, 1), 4.0)` [S56, unofficial].

#### Ray Marching through Distance Fields
A traversal method where ray marching is performed through distance fields. Naive storage of one distance value per voxel is impractical for large grids due to memory and bandwidth costs [S57, unofficial]. Memory costs are reduced by lowering field resolution to 1:4 scale (one distance value per 64 voxels), combined with occupancy masks [S57, unofficial]. Pre-processing takes 10x more GPU time than XBrickMap, and traversal is not considerably shorter than space partitioning methods (~8% difference) [S57, unofficial].

#### Ray Subdivision with Polynomial Interpolation for SCALIS Fields
A ray-tracing method for Scale-Invariant Integral Surfaces (SCALIS) that avoids expensive brute-force field evaluations by subdividing the ray into intervals based on local minima of homothetic distance fields, then using iterative quadratic polynomial interpolation to find isosurface crossings [S58]. The normalized field $g(f)$ for an infinite line primitive with radius $\tau_0$ is defined as $g(f) = \left( \tilde{k}^{-1} \left( \frac{f}{\tilde{\tau}} \right) \right)^2 = \left( \frac{d}{\tau_0} \right)^2$ [S58].

#### Blinn Soft Objects Ray Tracing
A method to ray trace Blinn soft objects with constant radius by applying sphere-tracing after performing a mapping that allows computing a signed distance approximation to the density field [S58].

#### Chunked Sphere Tracing
A technique to process rays in chunks to manage memory or parallelism. It initializes an empty list `chunk_colors`, loops over chunks of size `chunk_size`, slices the ray bundle, calls the parent forward method, and concatenates the results [S59, unofficial].

#### Coarse-to-Fine Sphere Tracing
A strategy to speed up sphere tracing by starting at low resolution and splitting rays to higher resolutions, leveraging the fact that nearby rays march similarly [S60]. It initializes rays at 1/4 original resolution, performs 3 marching steps, splits each ray into four rays to double the resolution, and repeats until full resolution is reached [S60].

#### OctantDF
Space skipping traversal using tiled 256³ 8-directional distance fields at 1:4 resolution, combined with 4³ occupancy bitmasks [S57, unofficial]. It creates 8 separate manhattan fields, each considering only voxels ahead of the associated ray octant direction [S57, unofficial]. Memory cost is 8 bytes per 4³ voxel plus 1 bit per 1³ voxel [S57, unofficial]. Compared to traditional distance fields, it lowers the overall number of iterations by ~20-25% and increases throughput for incoherent rays by ~30-50% [S57, unofficial]. On Sponza 2k, OctantDF achieves 129.0 Mrays/s for primary rays and 84.9 Mrays/s for primary + 1 diffuse bounce [S57, unofficial].

#### Span-Based Wing Un-bending
A technique to render thin, flapping sheets (like butterfly wings) by using the distance from the body's long axis (span) as an invariant coordinate [S61, unofficial]. The span is $\mathrm{span} = \lVert p_{xy} \rVert$, where $p_{xy}$ is the projection of the point $p$ onto the plane perpendicular to the body axis [S61, unofficial]. The flap angle is $\theta = \Theta\left(t - \lambda\,\mathrm{span}\right)$, where $\Theta$ is the stroke function, $t$ is time, and $\lambda$ is the tip lag coefficient [S61, unofficial].

#### Tri-planar procedural materials
Applies world-projected rgb-multiplier textures with a shared micro-grain layer, continuous across voxel boundaries [S62, unofficial]. Generates natural strata-and-crack stone, clumpy dirt with pebbles, wind-rippled sand, drifted sparkling snow, per-species bark, streaked ice, ore veins, pulsing lava crust cracks and ribbed spiny cactus [S62, unofficial].

#### Local Distance Field Ambient Occlusion
A modern approach offloading meshing and AO to compute shaders to achieve smooth, high-fidelity AO that avoids the rigid look of per-vertex discrete levels [S63, unofficial]. Voxel Neighborhood 3D Textures hold voxel chunk data in a 3D texture. A compute shader evaluates the local density field within a small spherical kernel (e.g., a 3x3x3 or 5x5x5 window) around each surface point [S63, unofficial]. If the procedural generation pipeline generates a distance field rather than raw block binaries, perform short-range cone tracing through the local SDF to provide smooth, realistic contact shadows baked directly into the output mesh buffers during the surface extraction pass (such as Marching Cubes or Surface Nets) [S63, unofficial].

#### VIPER Ray-Traversal and Surface Intersection Algorithm
The relative distance is calculated as $(target - value1) / (value2 - value1)$ [S64, unofficial]. Time complexity is O(p * r), where p is the number of pixels and r is the volume's largest dimension in voxels. Space complexity is dependent on the size of the data volume [S64, unofficial].

## Background
Distance-field ray marching, often referred to as sphere tracing, was popularized by Jim Blinn in his 1982 paper "Blinn's Box" and later formalized by Kevin Goldsmith and Paul Heckbert in "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" (1990) [S26]. The technique gained widespread attention in the computer graphics community through the work of Inigo Quilez, who demonstrated its potential for real-time procedural rendering in fragment shaders [S1, S11, unofficial]. It has since become a standard technique for rendering implicit surfaces, fractals (like Mandelbulbs), and procedural geometry in real-time graphics, particularly in demoscene productions and WebGL applications [S4, S16, S53, unofficial].

## Sources

[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S4] larsbrinkhoff/glsl-sphere-tracing, https://github.com/larsbrinkhoff/glsl-sphere-tracing, fetched 2026-10-26 (unofficial)
[S5] devnio/Optimized-CPU-Ray-Marcher, https://github.com/devnio/Optimized-CPU-Ray-Marcher, fetched 2026-10-26 (unofficial)
[S6] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S11] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S14] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S26] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S35] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S54] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S55] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S56] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
[S57] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S58] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S59] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S60] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S61] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S62] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S63] Baking Dynamic Ambient Occlusion in Procedural Voxels, https://salivity.github.io/game-development/article/baking-dynamic-ambient-occlusion-in-procedural-voxels, fetched 2026-10-29 (unofficial)
[S64] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
