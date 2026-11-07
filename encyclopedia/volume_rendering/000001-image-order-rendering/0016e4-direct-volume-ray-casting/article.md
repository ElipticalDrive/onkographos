# Direct volume ray casting

Also called: Ray casting, Backward ray marching, Image-order volume rendering

## Overview
Direct volume ray casting is an image-order rendering technique that synthesizes a 2D image from 3D scalar volumetric data by casting a ray from the camera through each pixel into the volume and integrating optical properties along the ray path [S3, S5, S11, unofficial]. It solves the problem of visualizing semi-transparent, internal structures without generating intermediate polygonal meshes, allowing all voxels to potentially contribute to the final image [S4, S9, unofficial]. What sets it apart from its siblings, such as [Ray-grid traversal](17-ray-grid-traversal.md) (which uses fixed voxel-aligned templates) and [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md) (which stops at a threshold), is its continuous integration of emission and absorption across the entire ray segment using front-to-back compositing [S1, S11, S13, unofficial].

## Shared principles
All direct volume ray casting methods share the fundamental loop structure defined in [Image-order rendering](16-image-order-rendering.md): for each pixel, a ray is defined in world space, intersected with the volume domain, and sampled/composited along its path [G]. The ray is parameterized as $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the direction [G]. The traversal order is determined by the pixel's position in the image plane [G].

The core computational engine consists of four sequential stages applied to each ray [S5, S10, S37, unofficial]:

1.  **Ray Generation and Discretization**: The ray segment $[t_{start}, t_{end}]$ intersecting the volume is determined, typically by intersecting the ray with the volume's bounding box [S1, S8, S16, unofficial]. The segment is discretized into $N$ samples at equidistant intervals $\Delta t = (t_{end} - t_{start}) / N$ [S5, S10, S17, unofficial].
2.  **Sampling and Interpolation**: At each sample point $\mathbf{p}_i = r(t_i)$, the scalar value $d_i$ is retrieved from the volume data structure [S5, S10, unofficial]. This value is obtained by interpolating neighboring voxel values using a reconstruction kernel, most commonly trilinear interpolation of the eight surrounding voxels [S7, S10, S43, unofficial].
3.  **Classification (Transfer Function)**: The scalar value $d_i$ is mapped to optical properties: color $C_i$ and opacity $\alpha_i$ [S5, S10, S13, unofficial]. This mapping is performed by a transfer function $T(d_i) \rightarrow (C_i, \alpha_i)$ [S2, S2789]. The opacity is often corrected for the step size to account for the discretization error, using the relation $\alpha'_i = 1 - (1 - \alpha_i)^{\Delta t}$ or similar approximations [S11, S32, unofficial].
4.  **Compositing**: The optical properties are accumulated into the final pixel color using the front-to-back volume rendering equation [S1, S13, unofficial][S56]. For each sample $i$ from front ($t_{start}$) to back ($t_{end}$), the accumulated color $C_{acc}$ and accumulated opacity $\alpha_{acc}$ are updated as follows:
    $$
    C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i C_i
    $$
    $$
    \alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i
    $$
    where $C_{acc}^{(0)} = 0$ and $\alpha_{acc}^{(0)} = 0$ [S12, S43, unofficial][S50, abstract only, unofficial]. The term $(1 - \alpha_{acc}^{(i-1)})$ represents the transmittance, i.e., the fraction of light reaching sample $i$ from the camera side [S1, S51, unofficial]. The final pixel color is often composited over a background color $C_{bg}$ using $C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}$ [S1, S14, unofficial].

Early ray termination is a common optimization where the loop breaks if $\alpha_{acc}$ exceeds a threshold (e.g., 1.0), as further samples contribute negligibly to the color [S10, S12, unofficial].

## Subtopics

### [Neural volume rendering](1576-neural-volume-rendering.md)
Replaces discrete texture lookups with neural network queries for continuous implicit representation. Choose for unstructured/sparse data.

### [Temporal Accumulation (Closest Hit)](2080-temporal-accumulation-closest-hit.md)
Stabilizes stochastic results by blending with history. Choose for noisy/stochastic rendering.

