# Direct volume ray casting

Also called: DVR, ray marching, front-to-back compositing, image-order volume rendering

## Overview
Direct volume ray casting (DVR) is an image-order rendering technique that synthesizes a 2D image from a 3D scalar volume by casting a ray through each pixel, sampling the volume along the ray, and integrating the optical properties (color and opacity) using the volume rendering integral [S1][S4, unofficial][S16, unofficial]. It solves the problem of visualizing the internal structure of semi-transparent volumetric data without extracting explicit geometry, allowing all voxels to potentially contribute to the final image [S5, unofficial][S12, unofficial]. Unlike surface-based methods (e.g., [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md)) that stop at the first opaque boundary, DVR accumulates contributions from the entire ray segment within the volume, preserving depth cues and transparency [S12, unofficial][S23, unofficial].

## Shared principles
All direct volume ray casting methods share the following core mechanism:

1.  **Ray Definition**: For each pixel $(x, y)$ in the image plane, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the viewing direction [G]. The ray is intersected with the volume bounding box to determine the entry parameter $t_{near}$ and exit parameter $t_{far}$ [S2, unofficial][S10, unofficial][S20, unofficial].
2.  **Sampling**: The ray segment $[t_{near}, t_{far}]$ is discretized into $N$ samples at positions $t_i = t_{near} + i \cdot \Delta t$, where $\Delta t = (t_{far} - t_{near}) / N$ [S2, unofficial][S77, unofficial]. At each sample position $\mathbf{p}_i = r(t_i)$, the scalar volume value $V(\mathbf{p}_i)$ is retrieved, typically via trilinear interpolation of the surrounding voxels [S9, unofficial][S12, unofficial][S53, unofficial].
3.  **Classification (Transfer Function)**: The scalar value $V(\mathbf{p}_i)$ is mapped to optical properties: a color vector $\mathbf{c}_i$ (RGB) and an opacity scalar $\alpha_i$ (A) [S25, unofficial][S41, unofficial]. This mapping is defined by a transfer function $T(V) \rightarrow (\mathbf{c}, \alpha)$ [S1][S4, unofficial]. The opacity is often corrected for the step size $\Delta t$ to approximate continuous attenuation, using models such as Beer-Lambert: $\alpha_i = 1 - e^{-\sigma_i \Delta t}$, where $\sigma_i$ is the extinction coefficient derived from $V(\mathbf{p}_i)$ [S26, unofficial][S33, unofficial][S39, unofficial].
4.  **Compositing (Integration)**: The samples are composited into the final pixel color $\mathbf{C}_{out}$ and accumulated opacity $\alpha_{out}$ using the front-to-back volume rendering equation [S17, unofficial][S43, unofficial][S68, abstract only, unofficial]:
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
    where $\mathbf{C}_{acc}$ and $\alpha_{acc}$ are initialized to $(0, 0)$ [S17, unofficial][S58, unofficial][S67, unofficial].
5.  **Shading**: At each sample, a shading model (e.g., Phong, Blinn-Phong) may be applied to compute the base color $\mathbf{c}_i$ based on the local gradient $\nabla V(\mathbf{p}_i)$, which serves as the surface normal [S12, unofficial][S19, unofficial][S41, unofficial].

## Subtopics

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
Generates a 2D projection by processing 3D data as 2D slices, applying transfer functions, and compositing. Slower than direct ray casting due to slice-based overhead.

### [Anti-Aliasing and Filtering](1574-anti-aliasing-and-filtering.md)
Addresses visual artifacts from undersampling by smoothing transitions between discrete samples and continuous geometry across different scales.

### [Neural volume rendering](1576-neural-volume-rendering.md)
Synthesizes novel views by querying a neural network along camera rays and integrating outputs using classical volume rendering techniques.

### [Temporal Accumulation (Closest Hit)](2080-temporal-accumulation-closest-hit.md)
Stabilizes stochastic or noisy results by blending the current frame with a history buffer, specifically selecting the closest hit to preserve depth boundaries.

### [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md)
Renders multiple volume datasets simultaneously by extending single-volume mappers to accept multiple datasets and properties, using GPU shaders for traversal.

