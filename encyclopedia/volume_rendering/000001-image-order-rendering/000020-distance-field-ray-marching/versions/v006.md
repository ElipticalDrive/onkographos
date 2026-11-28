# Distance-field ray marching

Also called: Sphere tracing, SDF ray marching, Distance field rendering

## Overview
Distance-field ray marching is an image-order rendering technique that finds surface intersections with implicit geometry defined by Signed Distance Functions (SDFs) by iteratively stepping along a camera ray [S1, S2, S6, unofficial]. It solves the problem of rendering complex, procedural, or implicit surfaces without explicit polygonal meshes by using the SDF value at the current ray position as a guaranteed safe step size, ensuring the ray never overshoots a surface [S8, S50, unofficial]. It sets itself apart from [Direct volume ray casting](19-direct-volume-ray-casting.md) by targeting zero-level sets (surfaces) rather than integrating volumetric density, and from [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md) by using adaptive step sizes derived from the distance field rather than fixed steps or analytical root-finding [S1, S27, unofficial].

## Shared principles
The method relies on the mathematical property of the Signed Distance Function (SDF). An SDF $f(\mathbf{p})$ maps a 3D point $\mathbf{p}$ to the shortest Euclidean distance to the nearest surface of the scene geometry [S2, S5, unofficial]. The sign indicates position: $f(\mathbf{p}) < 0$ inside, $f(\mathbf{p}) = 0$ on the surface, and $f(\mathbf{p}) > 0$ outside [S2, S5, S9643].

The core algorithm, often called sphere tracing, proceeds as follows for each pixel:
1.  **Ray Definition**: A ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector [G].
2.  **Initialization**: The accumulated distance $t$ is initialized to 0 (or a near-plane value) [S1, S3, unofficial].
3.  **Marching Loop**:
    *   Evaluate the scene SDF at the current ray position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ to get the distance $d = f(\mathbf{p})$ [S1, S34, unofficial].
    *   **Termination Check**: If $d < \epsilon$ (a small convergence threshold, e.g., $0.001$), the ray has hit the surface [S1, S39, unofficial]. If $t > t_{max}$ (maximum ray distance), the ray has escaped [S1, S34, unofficial].
    *   **Step**: Advance the ray by the distance $d$: $t \leftarrow t + d$ [S1, S33, unofficial]. This step is "safe" because the SDF guarantees no geometry exists within distance $d$ of $\mathbf{p}$ [S50, unofficial].
4.  **Shading**: Upon hit, the surface normal $\mathbf{n}$ is estimated, typically via finite differences of the SDF: $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [S2055, S4619]. Lighting is then computed [S1, unofficial].

Scene composition is achieved by combining primitive SDFs using Boolean operations: Union is $\min(d_1, d_2)$, Intersection is $\max(d_1, d_2)$, and Subtraction is $\max(-d_1, d_2)$ [S35, S38, unofficial]. Transformations (rotation, translation, scaling) are applied to the query point $\mathbf{p}$ before evaluation or to the SDF function itself [S3850].

## Subtopics

### [Sphere Tracing and Step Rules](9643-sphere-tracing-and-step-rules.md)
The foundational algorithm defining the iterative step rule $t_{i+1} = t_i + \text{scene}(p)$ and convergence criteria.

### [SDF Primitive Definitions](1578-sdf-primitive-definitions.md)
Analytical building blocks (spheres, boxes, planes) that return the signed Euclidean distance to geometric primitives.

### [Raymarching with Depth Buffer Occlusion](1880-raymarching-with-depth-buffer-occlusion.md)
Integrates sphere tracing with hardware depth buffers to allow SDF objects to occlude and be occluded by mesh geometry.

### [SDF Depth Buffer Integration](1955-sdf-depth-buffer-integration.md)
Extends sphere tracing by explicitly writing computed hit depths to the hardware depth buffer (`SV_Depth`) for correct occlusion ordering.

