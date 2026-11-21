# Compositing and Blending

Also called: Accumulate, Alternative Compositing Schemes, Blending Modes, First Hit, Front-to-back blending, Maximum Intensity blending, Minimum Intensity blending

## Overview
Compositing and Blending is the family of algorithms responsible for integrating optical properties (color and opacity) accumulated along a viewing ray into a final pixel value. It solves the problem of combining semi-transparent volume samples to produce a physically plausible image based on the emission-absorption model [S1, S2, unofficial]. What sets this family apart from sibling methods like [Direct Volume Rendering](1184-direct-volume-rendering.md) (which may use surface-based approximations) or [Neural volume rendering](1576-neural-volume-rendering.md) (which learns representations) is the explicit mathematical accumulation of per-sample contributions, typically via the Porter-Duff "over" operator or its variants [S2, unofficial].

## Shared principles
The core mechanism across all compositing methods is the sequential update of accumulated state variables as samples are traversed along a ray. The standard optical model assumes that light is emitted or scattered by the volume and attenuated by absorption [S1, S2, unofficial].

For a ray traversing $N$ samples, each sample $i$ provides a color $C_i$ and opacity $\alpha_i$. The state is maintained in two accumulators:
1.  $C_{acc}$: The accumulated color.
2.  $\alpha_{acc}$: The accumulated opacity (or transmittance).

The standard **Front-to-Back** compositing equation updates these accumulators for each sample $i$ from front to back:
$$
C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i C_i
$$
$$
\alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i
$$
where $C_{acc}^{(0)} = 0$ and $\alpha_{acc}^{(0)} = 0$ [S12, S43, S50, unofficial]. The term $(1 - \alpha_{acc}^{(i-1)})$ represents the transmittance, i.e., the fraction of light reaching sample $i$ from the camera side [S1, S51, unofficial].

Alternatively, **Back-to-Front** compositing can be used, particularly for specific hardware optimizations or depth-based sorting, where the accumulation order is reversed [S1, unofficial].

Many implementations employ **Early Ray Termination (ERT)**. If the accumulated opacity $\alpha_{acc}$ exceeds a threshold close to 1.0 (e.g., $0.99$ or $1 - \epsilon$), the ray is terminated because subsequent samples are fully occluded [S9, S12, S27, unofficial].

## Subtopics

### [Compositing](2436-compositing.md)
The standard implementation of the front-to-back "over" operator. It explicitly accumulates samples along the ray using the emission-absorption model. It is distinguished from [Pre-Integration Lookup](2854-pre-integration-lookup.md) by performing iterative per-sample updates rather than slab-based texture lookups, and from [Alpha Blending](5750-alpha-blending.md) by being the general category under which the specific recursive accumulation logic falls. Choose this for general-purpose, high-accuracy direct volume rendering.

### [Pre-Integration Lookup](2854-pre-integration-lookup.md)
A method that pre-calculates the optical properties of volume slabs (defined by three scalar samples) into a 3D lookup table, assuming linear scalar evolution within the slab. It sets itself apart from standard [Compositing](2436-compositing.md) by replacing iterative per-sample integration with a single 3D texture fetch per slab, reducing aliasing from under-sampling. Choose this when rendering with large step sizes or when hardware texture filtering can approximate the integration.

### [Binary Swap Compositing](3264-binary-swap-compositing.md)
A parallel image merging algorithm for distributed volume rendering. It combines front-to-back compositing results from multiple MPI ranks into a final image using a binary tree communication pattern along one axis (typically Z). It is distinct from standard [Compositing](2436-compositing.md) because it operates on pre-composited image tiles from spatial partitions rather than raw samples. Choose this for large-scale parallel volume rendering on clusters.

### [Blend Method](3390-blend-method.md)
A texture-based technique that offloads compositing to graphics hardware by resampling volume slices parallel to the image plane into a frame buffer and using the hardware's blending unit. It differs from software-based [Compositing](2436-compositing.md) by leveraging hardware acceleration for the accumulation step, making it faster for fully translucent volumes. Choose this for real-time rendering on older or constrained hardware where ray casting is too expensive.