### [Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md)
Defines how continuous scalar fields are approximated from discrete voxel data, trading computational cost, visual smoothness, and aliasing artifacts.

### [Forward Mapping](2913-forward-mapping.md)
Projects each voxel directly onto the image plane, relying on data traversal order for hidden surface removal, contrasting with image-order ray casting.

### [Layered Volume Rendering with Light Propagation](4083-layered-volume-rendering-with-light-propagation.md)
Processes a stack of parallel planes instead of individual pixels, propagating light and view rays across layers to accumulate color and opacity.

### [Volumetric Rendering with Spherical Harmonics](4962-volumetric-rendering-with-spherical-harmonics.md)
Represents view-dependent color and opacity data using SH coefficients stored in a voxel grid to solve expensive per-sample shading.

### [Customization Volume Integration](5291-customization-volume-integration.md)
Integrates a pre-computed volume to apply position-dependent modifications to visualization parameters, such as gamma correction.

### [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md)
Converts 3D volumetric data into 2D polygonal slices rasterized and composited by standard graphics hardware for efficient large-volume rendering.

### [Volume Ray-Casting on Graphics Hardware](7443-volume-ray-casting-on-graphics-hardware.md)
Implements direct volume ray casting on programmable graphics hardware using a multi-pass stream model to solve CPU performance bottlenecks.

### [Cutting Plane Integration](8414-cutting-plane-integration.md)
Dynamically slices a 3D dataset using a semi-transparent geometric plane to reveal internal structures occluded by opaque outer layers.

### [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md)
Treats volume datasets and geometric objects as Volume-of-Interests (VOIs) in a unified rendering framework with priority-based compositing.

### [Temporal and Procedural Techniques](9442-temporal-and-procedural-techniques.md)
Generates volumetric data or manages rendering performance dynamically using procedural evaluation instead of static pre-computed textures.

### [Specific Applications and Integrators](9443-specific-applications-and-integrators.md)
Deviates from standard front-to-back compositing to address specific requirements like projective diagnostic imaging, medical photo-realism, and neural scene composition.

### [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md)
Offloads computationally intensive sampling and compositing stages to the GPU, utilizing programmable shaders and texture hardware for interactive frame rates.

### [Splatch](9863-splatch.md)
A lightweight, CPU-based ray-tracing tool for fast visualization of point-like data from cosmological simulations, solving large-scale particle dataset rendering.

### [Raymarching with 3D Volume Texture](10839-raymarching-with-3d-volume-texture.md)
Replaces 2D texture atlases with a single 3D volume texture for raymarching, allowing larger volumes and simplifying edge handling.

### [AMR and Hierarchical Data Rendering](11062-amr-and-hierarchical-data-rendering.md)
Visualizes datasets with non-uniform spatial resolution by leveraging hierarchical structures to reduce computational and memory overhead.

### [Spherical Harmonic Volume Ray Casting](11248-spherical-harmonic-volume-ray-casting.md)
Approximates complex light-material interactions by pre-computing SH coefficients for material and light functions on the CPU for GPU ray marching.

### [Sliced Ray Casting with On-Demand Volume Reconstruction](11645-sliced-ray-casting-with-on-demand-volume-reconstruction.md)
Reconstructs particle density on-demand in screen space at viewport resolution, combining splatting, texture slicing, and ray casting to avoid aliasing.

### [Fixed-Point Volume Ray Casting](12056-fixed-point-volume-ray-casting.md)
Utilizes fixed-point arithmetic for ray traversal and compositing in VTK, particularly suited for rendering medical images.

### [Standard Ray Casting](12643-standard-ray-casting.md)
The foundational technique generating images by casting rays from the camera, sampling a 3D scalar field at regular intervals, and accumulating via front-to-back compositing.

### [Block-based Volume Rendering](13116-block-based-volume-rendering.md)
Divides space into independent blocks, rendering each separately and blending results to solve resource isolation in distributed environments.

### [Cel Shading for Volumes](13210-cel-shading-for-volumes.md)
Applies stylized, non-photorealistic shading effects including tone detail and specular highlights using discrete tone mapping.

### [Partial Segment Handling](13455-partial-segment-handling.md)
Addresses stair-stepping artifacts at volume boundaries by introducing a smaller final segment to complete the ray march.

