# Direct volume ray casting

Also called: DVR, ray marching, front-to-back compositing, image-order volume rendering

## Overview
Direct volume ray casting (DVR) is an image-order rendering technique that synthesizes a 2D image from a 3D scalar volume by casting a ray through each pixel, sampling the volume along the ray, and integrating the optical properties (color and opacity) using the volume rendering integral [S1][S4, S13, S14, unofficial]. It solves the problem of visualizing the internal structure of semi-transparent volumetric data without extracting explicit geometry, allowing all voxels to potentially contribute to the final image [S5, S12, unofficial]. Unlike surface-based methods (e.g., [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md)) that stop at the first opaque boundary, DVR accumulates contributions from the entire ray segment within the volume, preserving depth cues and transparency [S12, S23, unofficial].

## Shared principles
All direct volume ray casting methods share the following core mechanism:

1.  **Ray Definition**: For each pixel $(x, y)$ in the image plane, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the viewing direction [G]. The ray is intersected with the volume bounding box to determine the entry parameter $t_{near}$ and exit parameter $t_{far}$ [S2, S10, S20, unofficial].
2.  **Sampling**: The ray segment $[t_{near}, t_{far}]$ is discretized into $N$ samples at positions $t_i = t_{near} + i \cdot \Delta t$, where $\Delta t = (t_{far} - t_{near}) / N$ [S2, unofficial][S77]. At each sample position $\mathbf{p}_i = r(t_i)$, the scalar volume value $V(\mathbf{p}_i)$ is retrieved, typically via trilinear interpolation of the surrounding voxels [S9, S12, S53, unofficial].
3.  **Classification (Transfer Function)**: The scalar value $V(\mathbf{p}_i)$ is mapped to optical properties: a color vector $\mathbf{c}_i$ (RGB) and an opacity scalar $\alpha_i$ (A) [S25, S2789]. This mapping is defined by a transfer function $T(V) \rightarrow (\mathbf{c}, \alpha)$ [S2, S2789]. The opacity is often corrected for the step size $\Delta t$ to approximate continuous attenuation, using models such as Beer-Lambert: $\alpha_i = 1 - e^{-\sigma_i \Delta t}$, where $\sigma_i$ is the extinction coefficient derived from $V(\mathbf{p}_i)$ [S26, S33, S39, unofficial].
4.  **Compositing (Integration)**: The samples are composited into the final pixel color $\mathbf{C}_{out}$ and accumulated opacity $\alpha_{out}$ using the front-to-back volume rendering equation [S17, S43, unofficial][S68]:
    $$
    \mathbf{C}_{out} = \sum_{i=1}^{N} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    $$
    \alpha_{out} = 1 - \prod_{i=1}^{N} (1 - \alpha_i)
    $$
    In iterative form, for each sample $i$:
    $$
    \mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \mathbf{c}_i \cdot \alpha_i
    $$
    $$
    \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_i
    $$
    where $\mathbf{C}_{acc}$ and $\alpha_{acc}$ are initialized to $(0, 0)$ [S17, S58, S67, unofficial].
5.  **Shading**: At each sample, a shading model (e.g., Phong, Blinn-Phong) may be applied to compute the base color $\mathbf{c}_i$ based on the local gradient $\nabla V(\mathbf{p}_i)$, which serves as the surface normal [S12, S19, S41, unofficial].

## Subtopics

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
Generates a 2D projection by processing 3D data as 2D slices, applying transfer functions, and compositing. Slower than direct ray casting due to slice-based overhead.

### [Neural volume rendering](1576-neural-volume-rendering.md)
Uses a neural network (MLP) to predict density and color along rays, replacing explicit voxel sampling with differentiable function evaluation.

### [Temporal Accumulation (Closest Hit)](2080-temporal-accumulation-closest-hit.md)
Stabilizes noisy results by blending current frames with history, prioritizing the closest geometric hit to preserve depth boundaries.