### [Screen-space Post-processing](3621-screen-space-post-processing.md)
A full-screen pipeline that applies effects like vignette, bloom, and underwater refraction to the final rendered image. It differs from [Compositing](2436-compositing.md) by operating on the final 2D image rather than accumulating volume samples along rays. Choose this for artistic stylization or enhancing the visual quality of the final output after volume rendering is complete.

### [Layered SDF Compositing](3641-layered-sdf-compositing.md)
A method for rendering Signed Distance Field (SDF) glyphs by compositing multiple layers (face and outlines) using alpha blending and pre-multiplied alpha handling. It differs from standard volume ray casting by operating on discrete 2D texture atlas layers rather than continuous 3D volume data. Choose this for rendering sharp text or UI elements with variable-width outlines.

### [Training Ray Compositing](3785-training-ray-compositing.md)
Composites ray colors and depths according to the volume rendering formula during the training of neural radiance fields (NeRF). It explicitly computes gradients for density and color parameters to enable backpropagation. It is distinguished from standard [Compositing](2436-compositing.md) by its dual forward/backward CUDA kernel implementation. Choose this when training differentiable volume rendering models.

### [Reconstruction and Compositing Techniques](4646-reconstruction-and-compositing-techniques.md)
A family encompassing methods for evaluating continuous fields (via polynomial composition) and resampling discrete data (via spatial convolution) to support ray casting. It is distinguished by focusing on the algebraic or spatial mechanics of data evaluation rather than lookup-table optimization. Choose this when high-fidelity reconstruction of the scalar field is required before compositing.

### [Alpha Blending](5750-alpha-blending.md)
A direct implementation of the Porter-Duff "over" operator for digital image compositing, discretizing the continuous integral using a Riemann sum. It is distinguished by its direct recursive accumulation of sample contributions weighted by accumulated opacity. Choose this for standard, straightforward direct volume rendering implementations.

### [Lighting-Specific Compositing](5940-lighting-specific-compositing.md)
Techniques that integrate lighting calculations (such as in-scattering or shadowing) directly into the compositing pipeline. It differs from general [Compositing](2436-compositing.md) by embedding light-specific state (e.g., scattered intensity buffers) into the accumulation logic. Choose this when approximating complex light transport effects like volumetric scattering within the ray-casting loop.

#### Commutative Blend Direct Volume Rendering
A rendering technique for unsorted cell projection using optical models that lead to a commutative blend function, allowing compositing in arbitrary order [S3]. It models allowing arbitrary order compositing include maximum intensity projection and restrictions of the full volume density model to either the source term or the absorption term [S3]. Arbitrary transfer functions are applied via texture lookup, which only affects the generation of the texture map [S3].

#### Final Scene Composition
A post-processing step where accumulated volumetric data is applied to the scene [S4, unofficial]. The linear depth of the opaque scene fragment is retrieved from the scene's depth buffer [S4, unofficial]. The fragment's screen coordinates and linear depth are converted into 3D texture coordinates $(u, v, w)$ using the exponential depth equation [S4, unofficial]. The integrated 3D texture is sampled using hardware trilinear interpolation (GL_LINEAR) [S4, unofficial]. The final color is computed as:
$$ \text{Final Color} = \text{Scene Color} \times \text{Transmittance} + \text{InScattered Light} $$
[S4, unofficial].

#### Frame Buffer Copy / Blit
A pass that simply copies the current frame to the output, acting as a simple integration or blit step for the LAO renderer [S5, unofficial]. The step involves sampling the frame buffer: `oColor = texture(uFrame, vPosition);` [S5, unofficial].

## Background
The concept of compositing in volume rendering originates from the application of the emission-absorption optical model to discrete volume data. Early systems like VolumePro introduced various blending modes including front-to-back blending, minimum intensity, and maximum intensity projection [S1, unofficial]. The standard "accumulate" mode corresponds to the classic front-to-back integration described in foundational texts [S2, unofficial]. Alternative schemes such as "First Hit" (for isosurfaces) and "Averaging" (for X-ray-like images) were also defined in early educational materials on visualization [S2, unofficial]. The development of hardware-based view-independent cell projection highlighted the need for commutative blend functions to allow unsorted cell projection [S3]. More recent developments in neural volume rendering have adapted these compositing equations to be differentiable for training [S1, S4, unofficial].

## Sources
[S1] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S3] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S4] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
[S5] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