### [Ray Marching with Reflections](1963-ray-marching-with-reflections.md)
Extends the method by continuing the ray path after intersection to simulate specular reflections, transitioning to raytracing.

### [Lighting and Shading Models](2055-lighting-and-shading-models.md)
Computes final color using normals derived from the SDF gradient and light source parameters [G].

### [Ray Marching with Neural SDF](2409-ray-marching-with-neural-sdf.md)
Replaces analytical SDF queries with neural network inference to render learned implicit shapes.

### [Surface Point Sampling for SDFs](2708-surface-point-sampling-for-sdfs.md)
Generates 3D points near the zero-level set for neural rendering or reconstruction pipelines.

### [Recursive Ray Marching with Early Stopping](3546-recursive-ray-marching-with-early-stopping.md)
Parallelization technique using dynamic masks to skip converged rays in batch processing.

### [Deformation and Transformation Handling](3850-deformation-and-transformation-handling.md)
Applies spatial transformations (rigid, scaling, deformation) to SDFs during evaluation [G].

### [Lidar Sphere Tracing](3889-lidar-sphere-tracing.md)
Adapts ray marching to simulate LiDAR sensor data by projecting surface intersections as sparse point clouds [G].

### [Neon Raymarcher](4157-neon-raymarcher.md)
Real-time WebGL2 renderer for procedural synthwave cities using SDFs in a single fragment shader.

### [Scale-Invariant Integral Surface Rendering](4266-scale-invariant-integral-surface-rendering.md)
Efficiently renders integral surfaces with varying radii using scale-invariant step rules to reduce grazing ray steps.

### [Primary Ray Integrator](4401-primary-ray-integrator.md)
Composite method resolving primary rays by sequentially tracing terrain, water, sky, and clouds.

### [SDF Normal Estimation via Finite Differences](4619-sdf-normal-estimation-via-finite-differences.md)
Estimates surface normals by numerically computing the SDF gradient using central finite differences.

### [Antialiasing (AA)](4802-antialiasing-aa.md)
Prevents moiré artifacts in procedural SDFs using subpixel sampling or derivative-based step adjustments.

### [Volume Ray Marching with Temporal Accumulation](4804-volume-ray-marching-with-temporal-accumulation.md)
Renders volumetric data (smoke, fire) by blending ray-marched results with previous frames.

### [Volumetric and Atmospheric Rendering](4968-volumetric-and-atmospheric-rendering.md)
Renders participating media by integrating light transport along rays through density fields [G].

### [Acceleration and Optimization](6222-acceleration-and-optimization.md)
Reduces computational cost via culling, resolution scaling, over-relaxation, and beam optimization [G].

### [Dynamic SDF Ray Marching with Structured Buffers](7434-dynamic-sdf-ray-marching-with-structured-buffers.md)
Streams dynamic object data from CPU to GPU for per-pixel SDF evaluation without hardcoding [G].

### [Distance Field Representations and Sampling](8725-distance-field-representations-and-sampling.md)
Covers efficient querying of implicit surfaces and volumetric properties without meshes [G].

### [RBF Particle Volume Path Tracing](9641-rbf-particle-volume-path-tracing.md)
Path traces particle volumes modeled as additive radial basis functions (Gaussians) [G].

### [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md)
Evaluates full Gaussian mixtures along rays to handle primitive overlap and self-attenuation, differing from standard 3DGS.

### [Terrain Raymarching](1357-terrain-raymarching.md)
Renders 2D heightmap terrains by stepping rays until height drops below the terrain surface.

### Spatially Varying Threshold
Uses FBM to define isosurfaces $SDF(p) + B_n(p) = 0$, approximating the spatially varying threshold $v(p)$ via Taylor expansion to allow standard acceleration [S55].

### ST-Fallback Focus Sphere
Modulates per-sample alpha using a focus sphere kernel for focus+context visualization when WebGL2 atomic support is unavailable [S56, unofficial].

