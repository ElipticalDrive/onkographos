# Compositing and Blending

Also called: Accumulate, Alternative Compositing Schemes, Blending Modes, First Hit, Front-to-back blending, Maximum Intensity blending, Minimum Intensity blending

## Overview
Compositing and Blending is the stage in direct volume ray casting that accumulates optical properties (color and opacity) from discrete volume samples along a ray into a final 2D pixel value [S1, S2, unofficial]. It solves the problem of projecting a 3D density field onto a 2D image plane while preserving internal structure and translucency, eliminating the need for binary classification [S3]. Unlike [Alpha Blending](5750-alpha-blending.md), which typically blends two distinct layers, or [Pre-Integration Lookup](2854-pre-integration-lookup.md), which approximates integration via texture lookups, standard compositing explicitly accumulates samples along the ray using the emission-absorption model [S3].

## Shared principles
All compositing methods in this family operate on a sequence of samples $(c_i, \alpha_i)$ encountered along a ray, where $c_i$ is the sample color (RGB) and $\alpha_i$ is the sample opacity (A) [S1, S2, unofficial]. The core mechanism involves an accumulator state $(C_{acc}, \alpha_{acc})$ initialized to $(0, 0)$ (or background color/opacity) and updated iteratively for each sample [S1, S2, unofficial].

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

### [Compositing](2436-compositing.md)
[Compositing](2436-compositing.md) combines multiple contributions along a ray, a step required only for semi-transparent rendering. It solves the problem of projecting a 3D density field onto a 2D image plane while preserving internal structure and translucency, eliminating the need for binary classification. Unlike [Alpha Blending](5750-alpha-blending.md), which typically blends two distinct layers, or [Pre-Integration Lookup](2854-pre-integration-lookup.md), which approximates integration via texture lookups, standard compositing explicitly accumulates samples along the ray using the emission-absorption model.

### [Final Scene Composition](2722-final-scene-composition.md)
[Final Scene Composition](2722-final-scene-composition.md)

### [Pre-Integration Lookup](2854-pre-integration-lookup.md)
[Pre-Integration Lookup](2854-pre-integration-lookup.md) accelerates direct volume ray casting by replacing per-sample compositing with a single texture fetch per slab, using pre-calculated optical properties that account for the integration of light and attenuation across a volume slice. It sets itself apart from standard per-sample compositing [Compositing] by assuming linear data evolution within a slab and encoding the cumulative effect of multiple samples into a 3D texture indexed by entry density, exit density, and slab thickness.

### [Binary Swap Compositing](3264-binary-swap-compositing.md)
[Binary Swap Compositing](3264-binary-swap-compositing.md) is a parallel image merging algorithm used to combine independently rendered sub-images from multiple processors into a final volume-rendered image. It solves the bottleneck problem of naive binary compositing, where processors become idle as the compositing tree narrows, by ensuring every processor participates in every stage of the merge. It sets itself apart from sibling methods like [Compositing](2436-compositing.md) by explicitly managing distributed memory communication via a binary-swap tree structure, and from [Pre-Integration Lookup](2854-pre-integration-lookup.md) by operating on already-composited sub-images rather than raw samples.

### [Blend Method](3390-blend-method.md)
[Blend Method](3390-blend-method.md) is a texture-based volume rendering technique that offloads the compositing operation to graphics hardware by resampling volume slices parallel to the image plane into a frame buffer and using the hardware's blending unit (specifically the "over" operator) to composite them. It differs from the Planes method, which performs compositing in software on the CPU, by leveraging hardware acceleration for the accumulation step, making it significantly faster for fully translucent volumes where rays rarely reach full opacity.

### [Screen-space Post-processing](3621-screen-space-post-processing.md)
[Screen-space Post-processing](3621-screen-space-post-processing.md) is a full-screen rendering pass that applies aspect-ratio preserving UV covering, luminance-based bright passes, color grading, specialized refraction effects, bloom, blurring, and dithering to a scene texture to produce the final displayable image. It differs from [Compositing](2436-compositing.md) by operating on the final rendered image rather than accumulating volume samples along rays, and from [HDR Post-Processing Pipeline](4633-hdr-post-processing-pipeline.md) by implementing specific spatial logic (such as underwater refraction and cross-shaped bloom) rather than fusing global tone-mapping and bloom into a single pass.

### [Layered SDF Compositing](3641-layered-sdf-compositing.md)
[Layered SDF Compositing](3641-layered-sdf-compositing.md) is a method for rendering Signed Distance Field (SDF) glyphs by compositing multiple layers (typically a face and one or more outline layers) using alpha blending and pre-multiplied alpha handling. It differs from standard volume ray casting by operating on discrete, pre-sampled layers of a 2D texture atlas rather than continuous 3D volume data, allowing for efficient, sharp text rendering with variable-width outlines.

### [Training Ray Compositing](3785-training-ray-compositing.md)
[Training Ray Compositing](3785-training-ray-compositing.md) is a differentiable volume rendering method that composites per-ray samples of density ($\sigma$) and color ($\mathbf{c}$) into final image colors, alpha weights, and depth, while simultaneously computing gradients with respect to the input parameters for training. It extends standard alpha compositing by including ambient light contributions and supporting a custom backward pass to propagate gradients from the loss function back to the density and color fields. It is distinguished from its sibling [Compositing](2436-compositing.md) by its explicit support for gradient computation through the transmittance chain, enabling end-to-end optimization of neural radiance fields.

