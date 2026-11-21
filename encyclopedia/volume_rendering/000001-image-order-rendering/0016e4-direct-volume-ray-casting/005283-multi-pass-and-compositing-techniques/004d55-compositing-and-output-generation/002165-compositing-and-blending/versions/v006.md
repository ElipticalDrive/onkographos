# Compositing and Blending

Also called: Accumulate, Alternative Compositing Schemes, Blending Modes, First Hit, Front-to-back blending, Maximum Intensity blending, Minimum Intensity blending

## Overview
Compositing and Blending defines the mathematical and algorithmic rules for combining volumetric samples along a ray into a final pixel color and opacity. It solves the problem of optical integration, determining how light emitted or scattered by volume elements is attenuated by absorption and combined with background light. It sets itself apart from [Projection and MIP Techniques](6305-projection-and-mip-techniques.md) by explicitly tracking opacity and transmittance for alpha blending, and from [Volume Raycasting Output Generation](4277-volume-raycasting-output-generation.md) by focusing on the accumulation logic itself rather than the broader output pipeline.

## Shared principles
All compositing methods in this family operate on a sequence of samples $(c_i, \alpha_i)$ encountered along a ray, where $c_i$ is the sample color (RGB) and $\alpha_i$ is the sample opacity (A) [S1, unofficial][S2, unofficial]. The core mechanism involves an accumulator state $(C_{acc}, \alpha_{acc})$ initialized to $(0, 0)$ (or background color/opacity) and updated iteratively for each sample [S1, unofficial][S2, unofficial].

The standard optical model assumes that light is emitted or scattered by the volume and attenuated by absorption [S1, unofficial]. The standard front-to-back compositing equation updates these accumulators for each sample $i$ from front to back:
$$
C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i C_i
$$
$$
\alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i
$$
where $C_{acc}^{(0)} = 0$ and $\alpha_{acc}^{(0)} = 0$ [S12, S43, S50, unofficial]. The term $(1 - \alpha_{acc}^{(i-1)})$ represents the transmittance, i.e., the fraction of light reaching sample $i$ from the camera side [S1, S51, unofficial].

In iterative form using transmittance $T = 1 - \alpha_{acc}$:
$$
\begin{aligned}
C_{final} &\leftarrow C_{final} + c_i \cdot \alpha_i \cdot T \\
T &\leftarrow T \cdot (1 - \alpha_i)
\end{aligned}
$$
Where $T$ is initialized to $1.0$ and represents the accumulated transparency $\prod (1-\alpha_j)$ [G].

The final pixel color is often composited over a background color $C_{bg}$ using $C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}$ [S1, S14, unofficial].

Early ray termination is a common optimization where the loop breaks if $\alpha_{acc}$ exceeds a threshold (e.g., 1.0), as further samples contribute negligibly to the color [S10, S12, unofficial].

## Subtopics

### [Pre-Integration Lookup](2854-pre-integration-lookup.md)
[Pre-Integration Lookup](2854-pre-integration-lookup.md) is a direct volume ray casting optimization that replaces the iterative accumulation of color and opacity along a ray with a single texture fetch from a pre-computed table. It solves the performance bottleneck of evaluating the optical integration equation for every sample by assuming linear data evolution within a slab, allowing the final color and opacity contribution of a slab to be determined solely by its entry density, exit density, and thickness. It sets itself apart from iterative compositing by collapsing multiple samples into a single lookup, and from simplified pre-integration by accounting for color attenuation within the slab rather than neglecting it.

### [Binary Swap Compositing](3264-binary-swap-compositing.md)
[Binary Swap Compositing](3264-binary-swap-compositing.md) is a parallel image merging algorithm used to combine independently rendered sub-images from multiple processors into a final volume-rendered image. It solves the bottleneck problem of naive binary compositing, where processors become idle as the compositing tree narrows, by ensuring every processor participates in every stage of the merge. It sets itself apart from sibling methods like [Compositing and Blending](2515-compositing-and-blending.md) by explicitly managing distributed memory communication via a binary-swap tree structure, and from [Pre-Integration Lookup](2854-pre-integration-lookup.md) by operating on already-composited sub-images rather than raw samples.

### [Screen-space Post-processing](3621-screen-space-post-processing.md)
[Screen-space Post-processing](3621-screen-space-post-processing.md) is a full-screen rendering pass that applies aspect-ratio preserving UV covering, luminance-based bright passes, color grading, and spatial effects (such as bloom, blur, and vignetting) to a scene texture to produce a display-ready image. It differs from ray-casting compositing methods by operating entirely in screen space on the final 2D image rather than during volumetric ray traversal, allowing for complex aesthetic effects like underwater refraction and tone mapping without modifying the volume rendering pipeline.