### [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md)
Renders multiple aligned volumes in a single pass. Choose for multi-modal data (CT/MRI).

### [Compositing and Blending](2515-compositing-and-blending.md)
Defines the optical integration models (front-to-back, back-to-front, MIP). Choose for specific visual effects.

### [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md)
Stops at the first threshold hit. Choose for surface-like visualization without meshing.

### [Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md)
Defines reconstruction kernels (trilinear, nearest-neighbor). Choose to balance quality vs. speed.

### [Transfer Function and Opacity Mapping](2789-transfer-function-and-opacity-mapping.md)
Maps scalar values to RGBA. Choose to control visibility of structures.

### [Forward Mapping](2913-forward-mapping.md)
Projects voxels to screen (object-order). Choose for hardware accumulation buffer acceleration.

### [Doppler Volume Rendering](3536-doppler-volume-rendering.md)
Simulates relativistic Doppler shifts. Choose for velocity fields.

### [Multiresolution Volume Rendering](3541-multiresolution-volume-rendering.md)
Uses hierarchical data (wavelets). Choose for large datasets with adaptive detail.

### [Parallel Domain-Decomposed Ray Casting](3625-parallel-domain-decomposed-ray-casting.md)
Distributes volume across nodes. Choose for massive datasets exceeding single-GPU memory.

### [Layered Volume Rendering with Light Propagation](4083-layered-volume-rendering-with-light-propagation.md)
Processes planes instead of rays. Choose for efficient light transport.

### [Volume Raycasting Output Generation](4277-volume-raycasting-output-generation.md)
Handles color space conversion and NDC depth. Choose for standard pipeline integration.

### [Volumetric Rendering with Spherical Harmonics](4962-volumetric-rendering-with-spherical-harmonics.md)
Uses SH coefficients for view-dependent color. Choose for anisotropic scattering.

### [Pre-integrated Volume Rendering](5327-pre-integrated-volume-rendering.md)
Pre-computes integration over segments. Choose for low-res data with non-linear transfer functions.

### [Two-pass and Multi-pass Techniques](6224-two-pass-and-multi-pass-techniques.md)
Decouples light/geometry from compositing. Choose for complex lighting or hardware limits.

### [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md)
Rasterizes proxy geometry. Choose for efficient texture mapping hardware.

### [Projection and MIP Techniques](6305-projection-and-mip-techniques.md)
Uses order-independent aggregation (max/min). Choose for emphasizing high-intensity structures.

### [WebGL Multi-pass Ray-casting with 3D Geometry Intersection](6575-webgl-multi-pass-ray-casting-with-3d-geometry-intersection.md)
Handles 3D meshes in WebGL. Choose for web apps with mixed geometry.

### [Programmable Transfer Function DVR](8220-programmable-transfer-function-dvr.md)
Computes TF in shader. Choose for interactive TF customization.

### [Hybrid Ray Tracing for Geometric/Volumetric Mixing](8261-hybrid-ray-tracing-for-geometric-volumetric-mixing.md)
Combines polygons and volumes. Choose for synthetic objects in volumes.

### [Lighting and Shading Techniques](8281-lighting-and-shading-techniques.md)
Adds Phong shading via gradients. Choose for depth cues.

### [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md)
Core integration engine. Choose for custom sampling strategies.

### [WebGL Path Tracing Volume Rendering](8592-webgl-path-tracing-volume-rendering.md)
Implements path tracing in WebGL. Choose for realistic lighting in browsers.

### [Temporal and Procedural Techniques](9442-temporal-and-procedural-techniques.md)
Generates data procedurally. Choose for dynamic effects (smoke/fire).

### [Specific Applications and Integrators](9443-specific-applications-and-integrators.md)
Domain-specific integrators. Choose for medical/manufacturing.

### [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md)
Offloads to GPU. Choose for real-time performance.

### Svox1 Rendering Formula (Nearest-Neighbor)
Implements the Svox1 rendering formula using nearest-neighbor interpolation with infinitely many steps. Includes forward, backward, and fused variants [S65, unofficial]. The core function is `volume_render_svox1(SparseGridSpec &, RaysSpec &, RenderOptions &)` [S65, unofficial].