### [Reconstruction and Compositing Techniques](4646-reconstruction-and-compositing-techniques.md)
[Reconstruction and Compositing Techniques](4646-reconstruction-and-compositing-techniques.md) encompasses methods that bridge the gap between discrete volumetric data and the continuous optical model required for ray casting, specifically focusing on how input data is reconstructed into a continuous field and how that field is composited along rays. It addresses the problem of aliasing and interpolation errors during sampling, as well as advanced compositing strategies that incorporate reconstruction quality or specific physical approximations. It differs from standard compositing by explicitly modeling the reconstruction kernel's effect on the signal or by modifying the compositing equation with validity weights, rather than treating samples as point values.

### [Lighting-Specific Compositing](5940-lighting-specific-compositing.md)
[Lighting-Specific Compositing](5940-lighting-specific-compositing.md) integrates lighting calculations (e.g., in-scattering, shadows) directly into the volume compositing pipeline rather than treating them as post-processes [G]. It solves the problem of efficiently approximating complex light transport effects within the ray-casting accumulation loop [G]. It differs from general [Compositing](2436-compositing.md) by embedding light-specific state into the accumulation logic, and from [Pre-Integration Lookup](2854-pre-integration-lookup.md) by avoiding precomputed lookup tables in favor of dynamic accumulation [G].

### [Layer-Centric Volume Rendering Framework](6994-layer-centric-volume-rendering-framework.md)
[Layer-Centric Volume Rendering Framework](6994-layer-centric-volume-rendering-framework.md) is a framework where the final image is obtained by compositing a number of renderings, each represented as a separate layer, providing a rich set of 2D operators and interactions.

### [MAB Compositing with Gaze-Contingent Blending](8211-mab-compositing-with-gaze-contingent-blending.md)
[MAB Compositing with Gaze-Contingent Blending](8211-mab-compositing-with-gaze-contingent-blending.md) is a compositing technique that maps framebuffer fragments to a Multi-Anchor Buffer (MAB) to retrieve and blend color values from different resolution levels based on gaze distance.

### [IDVR-Compositing](8316-idvr-compositing.md)
[IDVR-Compositing](8316-idvr-compositing.md) is a post-processing compositing step in Importance-Driven Volume Rendering (IDVR) that modulates the visual prominence of objects based on their importance and occlusion status. It solves the problem of visual clutter in multi-object scenes by assigning a "sparseness" value to each object along each ray, making occluded or less important objects more transparent while keeping important objects dense. Unlike standard front-to-back compositing which blends all samples equally, IDVR-Compositing introduces a view-dependent weighting factor before the final global blend.

### [Multi-view image blending](9831-multi-view-image-blending.md)
[Multi-view image blending](9831-multi-view-image-blending.md)

### Commutative Blend Direct Volume Rendering
Rendering technique for unsorted cell projection using optical models that lead to a commutative blend function, allowing compositing in arbitrary order [S3]. Models allowing arbitrary order compositing include maximum intensity projection and restrictions of the full volume density model to either the source term or the absorption term [S3]. Arbitrary transfer functions are applied via texture lookup, which only affect the generation of the texture map [S3].

### Frame Buffer Copy / Blit
A pass that simply copies the current frame to the output, acting as a simple integration or blit step for the LAO renderer [S4, unofficial]. The step involves sampling the frame buffer: `oColor = texture(uFrame, vPosition);` [S4, unofficial].

### Virtual Dimension Expansion for Compositing
A post-processing step where partial images rendered from blocks of different resolutions are expanded to a common 'virtual dimension' (the highest resolution dimension of the block) before compositing [S5, unofficial]. This allows standard compositing algorithms to work despite varying input resolutions [S5, unofficial]. The virtual dimension can be set to the dimension of the largest subimage in the scene to minimize padding, or fixed to the global high-resolution block size [S5, unofficial].

### Hybrid Compositing (Radix-k)
Hybrid methods like Radix-k combine direct send and binary swap approaches depending on the problem setting, adaptively running one or the other based on criteria [S6, unofficial].

## Background
The concept of compositing in volume rendering originated with the foundational work on direct volume ray casting. The "Accumulate" mode, representing standard front-to-back blending, is the most common scheme [S2, unofficial]. Alternative schemes such as "First Hit" (rendering an isosurface by taking the first sample exceeding a threshold), "Averaging" (producing X-ray-like images), and "Maximum Intensity Projection" (MIP) were introduced to handle specific visualization tasks [S2, unofficial]. MIP is often used to extract vessel structures in magnetic resonance angiograms [S2, unofficial].

The VolumePro system demonstrated support for front-to-back blending, minimum intensity, and maximum intensity blending modes, establishing these as standard blending modes in real-time ray-casting systems [S1, unofficial]. Hardware-based view-independent cell projection methods further expanded the field by enabling commutative blend functions for unsorted cell projection, allowing compositing in arbitrary order [S3].

Parallel compositing techniques, such as binary swap compositing, were developed to address scalability issues in distributed rendering environments, ensuring all processors participate in every stage of the merge [S6, unofficial]. Hybrid methods like Radix-k combine direct send and binary swap approaches to adapt to different problem settings [S6, unofficial].

## Sources
[S1] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S3] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S4] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S5] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S6] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