### Ray Subdivision with Polynomial Interpolation for SCALIS Fields
Subdivides rays based on local minima of homothetic distance fields and uses quadratic interpolation to find isosurface crossings, avoiding brute-force evaluation [S57].

### Blinn Soft Objects Ray Tracing
Applies sphere-tracing after mapping to compute signed distance approximations to density fields for Blinn soft objects [S57].

### Span-Based Wing Un-bending
Renders thin flapping sheets by rotating points back by local flap angle to evaluate a flat 2D shape in the span-invariant plane [S58, unofficial].

### Tri-planar procedural materials
Applies world-projected textures with micro-grain layers continuous across voxel boundaries [S59, unofficial].

### Local Distance Field Ambient Occlusion
Offloads meshing and AO to compute shaders, evaluating local density fields or performing short-range cone tracing through SDFs for smooth contact shadows [S60, unofficial].

### X-Ray Visualization
Computes average density values along rays to visualize data as grayscale images similar to medical X-rays [S61].

### Augmented Sphere Tracing
Combines hybrid implicit lattice and mesh shell representations, using local tracing distance queries within periodic cells to avoid costly SDF calculations [S62, abstract only, unofficial].

### Ray Tracing Deterministic 3-D Fractals
Applies ray-tracing to 3-D deterministic fractals (Julia sets) using frame buffer storage, replacing boundary-tracking algorithms [S49, unofficial].