### [Plane-Based Sampling](13531-plane-based-sampling.md)
Replaces per-voxel trilinear interpolation with a two-stage process of calculating ray-plane intersections and interpolating between them.

### [Improved Ray Casting Volume Rendering](13625-improved-ray-casting-volume-rendering.md)
Optimized for large-scale regular 3D spatial data fields, prioritizing rendering speed and high-frequency component preservation over physical lighting accuracy.

### [Multi-Pass and Compositing Techniques](14748-multi-pass-and-compositing-techniques.md)
Decomposes the workload into multiple passes to decouple computational stages like light attenuation and color compositing.

### [Multi-Mode Volume Ray Casting](14762-multi-mode-volume-ray-casting.md)
Extends standard compositing to support Full, Maximum Intensity Projection (MIP), and XRay rendering modes.

### [Transfer Function and Shading Techniques](15496-transfer-function-and-shading-techniques.md)
Defines how scalar data is mapped to color and opacity, enabling visualization of internal structures by assigning transparency.

#### Commutative Blend Direct Volume Rendering
A rendering technique for unsorted cell projection using optical models that lead to a commutative blend function, allowing compositing in arbitrary order [S93]. Models allowing arbitrary order compositing include maximum intensity projection and restrictions of the full volume density model to either the source term or the absorption term [S93].

#### Svox1 Rendering Formula (Nearest-Neighbor)
Implements the Svox1 rendering formula using nearest-neighbor interpolation with infinitely many steps, including forward, backward, and fused variants [S94, unofficial].

#### Isosurface-Accelerated Ray Casting with Weighted Last Sample
An optimization for ray casting that uses the extracted isosurface as an opaque barrier to terminate rays early, rendering the isosurface's 3D texture coordinates into the exit point texture [S95]. To counteract the last incorrect sample, it weights it by the fraction of the step that is outside of the surface [S95]. Straightforward integration causes aliasing/banding artifacts because the isosurface coordinates may not lie exactly on the sample point of a marching ray [S95].

#### Direct Volume Rendering from Mixture Graph
Rendering directly from the binary Mixture Graph representation at interactive rates on the GPU, supporting real-time transfer function editing across all scales of the mipmap [S96]. The rendering uses a direct volume raymarcher sampling at 0.5 voxels [S96]. Pre-computed normals derived from the method cause artifacts in the form of caps around the medial axis where segments leave the domain [S96]. Lighting contributions are computed without knowledge of position, restricting the method to light and camera at infinity [S96]. Used for real-time visual analysis applications in neuroscience and material science [S96].

#### RGBA Volume Rendering
A technique to render multiple volumes by taking the value of the texture itself as the color, bypassing the colormap lookup, using a 4D array (z, y, x, 4) passed directly to the texture [S97, unofficial]. Requires colormaps that can work in Python on Numpy arrays [S97, unofficial]. Implemented in the vispy-multivol repository as the RGBAVolumeVisual class [S97, unofficial].

#### Variable Sample Distance / Random Offset
A strategy to break the stripe pattern of onion ring artifacts by varying sample locations of rays in adjacent pixels, either by using variable sample distances or a random offset at the beginning of each ray [S98, unofficial]. Result is a more 'dithered' image where errors are randomly distributed over pixels [S98, unofficial].

#### CPU Volume Transport (Multi-Ray)
Solves transport for multiple rays in parallel on CPU by iterating over rays and calling the single-ray solver for each [S99, unofficial]. Input arrays sigma, source, initial_intensity are row-major [ray][sample] [S99, unofficial].

#### TSDF Volume Rendering
Computes unique frustum block coordinates for the camera viewpoint to determine which volume blocks need to be rendered [S100, unofficial]. Performs ray casting on the volume blocks using intrinsic, extrinsic, width, height, and render attributes [S100, unofficial]. Extracts voxel indices and interpolation ratios from the ray cast results to reconstruct color via weighted interpolation of volume attributes [S100, unofficial]. Reconstructs color by multiplying the volume color attribute at the hit indices by the interpolation ratios, summing over the 8 corner voxels of the hit cell, and normalizing by 255 [S100, unofficial].