### [Layered SDF Compositing](3641-layered-sdf-compositing.md)
[Layered SDF Compositing](3641-layered-sdf-compositing.md) is a method for rendering Signed Distance Field (SDF) glyphs by compositing multiple layers (typically a face and one or more outline layers) using alpha blending and pre-multiplied alpha handling. It differs from standard volume ray casting by operating on discrete, pre-sampled layers of a 2D texture atlas rather than continuous 3D volume data, allowing for efficient, sharp text rendering with variable-width outlines.

### [Reconstruction and Compositing Techniques](4646-reconstruction-and-compositing-techniques.md)
[Reconstruction and Compositing Techniques](4646-reconstruction-and-compositing-techniques.md) encompass the mathematical and algorithmic methods used to transform discrete volumetric data into continuous signals and combine those signals into final pixel colors. This family includes methods for evaluating polynomial fields along rays, resampling discrete data via convolution, and integrating optical properties using various compositing models. It is distinguished from sibling methods by its focus on the underlying signal processing (reconstruction) and the specific mathematical formulation of the accumulation (compositing), ranging from standard alpha blending to differentiable training passes and importance-driven modulation.

### [Lighting-Specific Compositing](5940-lighting-specific-compositing.md)
[Lighting-Specific Compositing](5940-lighting-specific-compositing.md) refers to a family of volume rendering techniques that modify the standard front-to-back compositing equation to incorporate complex lighting effects, such as in-scattering, multi-channel illumination, geometric boundary enhancement, and Phong shading, directly into the accumulation step. Unlike standard compositing which treats samples as simple opaque/translucent entities, these methods compute lighting contributions ($\Delta C_i, \Delta \alpha_i$) per sample or per channel before blending them into the accumulator. This family is distinct from [Pre-Integration Lookup](2854-pre-integration-lookup.md) (which approximates integration via texture lookups) and [Binary Swap Compositing](3264-binary-swap-compositing.md) (which focuses on parallel reduction structures), as it specifically addresses the *illumination model* within the compositing loop.

### [Layer-Centric Volume Rendering Framework](6994-layer-centric-volume-rendering-framework.md)
[Layer-Centric Volume Rendering Framework](6994-layer-centric-volume-rendering-framework.md) is a system architecture that decomposes volume visualization into two distinct phases: rendering small regions of interest (ROIs) as separate layers, and compositing these layers using a rich set of 2D operators. Unlike traditional single-pass ray casting, this method treats the final image as a composition of independent renderings, allowing for intuitive 2D-based user interactions and complex material manipulations. It builds on [Compositing and Blending](2515-compositing-and-blending.md) by extending standard Porter-Duff operators to general functional compositing functions applied to layer textures.

### [MAB Compositing with Gaze-Contingent Blending](8211-mab-compositing-with-gaze-contingent-blending.md)
[MAB Compositing with Gaze-Contingent Blending](8211-mab-compositing-with-gaze-contingent-blending.md) is a gaze-contingent rendering technique that maps framebuffer fragments to a Multi-Anchor Buffer (MAB) to retrieve and blend color values from different resolution levels based on the distance to the user's gaze. It solves the problem of maintaining high frame rates in immersive VR environments by dynamically adjusting the rendering resolution distribution in real-time, thereby preventing performance fluctuations that cause cybersickness. Unlike standard fixed-resolution compositing, it blends multiple resolution levels simultaneously using smooth-step transitions to ensure unnoticeable visual artifacts between detail levels.

### [Multi-view image blending](9831-multi-view-image-blending.md)
[Multi-view image blending](9831-multi-view-image-blending.md) reduces view-dependent artifacts, such as ghosting and transparency variation, by combining multiple pre-rendered images captured from different camera angles. Unlike single-view texture mapping, which suffers from fading and duplication when the viewpoint changes, this method selects and blends at least three textures to approximate the volumetric appearance. It is distinguished from single-view techniques by its reliance on spatial redundancy across views, at the cost of increased memory bandwidth and selection complexity.

### [Dynamic Job-Scheduling for Grid Volume Rendering](10585-dynamic-job-scheduling-for-grid-volume-rendering.md)
[Dynamic Job-Scheduling for Grid Volume Rendering](10585-dynamic-job-scheduling-for-grid-volume-rendering.md) is a distributed rendering strategy for grid-based volume rendering that dynamically determines the processing order of sub-volumes based on their visibility, using obstacle-flags to avoid the bottlenecks of static sequential scheduling. Unlike sequential methods that force agents to wait for prior results to be combined before sending the next job, this approach continuously monitors obscuring volumes and dispatches jobs as soon as they become combinable (obscure count 0), even if prior jobs are still being processed by other agents. It solves the inefficiency caused by heterogeneous agent speeds and varying resource availability in grid computing environments.