## Background
Distance-field ray marching, specifically sphere tracing, was introduced by John C. Hart in 1994 in the paper "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" [S26][S49, unofficial]. It was developed to render implicit surfaces (such as fractals and complex Boolean combinations) without the overhead of explicit mesh generation or the precision issues of fixed-step ray marching. The method gained significant popularity in the demoscene and real-time graphics communities in the 2010s through shader-based implementations by artists like Inigo Quilez, who popularized the use of SDFs for procedural geometry [S1, S2, S27, unofficial]. It is widely used in real-time rendering, procedural generation, and implicit surface visualization [S1, S5, S12, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S3] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S4] larsbrinkhoff/glsl-sphere-tracing, https://github.com/larsbrinkhoff/glsl-sphere-tracing, fetched 2026-10-26 (unofficial)
[S5] devnio/Optimized-CPU-Ray-Marcher, https://github.com/devnio/Optimized-CPU-Ray-Marcher, fetched 2026-10-26 (unofficial)
[S6] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S7] AustinGoodman/non-linear-sphere-tracing, https://github.com/AustinGoodman/non-linear-sphere-tracing, fetched 2026-10-26 (unofficial)
[S8] gqvz/SphereTracing, https://github.com/gqvz/SphereTracing, fetched 2026-10-26 (unofficial)
[S9] haydenparker3/3D-Sphere-Tracing, https://github.com/haydenparker3/3D-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S10] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S11] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S12] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
[S13] gokselgoktas/sdf-ray-marching, https://github.com/gokselgoktas/sdf-ray-marching, fetched 2026-10-26 (unofficial)
[S14] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S15] MosheBenZacharia/Unity_DistanceFieldRaymarching, https://github.com/MosheBenZacharia/Unity_DistanceFieldRaymarching, fetched 2026-10-26 (unofficial)
[S16] ZeusYang/2DRayMarchingRenderer, https://github.com/ZeusYang/2DRayMarchingRenderer, fetched 2026-10-26 (unofficial)
[S17] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S18] ChadMcKinney/Dirac-Sea-Engine, https://github.com/ChadMcKinney/Dirac-Sea-Engine, fetched 2026-10-26 (unofficial)
[S19] agill-dev-0/SDF-ray-marcher, https://github.com/agill-dev-0/SDF-ray-marcher, fetched 2026-10-26 (unofficial)
[S20] eolicneb/Distance_Estimator_render, https://github.com/eolicneb/Distance_Estimator_render, fetched 2026-10-26 (unofficial)
[S21] el-bakkali/manim-glsl-jellyfish, https://github.com/el-bakkali/manim-glsl-jellyfish, fetched 2026-10-26 (unofficial)
[S22] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
[S23] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S24] Add OpenGL GPU ray marching renderers for Intel HD Graphics 630 with Jump Flood Algorithm optimization, https://github.com/rhythmatician/phyllotaxis-torus/pull/1, fetched 2026-10-26 (unofficial)
[S25] Ray Marching: Signed Distance Fields, Sphere Tracing, and SDF Shading, https://tutorials.tektite.studio/ray-marching/, fetched 2026-10-27 (unofficial)
[S26] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S27] Sphere Tracing Basics | ronja-tutorials/ShaderTutorials | DeepWiki, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5.1-sphere-tracing-basics, fetched 2026-10-27 (unofficial)
[S28] EmmetOT/SignedDistanceFun, https://github.com/EmmetOT/SignedDistanceFun, fetched 2026-10-27 (unofficial)
[S29] zesterer/sdf-test, https://github.com/zesterer/sdf-test, fetched 2026-10-27 (unofficial)
[S30] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S31] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S32] P41NT/CUDA_Raymarcher, https://github.com/P41NT/CUDA_Raymarcher, fetched 2026-10-27 (unofficial)
[S33] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S34] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S35] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S36] Ray Marching - Rendering Without Triangles Using Distance Fields | KipuHub, https://www.kipuhub.com/en/article/ray-marching, fetched 2026-10-27 (unofficial)
[S37] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S38] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S39] desttinghim/kha-sdf, https://github.com/desttinghim/kha-sdf, fetched 2026-10-27 (unofficial)
[S40] hassony2/pyrender_sdf, https://github.com/hassony2/pyrender_sdf, fetched 2026-10-27 (unofficial)
[S41] xms0g/terrain, https://github.com/xms0g/terrain, fetched 2026-10-27 (unofficial)
[S42] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S43] conradplake/ray-marching-shader, https://github.com/conradplake/ray-marching-shader, fetched 2026-10-27 (unofficial)
[S44] AshMagorian/RayMarching, https://github.com/AshMagorian/RayMarching, fetched 2026-10-27 (unofficial)
[S45] Dionysos199/Ray-marching, https://github.com/Dionysos199/Ray-marching, fetched 2026-10-27 (unofficial)
[S46] Assets/043_SphereTracingBasics/SphereTracingBasics.shader, https://github.com/ronja-tutorials/ShaderTutorials/blob/31899f7b/Assets/043_SphereTracingBasics/SphereTracingBasics.shader, fetched 2026-10-27 (unofficial)
[S47] Assets/044_SphereTracingShading/SphereTracingShading.shader, https://github.com/ronja-tutorials/ShaderTutorials/blob/31899f7b/Assets/044_SphereTracingShading/SphereTracingShading.shader, fetched 2026-10-27 (unofficial)
[S48] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S49] (PDF) Accelerating Sphere Tracing - ResearchGate, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S50] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S51] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
[S52] Potatro, RayMarching and DistanceFields : a story of SphereTracing | xoofx, https://xoofx.github.io/blog/2009/10/12/potatro-and-raymarching-story-of.html, fetched 2026-10-29 (unofficial)
[S53] Raymarching - by Xor - GM Shaders, https://mini.gmshaders.com/p/gm-shaders-mini-raymarching-1351092, fetched 2026-10-30 (unofficial)
[S54] TEA STORM by @p01, http://www.p01.org/tea_storm/, fetched 2026-10-30 (unofficial)
[S55] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S56] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
[S57] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S58] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S59] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S60] Baking Dynamic Ambient Occlusion in Procedural Voxels, https://salivity.github.io/game-development/article/baking-dynamic-ambient-occlusion-in-procedural-voxels, fetched 2026-10-29 (unofficial)
[S61] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
[S62] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