### Voronoi Painter's Algorithm Compositing
A rendering compositing strategy where the scene is rendered part-by-part (one Voronoi cell at a time) and the resulting cell images are composited back-to-front to generate the final image. It relies on the proof that Voronoi cells admit a partial ordering compatible with occlusion [S66]. The ordering is defined as $V_{P'} <_Q V_P$ iff $d(P', Q) < d(P, Q)$ [S66]. Unlike the standard Painter's Algorithm, which fails on cyclic occlusion graphs, Voronoi Painter's is always compatible because the distance-to-camera ordering is a strict partial order [S66].

### Quadratic Approximation for Exponential Blending
A technique to approximate the exponential light attenuation and blending function in volume rendering with a quadratic function to allow efficient hardware implementation using Gouraud shading, while maintaining accuracy for small cell depths [S67]. The target exponential function is $f(z) = 1 - e^{-\tau z}$, where $\tau$ is the opacity-per-unit-distance and $z$ is the depth. The approximation is $g(z) = (c_0 + c_1 z)(a_0 + a_1 z)$, with the first coefficient $a_0$ set to 0 to simplify the blend function structure [S67].

### Topology-Based Direct Volume Rendering
A method applying topological segmentation to scalar fields to guide direct volume rendering attributes (opacity, color), rather than just indirect mesh visualization [S68, unofficial]. It uses a segmentation algorithm (e.g., contour tree or volume skeleton tree) to identify boundaries between objects in the scalar field topology, then applies traditional direct volume rendering techniques while assigning rendering attributes based on topological attributes of the identified objects [S68, unofficial].

### Isosurface-Accelerated Ray Casting with Weighted Last Sample
An optimization for ray casting that uses the extracted isosurface as an opaque barrier to terminate rays early, rendering the isosurface's 3D texture coordinates into the exit point texture [S69]. To counteract the last incorrect sample (inside the surface), it is weighted by the fraction of the step that is outside of the surface [S69]. A pitfall of straightforward integration is aliasing/banding artifacts because the isosurface coordinates may not lie exactly on the sample point of a marching ray, causing the last incorrect sample to be fully included [S69].

## Background
The term "direct volume rendering" was coined by Marc Levoy in his seminal 1988 paper, "Display of Surfaces from Volume Data" [S9, S10, unofficial]. Levoy defined the volume rendering pipeline, which became the influential framework for subsequent techniques [S9, unofficial]. The first volume rendering method based on a ray casting mechanism was developed by Kajiya [S10, unofficial]. The method has been extensively optimized for graphics hardware, with early GPU implementations appearing in 2003 by Roettger et al. [S6, unofficial]. It is widely used in medical imaging (CT/MRI visualization), scientific visualization, and computer graphics for rendering complex semi-transparent phenomena [S5, S9, S10, unofficial].