## Background
Direct volume ray casting was introduced by Marc Levoy in his seminal 1988 paper "Display of Surfaces from Volume Data" [S12, unofficial][S59, unofficial]. Levoy coined the term "direct volume rendering" and defined the pipeline of classification, shading, and compositing, emphasizing the visualization of weak or fuzzy surfaces with smooth silhouettes [S12, unofficial]. The method is based on the volume rendering equation, which integrates radiative energy along rays of sight [S13, unofficial]. Early implementations were software-based, but the advent of programmable graphics hardware enabled GPU-accelerated ray casting, significantly improving performance [S8, unofficial][S74, unofficial]. The technique is used in scientific visualization and computer graphics for visualizing semi-transparent materials [S4, unofficial][S12, unofficial][S70].

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
[S51] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27
[S52] hwenchi/visible-human-volume, https://github.com/hwenchi/visible-human-volume, fetched 2026-10-27 (unofficial)
[S53] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S54] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
[S55] vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, https://github.com/vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, fetched 2026-10-27 (unofficial)
[S56] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S57] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
[S58] overhaul88/Parallel_Volume_Rendering, https://github.com/overhaul88/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S59] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (unofficial)
[S60] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S61] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (unofficial)
[S62] JCLemme/voxnine, https://github.com/JCLemme/voxnine, fetched 2026-10-28 (unofficial)
[S63] WebGL Volume Rendering made simple - Lebarba, https://www.barbaleon.co.uk/webgl/index.html, fetched 2026-10-28 (unofficial)
[S64] (PDF) GPU-Based Volume Rendering for Medical Imagery - ResearchGate, https://www.researchgate.net/publication/228912978_GPU-Based_Volume_Rendering_for_Medical_Imagery, fetched 2026-10-28 (unofficial)
[S65] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (book)
[S66] Home, https://graphics.cg.uni-saarland.de/competition/cg1-2018/original/groupR_web/, fetched 2026-10-28
[S67] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S69] (PDF) Differentiable Direct Volume Rendering - ResearchGate, https://www.researchgate.net/publication/353510271_Differentiable_Direct_Volume_Rendering, fetched 2026-10-30 (unofficial)
[S70] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29
[S71] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
[S72] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S73] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S74] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29 (unofficial)
[S75] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29 (unofficial)
[S76] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
[S77] GPU-Enabled Volume Renderer for Use with MATLAB, https://doi.org/10.3390/digital4040049, fetched 2026-10-29 (unofficial)
[S78] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30
[S79] Volume Rendering in Unity3D - Speaker Deck, https://speakerdeck.com/mlavik1/volume-rendering-in-unity3d, fetched 2026-10-30 (unofficial)
[S80] Volumetric Rendering - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/volumetric-rendering/, fetched 2026-10-30 (unofficial)
[S81] (PDF) A Comparison between GPU-based Volume Ray Casting ..., https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26 (book)
[S82] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26 (unofficial)
[S83] Rapid Exploration of Curvilinear Grids Using Direct Volume Rendering, https://ntrs.nasa.gov/api/citations/19930015755/downloads/19930015755.pdf, fetched 2026-10-24 (unofficial)
[S84] Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline, https://doi.org/10.1145/1281500.1361219, fetched 2026-11-03 (unofficial)
[S85] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S86] comedate/VolumeRendering, https://github.com/comedate/VolumeRendering, fetched 2026-10-27
[S87] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
[S93] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S94] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S95] Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline, https://doi.org/10.1145/1281500.1361219, fetched 2026-11-03
[S96] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S97] Showing multiple volumes in a single volume rendering, https://github.com/vispy/vispy/issues/1156, fetched 2026-10-26 (unofficial)
[S98] Common Artifacts in Volume Rendering, https://doi.org/10.48550/arxiv.2109.13704, fetched 2026-10-26 (unofficial)
[S99] wyh1309/hpc_foundational_demos, https://github.com/wyh1309/hpc_foundational_demos, fetched 2026-10-26 (unofficial)
[S100] ToniRV/NeRF-SLAM, https://github.com/ToniRV/NeRF-SLAM, fetched 2026-10-26 (unofficial)