### [Compositing and Blending](2515-compositing-and-blending.md)
The final accumulation stage integrating optical properties along the ray using the over operator.

### [Energy Region Compositing](2717-energy-region-compositing.md)
Composites multiple energy-specific volumes within a region of interest defined by a magnitude volume, using intensity thresholds.

### [Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md)
Methods for retrieving scalar values from discrete grids (e.g., trilinear, nearest-neighbor) to mitigate aliasing.

### [Transfer Function and Opacity Mapping](2789-transfer-function-and-opacity-mapping.md)
Maps scalar data to color and opacity, enabling the visualization of internal structures by assigning transparency.

### [Forward Mapping](2913-forward-mapping.md)
Projects voxels directly onto the image plane, relying on data traversal order for hidden surface removal, contrasting with image-order ray casting.

### [Doppler Volume Rendering](3536-doppler-volume-rendering.md)
Simulates the Doppler effect on light for moving volumetric matter, integrating spectral radiance instead of scalar opacity.

### [Parallel Domain-Decomposed Ray Casting](3625-parallel-domain-decomposed-ray-casting.md)
Accelerates rendering by dividing the volume among multiple nodes, each casting rays independently.

### [Layered Volume Rendering with Light Propagation](4083-layered-volume-rendering-with-light-propagation.md)
Processes parallel planes (layers) instead of individual rays, propagating light and view rays across layers.

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Accelerates ray casting by mapping data to GPU textures and sampling via rasterized proxy geometry.

### [Volumetric Rendering with Spherical Harmonics](4962-volumetric-rendering-with-spherical-harmonics.md)
Represents view-dependent properties using SH coefficients for efficient rendering of complex light-material interactions.

### [Customization Volume Integration](5291-customization-volume-integration.md)
Integrates a pre-computed volume to apply position-dependent modifications (e.g., gamma correction) to visualization parameters.

### [Pre-integrated Volume Rendering](5327-pre-integrated-volume-rendering.md)
Pre-computes integration of transfer functions over ray segments in a lookup table to reduce aliasing and improve performance.

### [Two-pass and Multi-pass Techniques](6224-two-pass-and-multi-pass-techniques.md)
Decomposes the workload into multiple passes (e.g., light attenuation vs. color compositing) to decouple computational stages.

### [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md)
Converts volume to 2D polygonal slices rasterized and composited by standard graphics hardware.

### [Projection and MIP Techniques](6305-projection-and-mip-techniques.md)
Projects data by aggregating scalar values (e.g., maximum intensity) along rays, discarding depth-dependent opacity accumulation.

### [Volume Ray-Casting on Graphics Hardware](7443-volume-ray-casting-on-graphics-hardware.md)
Implements ray casting on programmable GPUs (e.g., DirectX 9) using multi-pass stream models.

### [View-Dependent Transparency](8017-view-dependent-transparency.md)
Adjusts opacity based on the angle between the viewing direction and surface normal to reduce occlusion.

### [Programmable Transfer Function DVR](8220-programmable-transfer-function-dvr.md)
Computes transfer function values via shader functions at each sample, trading memory fetches for computation.

### [Lighting and Shading Techniques](8281-lighting-and-shading-techniques.md)
Modulates base color with lighting terms (e.g., Phong) computed from local gradients for depth perception.

### [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md)
Defines the computational framework for iterating along rays, including step size strategies and integration methods.

### [Cutting Plane Integration](8414-cutting-plane-integration.md)
Dynamically slices the volume with a clipping plane to reveal internal structures occluded by front-facing regions.

### [Topology-Based Direct Volume Rendering](8480-topology-based-direct-volume-rendering.md)
Uses topological segmentation to guide rendering attributes, distinguishing connected structures.

### [WebGL Path Tracing Volume Rendering](8592-webgl-path-tracing-volume-rendering.md)
Combines direct ray casting with path tracing for lighting/shadows in web browsers.