### [HardSoftRas Blending](10747-hardsoftras-blending.md)
[HardSoftRas Blending](10747-hardsoftras-blending.md) is a pixel-wise compositing strategy for differentiable rendering that distinguishes between edge pixels (silhouettes and vertical surfaces) and non-edge pixels to balance anti-aliasing with texture sharpness. Unlike SoftRas, which applies weighted blending across all faces including those inside silhouettes, HardSoftRas preserves the most frontal color for non-edge regions while using probabilistic blending for edges, thereby maintaining sharp internal textures while generating far-range gradients for optimization.

### Commutative Blend Direct Volume Rendering
Rendering technique for unsorted cell projection using optical models that lead to a commutative blend function, allowing compositing in arbitrary order [S3]. Models allowing arbitrary order compositing include maximum intensity projection and restrictions of the full volume density model to either the source term or the absorption term [S3]. It applies arbitrary transfer functions via texture lookup, which only affect the generation of the texture map [S3].

### Frame Buffer Copy / Blit
A pass that simply copies the current frame to the output, acting as a simple integration or blit step for the LAO renderer [S4, unofficial]. It samples the frame buffer using `oColor = texture(uFrame, vPosition)` [S4, unofficial].

### Virtual Dimension Expansion for Compositing
A post-processing step where partial images rendered from blocks of different resolutions are expanded to a common 'virtual dimension' (the highest resolution dimension of the block) before compositing [S5, unofficial]. This allows standard compositing algorithms to work despite varying input resolutions. The virtual dimension can be set to the dimension of the largest subimage in the scene to minimize padding, or fixed to the global high-resolution block size [S5, unofficial].

### Voronoi Painter's Algorithm Compositing
A rendering compositing strategy where the scene is rendered part-by-part (one Voronoi cell at a time) and the resulting cell images are composited back-to-front to generate the final image [S6]. It relies on the proof that Voronoi cells admit a partial ordering compatible with occlusion. The ordering is defined such that $V_{P'} <_Q V_P$ if and only if $d(P', Q) < d(P, Q)$ [S6]. Standard Painter's Algorithm fails on cyclic occlusion graphs, but Voronoi Painter's is always compatible because the distance-to-camera ordering is a strict partial order [S6].

### Hybrid Compositing (Radix-k)
Hybrid methods like Radix-k combine direct send and binary swap approaches depending on the problem setting, adaptively running one or the other based on criteria [S7, unofficial].

### Fabric Compositing
A compositing pass that creates an initial fabric-like texture by combining advected noise fields, silhouette data, and lighting [S8, book]. Pixels not part of the geometry are discarded using the depth buffer. The color channels after $k$ iterations are defined as:
$$
R(x,y) = \frac{r \cdot f_{\lambda_2}(x,y)}{8 \cdot f_{\lambda_1}^2(x,y)} + e_{x,y} + \text{light}(\mathcal{L}_{x,y})
$$
$$
G(x,y) = \frac{(1-r) \cdot f_{\lambda_1}(x,y)}{8 \cdot f_{\lambda_2}^2(x,y)} + e_{x,y} + \text{light}(\mathcal{L}_{x,y})
$$
$$
B(x,y) = e_{x,y} + \text{light}(\mathcal{L}_{x,y})
$$
Here $f_{\lambda_1}$ and $f_{\lambda_2}$ are the advected fields $p_{\lambda_1}^k$ and $p_{\lambda_2}^k$. $e_{x,y}$ is the silhouette image. $r$ blends between tensor directions. Squaring the denominator emphasizes differences [S8, book].

## Background
The concept of compositing in volume rendering originated with the need to efficiently render 3D scalar fields from medical imaging data. The "Accumulate" mode, representing standard front-to-back alpha blending, is the foundational method described in early volume rendering literature [S2, unofficial]. The VolumePro system, developed at MERL, popularized support for multiple blending modes including Front-to-back blending, Minimum Intensity, and Maximum Intensity blending, providing a hardware-accelerated framework for these operations [S1, unofficial]. Maximum Intensity Projection (MIP) became widely used for extracting vessel structures in magnetic resonance angiograms [S2, unofficial]. Other variants like "First Hit" (for isosurface rendering) and "Averaging" (for X-ray-like images) were introduced to address specific visualization needs [S2, unofficial]. The theoretical basis for commutative blending in unsorted cell projection was established by Weiler and Kraus, showing that certain optical models allow compositing in arbitrary order [S3].

## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S3] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S4] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S5] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S6] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
[S7] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S8] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