## Sources
[S1] The Ray-Marching Algorithm, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/ray-marching-algorithm.html, fetched 2026-10-25 (unofficial)
[S2] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S3] Volume Ray Casting Techniques, https://www.scribd.com/document/102700078/03-raycasting, fetched 2026-10-25 (unofficial)
[S4] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S5] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S6] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S7] Shear-warp factorization volume rendering visualization system, https://archive.org/details/0162018299360, fetched 2026-10-26 (unofficial)
[S8] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S9] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S10] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S11] palasjir/ct-volumetric-rendering, https://github.com/palasjir/ct-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S12] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S13] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26 (unofficial)
[S14] khizarsiddiqui/Volume-Rendering, https://github.com/khizarsiddiqui/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S15] RuolinZheng08/volume-rendering, https://github.com/RuolinZheng08/volume-rendering, fetched 2026-10-26 (unofficial)
[S16] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S17] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S18] Visualization Lecture 09: VolVis Part 3-Direct Volume Rendering, Ray and Transfer Functions, https://www.youtube.com/watch?v=FYAJQgYdeqQ, fetched 2026-10-27 (unofficial)
[S19] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S20] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S21] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S22] gillesferrand/Unity-RayTracing, https://github.com/gillesferrand/Unity-RayTracing, fetched 2026-10-26 (unofficial)
[S23] Dy111111/volumeRenderingGPU, https://github.com/Dy111111/volumeRenderingGPU, fetched 2026-10-26 (unofficial)
[S24] vulekhanh/VolumeRendering, https://github.com/vulekhanh/VolumeRendering, fetched 2026-10-26 (unofficial)
[S25] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S26] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S27] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S28] hands133/opengl-directVolumeRenderer, https://github.com/hands133/opengl-directVolumeRenderer, fetched 2026-10-26 (unofficial)
[S29] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S30] thePZH/VolumeRendering-OpenGL, https://github.com/thePZH/VolumeRendering-OpenGL, fetched 2026-10-26 (unofficial)
[S31] manishaa1704/Interactive-DVR-System, https://github.com/manishaa1704/Interactive-DVR-System, fetched 2026-10-26 (unofficial)
[S32] GitHub - LDeakin/VkVolume: Vulkan-based volume renderer with efficient ..., https://github.com/LDeakin/VkVolume, fetched 2026-10-26 (unofficial)
[S33] Volume Rendering With Ray Casting - WPI, https://web.cs.wpi.edu/~matt/courses/cs563/talks/powwie/p1/ray-cast.htm, fetched 2026-10-26 (unofficial)
[S34] PDF VOLUME RENDERING BY RAY CASTING - Warse, https://www.warse.org/IJATCSE/static/pdf/file/icetetssp47.pdf, fetched 2026-10-30 (unofficial)
[S35] 20 volume rendering (using glsl) with ray casting algorithm, https://stackoverflow.com/questions/9482572/volume-rendering-using-glsl-with-ray-casting-algorithm?rq=4, fetched 2026-10-26 (unofficial)
[S36] Guide for beginners: What is Volume Ray Casting? - VisCircle, https://viscircle.com/guide-for-beginners-what-is-volume-ray-casting/, fetched 2026-10-27 (unofficial)
[S37] Volume Ray Casting Techniques and Applications using ..., https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S38] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S39] 3D: ray-cast volume rendering instead of a stack of slices, https://github.com/jamescarruthers/quick_dicom/pull/4, fetched 2026-10-27 (unofficial)
[S40] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
[S41] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
[S42] hwenchi/visible-human-volume, https://github.com/hwenchi/visible-human-volume, fetched 2026-10-27 (unofficial)
[S43] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S44] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
[S45] vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, https://github.com/vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, fetched 2026-10-27 (unofficial)
[S46] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S47] overhaul88/Parallel_Volume_Rendering, https://github.com/overhaul88/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S48] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S49] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S50] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
[S51] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
[S52] WebGL Volume Rendering made simple - Lebarba, https://www.barbaleon.co.uk/webgl/index.html, fetched 2026-10-28 (unofficial)
[S53] (PDF) GPU-Based Volume Rendering for Medical Imagery - ResearchGate, https://www.researchgate.net/publication/228912978_GPU-Based_Volume_Rendering_for_Medical_Imagery, fetched 2026-10-28 (unofficial)
[S54] Home, https://graphics.cg.uni-saarland.de/competition/cg1-2018/original/groupR_web/, fetched 2026-10-28 (unofficial)
[S55] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S57] (PDF) Differentiable Direct Volume Rendering - ResearchGate, https://www.researchgate.net/publication/353510271_Differentiable_Direct_Volume_Rendering, fetched 2026-10-30 (unofficial)
[S58] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
[S59] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S60] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S63] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
[S64] Volumetric Rendering - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/volumetric-rendering/, fetched 2026-10-30 (unofficial)
[S65] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S66] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
[S67] Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering, https://ntrs.nasa.gov/api/citations/19930015755/downloads/19930015755.pdf, fetched 2026-10-24
[S68] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
[S69] Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline, https://doi.org/10.1145/1281500.1361219, fetched 2026-11-03