### [Temporal and Procedural Techniques](9442-temporal-and-procedural-techniques.md)
Generates data or manages performance dynamically, using procedural evaluation instead of fixed textures.

### [Specific Applications and Integrators](9443-specific-applications-and-integrators.md)
Tailored techniques for specific physical processes or medical imaging requirements.

### [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md)
Offloads sampling and compositing to GPUs using shaders and texture hardware.

### [NeRV Rendering](10264-nerv-rendering.md)
Computes pixel color by integrating reflected radiance from a physically-based shading model, differing from standard NeRF.

### [Distributed AMR Volume Rendering with Cost-Range Decomposition](10508-distributed-amr-volume-rendering-with-cost-range-decompositi.md)
Decomposes AMR data for parallel rendering based on estimated cost to balance load.

### [Volume Rendering in 3D Slicer](10540-volume-rendering-in-3d-slicer.md)
Implements DVR in the Slicer framework using VTK backends (CPU/GPU Ray Cast, Volume Rasterization).

### [Raymarching with 3D Volume Texture](10839-raymarching-with-3d-volume-texture.md)
Uses a single 3D volume texture for raymarching, simplifying edge handling compared to 2D atlases.

### [AMR and Hierarchical Data Rendering](11062-amr-and-hierarchical-data-rendering.md)
Visualizes non-uniform resolution data (AMR) using hierarchical structures to reduce memory bandwidth.

### [Spherical Harmonic Volume Ray Casting](11248-spherical-harmonic-volume-ray-casting.md)
Pre-computes SH coefficients on CPU for GPU ray marching to approximate complex illumination.

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
Integrates artistic style transfer and contour lines for non-photorealistic visualization.

### [Illustrative Volume Rendering with Style Transfer](11445-illustrative-volume-rendering-with-style-transfer.md)
Applies artistic styles via a Style Transfer Function (STF) to volumetric data.

### [Sliced Ray Casting with On-Demand Volume Reconstruction](11645-sliced-ray-casting-with-on-demand-volume-reconstruction.md)
Reconstructs particle density on-demand in screen space to avoid aliasing and memory issues.

### Interactive Directional Ambient Occlusion and Shadow for Volume Ray Casting
A strategy to compute directional ambient occlusion and shadows for structured datasets by evaluating Gaussian integrals at discrete samples along the cone axis [S81]. It creates an extra volume for Gaussian amplitudes and uses mipmapping with circle packing to evaluate integrals, delivering a better quality-performance balance than previous specialized procedures [S81].

### Voronoi Painter's Algorithm Compositing
A compositing strategy where the scene is rendered part-by-part (one Voronoi cell at a time) and composited back-to-front [S82]. It relies on the partial ordering of Voronoi cells based on distance from the camera ($V_{P'} <_Q V_P$ iff $d(P', Q) < d(P, Q)$), ensuring closer objects occlude further ones, unlike the standard Painter's Algorithm which fails on cyclic occlusion [S82].

### Quadratic Approximation for Exponential Blending
A technique to approximate the exponential light attenuation function $f(z) = 1 - e^{-\tau z}$ with a quadratic function $g(z) = (c_0 + c_1 z)(a_0 + a_1 z)$ for efficient hardware implementation using Gouraud shading, maintaining accuracy for small cell depths [S83].

### Isosurface-Accelerated Ray Casting with Weighted Last Sample
An optimization using extracted isosurfaces as opaque barriers to terminate rays early [S84]. To counteract aliasing from the last incorrect sample (inside the surface), it weights the sample by the fraction of the step outside the surface [S84].

### Direct Volume Rendering from Mixture Graph
Rendering directly from binary Mixture Graph representations at interactive rates on the GPU, supporting real-time transfer function editing [S85]. It uses a raymarcher sampling at 0.5 voxels, though pre-computed normals can cause artifacts around medial axes [S85].

### Fixed-Point Volume Ray Casting
A VTK implementation using fixed-point arithmetic for ray casting, with specific shading coefficients (Ambient: 0.2, Diffuse: 0.9, Specular: 0.2, Power: 10) and piecewise transfer functions for opacity, color, and gradient [S86, unofficial].

### Pre-integration of Transfer Function Contribution
A technique by Engel et al. to smooth image imperfections from low sampling frequencies by pre-integrating transfer function contributions along ray segments, addressing the dichotomy between quality and performance [S87, unofficial].

## Background
Direct volume ray casting was introduced by Marc Levoy in his seminal 1988 paper "Display of Surfaces from Volume Data" [S12, unofficial][S59, book]. Levoy coined the term "direct volume rendering" and defined the pipeline of classification, shading, and compositing, emphasizing the visualization of weak or fuzzy surfaces with smooth silhouettes [S12, unofficial]. The method is based on the volume rendering equation, which integrates radiative energy along rays of sight [S13, unofficial]. Early implementations were CPU-based, but the advent of programmable graphics hardware (e.g., DirectX 9, OpenGL shaders) enabled GPU-accelerated ray casting, significantly improving performance [S8, S7443]. The technique is widely used in medical imaging (e.g., CT, MRI), scientific visualization, and computer graphics for visualizing semi-transparent materials [S4, S12, unofficial][S70, book].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] The Ray-Marching Algorithm, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/ray-marching-algorithm.html, fetched 2026-10-25 (unofficial)
[S3] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S4] Volume Ray Casting Techniques, https://www.scribd.com/document/102700078/03-raycasting, fetched 2026-10-25 (unofficial)
[S5] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S6] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S7] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S8] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S9] Shear-warp factorization volume rendering visualization system, https://archive.org/details/0162018299360, fetched 2026-10-26 (unofficial)
[S10] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S11] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S12] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S13] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S14] palasjir/ct-volumetric-rendering, https://github.com/palasjir/ct-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S15] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
[S16] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S17] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26 (unofficial)
[S18] khizarsiddiqui/Volume-Rendering, https://github.com/khizarsiddiqui/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S19] RuolinZheng08/volume-rendering, https://github.com/RuolinZheng08/volume-rendering, fetched 2026-10-26 (unofficial)
[S20] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S21] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S22] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S23] Visualization Lecture 09: VolVis Part 3-Direct Volume Rendering, Ray and Transfer Functions, https://www.youtube.com/watch?v=FYAJQgYdeqQ, fetched 2026-10-27 (unofficial)
[S24] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S25] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S26] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S27] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S28] MPanknin/volume-occlusion-editor, https://github.com/MPanknin/volume-occlusion-editor, fetched 2026-10-26 (unofficial)
[S29] gillesferrand/Unity-RayTracing, https://github.com/gillesferrand/Unity-RayTracing, fetched 2026-10-26 (unofficial)
[S30] Dy111111/volumeRenderingGPU, https://github.com/Dy111111/volumeRenderingGPU, fetched 2026-10-26 (unofficial)
[S31] vulekhanh/VolumeRendering, https://github.com/vulekhanh/VolumeRendering, fetched 2026-10-26 (unofficial)
[S32] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S33] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S34] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S35] hands133/opengl-directVolumeRenderer, https://github.com/hands133/opengl-directVolumeRenderer, fetched 2026-10-26 (unofficial)
[S36] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S37] thePZH/VolumeRendering-OpenGL, https://github.com/thePZH/VolumeRendering-OpenGL, fetched 2026-10-26 (unofficial)
[S38] manishaa1704/Interactive-DVR-System, https://github.com/manishaa1704/Interactive-DVR-System, fetched 2026-10-26 (unofficial)
[S39] asheibanifard/Direct_Volume_Rendering-DVR-, https://github.com/asheibanifard/Direct_Volume_Rendering-DVR-, fetched 2026-10-26 (unofficial)
[S40] GitHub - LDeakin/VkVolume: Vulkan-based volume renderer with efficient ..., https://github.com/LDeakin/VkVolume, fetched 2026-10-26 (unofficial)
[S41] Volume Rendering With Ray Casting - WPI, https://web.cs.wpi.edu/~matt/courses/cs563/talks/powwie/p1/ray-cast.htm, fetched 2026-10-26 (unofficial)
[S42] PDF VOLUME RENDERING BY RAY CASTING - Warse, https://www.warse.org/IJATCSE/static/pdf/file/icetetssp47.pdf, fetched 2026-10-30 (unofficial)
[S43] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S44] 20 volume rendering (using glsl) with ray casting algorithm, https://stackoverflow.com/questions/9482572/volume-rendering-using-glsl-with-ray-casting-algorithm?rq=4, fetched 2026-10-26 (unofficial)
[S45] Volume ray casting doesn't work fine (Webgl + GLSL + Three.js), https://stackoverflow.com/questions/16274334/volume-ray-casting-doesnt-work-fine-webgl-glsl-three-js?noredirect=1&lq=1, fetched 2026-10-27 (unofficial)
[S46] Guide for beginners: What is Volume Ray Casting? - VisCircle, https://viscircle.com/guide-for-beginners-what-is-volume-ray-casting/, fetched 2026-10-27 (unofficial)
[S47] Volume Ray Casting Techniques and Applications using ..., https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S48] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S49] 3D: ray-cast volume rendering instead of a stack of slices, https://github.com/jamescarruthers/quick_dicom/pull/4, fetched 2026-10-27 (unofficial)
[S50] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
[S51] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
[S52] hwenchi/visible-human-volume, https://github.com/hwenchi/visible-human-volume, fetched 2026-10-27 (unofficial)
[S53] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S54] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
[S55] vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, https://github.com/vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, fetched 2026-10-27 (unofficial)
[S56] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S57] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
[S58] overhaul88/Parallel_Volume_Rendering, https://github.com/overhaul88/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S59] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S60] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S61] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
[S62] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
[S63] WebGL Volume Rendering made simple - Lebarba, https://www.barbaleon.co.uk/webgl/index.html, fetched 2026-10-28 (unofficial)
[S64] (PDF) GPU-Based Volume Rendering for Medical Imagery - ResearchGate, https://www.researchgate.net/publication/228912978_GPU-Based_Volume_Rendering_for_Medical_Imagery, fetched 2026-10-28 (unofficial)
[S65] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S66] Home, https://graphics.cg.uni-saarland.de/competition/cg1-2018/original/groupR_web/, fetched 2026-10-28 (unofficial)
[S67] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S69] (PDF) Differentiable Direct Volume Rendering - ResearchGate, https://www.researchgate.net/publication/353510271_Differentiable_Direct_Volume_Rendering, fetched 2026-10-30 (unofficial)
[S70] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S71] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
[S72] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S73] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S74] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
[S75] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
[S76] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
[S77] GPU-Enabled Volume Renderer for Use with MATLAB, https://doi.org/10.3390/digital4040049, fetched 2026-10-29
[S78] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30 (unofficial)
[S79] Volume Rendering in Unity3D - Speaker Deck, https://speakerdeck.com/mlavik1/volume-rendering-in-unity3d, fetched 2026-10-30 (unofficial)
[S80] Volumetric Rendering - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/volumetric-rendering/, fetched 2026-10-30 (unofficial)
[S81] (PDF) A Comparison between GPU-based Volume Ray Casting ..., https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S82] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
[S83] Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering, https://ntrs.nasa.gov/api/citations/19930015755/downloads/19930015755.pdf, fetched 2026-10-24
[S84] Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline, https://doi.org/10.1145/1281500.1361219, fetched 2026-11-03
[S85] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S86] comedate/VolumeRendering, https://github.com/comedate/VolumeRendering, fetched 2026-10-27 (unofficial)
[S87] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
