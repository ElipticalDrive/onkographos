# Lighting and Shading Techniques


## Overview
Lighting and Shading Techniques modulate the base color derived from transfer functions with a lighting term computed from local gradients or pre-computed illumination caches to provide realistic depth perception and highlight occluded regions in semi-transparent volumetric data [G]. Unlike standard direct volume rendering which often produces "cloud-like" or fuzzy images, these methods introduce specular and diffuse cues to reveal internal structures and surface-like features [G]. They differ from siblings like [Pre-integrated Volume Rendering](5327-pre-integrated-volume-rendering.md) by focusing on the illumination model rather than the compositing integration, and from [Next Event Estimation for Emissive Volumes](3557-next-event-estimation-for-emissive-volumes.md) by typically relying on local gradient-based approximations rather than explicit path-traced light sampling, though some variants integrate global effects via caching [G].

## Shared principles
The family operates on the scalar field $V(\mathbf{p})$ sampled at positions $\mathbf{p}$ along the ray [G]. The core mechanism involves a transfer function $T$ that maps the scalar value to a color vector $\mathbf{c} \in [0,1]^3$ and an opacity scalar $\alpha \in [0,1]$:
$$ T: V \rightarrow (\mathbf{c}, \alpha) $$
These values are subsequently used in the front-to-back compositing equation defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. Transfer functions are typically parameterized by control points $(V_k, \mathbf{c}_k, \alpha_k)$ with interpolation between points [G].

The opacity $\alpha$ is derived solely from the opacity transfer function and is generally not modified by the shading model, preserving the physical interpretation of density [G]. The shading model affects only the color channel [G].

The lighting term $L(\mathbf{p}_i)$ is typically computed using a Phong illumination model or a simplified intensity model [G]. This requires estimating the surface normal $\mathbf{n}_i$ at the sample point, usually by normalizing the gradient of the scalar field $\nabla V(\mathbf{p}_i)$ [G].

$$ \mathbf{n}_i = \frac{\nabla V(\mathbf{p}_i)}{\|\nabla V(\mathbf{p}_i)\|} $$

The gradient $\nabla V(\mathbf{p}_i)$ is estimated using finite differences [G]:

$$ \nabla V(\mathbf{p}_i) \approx \left( \frac{V(\mathbf{p}_i + \Delta x) - V(\mathbf{p}_i - \Delta x)}{2\Delta x}, \frac{V(\mathbf{p}_i + \Delta y) - V(\mathbf{p}_i - \Delta y)}{2\Delta y}, \frac{V(\mathbf{p}_i + \Delta z) - V(\mathbf{p}_i - \Delta z)}{2\Delta z} \right) $$

The Phong lighting term combines ambient, diffuse, and specular components [G]:

$$ L(\mathbf{p}_i) = k_a + k_d (\mathbf{n}_i \cdot \mathbf{l}) + k_s (\mathbf{r}_i \cdot \mathbf{v})^n $$

Where:
*   $k_a, k_d, k_s$: Ambient, diffuse, and specular coefficients [G].
*   $\mathbf{l}$: Unit vector from $\mathbf{p}_i$ to the light source [G].
*   $\mathbf{v}$: Unit view vector from $\mathbf{p}_i$ to the camera [G].
*   $\mathbf{r}_i$: Reflection vector of $\mathbf{l}$ about $\mathbf{n}_i$, calculated as $\mathbf{r}_i = 2(\mathbf{n}_i \cdot \mathbf{l})\mathbf{n}_i - \mathbf{l}$ [G].
*   $n$: Shininess exponent [G].

The shaded sample color $C_i$ is computed as:

$$ C_i = C_{base} \cdot L(\mathbf{p}_i) $$

This shaded color is then accumulated using the front-to-back alpha compositing equation defined in the parent [19-direct-volume-ray-casting.md] [G].

## Subtopics

### [Doppler Volume Rendering](3536-doppler-volume-rendering.md)
Doppler Volume Rendering is a spectral volume rendering technique that accounts for the Doppler effect of light in an emission-absorption model, where volumetric matter moves according to a 3D velocity field and emits/attenuates light at wavelengths subject to Doppler shifts. It employs a novel piecewise linear spectral representation to achieve interactive frame rates while maintaining energy conservation, overcoming the inaccuracies of traditional point-sampled spectral representations. Unlike standard direct volume ray casting which maps scalar values to RGB colors, this method operates in the spectral domain, allowing for the visualization of non-thermal radiation (spectral lines) and thermal radiation (blackbody spectra) with accurate wavelength shifts due to motion.

### [Next Event Estimation for Emissive Volumes](3557-next-event-estimation-for-emissive-volumes.md)
Next Event Estimation (NEE) for Emissive Volumes is a direct lighting estimation technique used in path tracing within participating media. It solves the problem of noisy or incorrect direct illumination from emissive volumes by explicitly sampling light sources and evaluating the emission along the ray segment between the sample point and the light source. Unlike standard volume shading methods that may rely on ambient terms or approximate scattering, this method ensures energy consistency by correctly handling the probability density functions (PDFs) associated with sampling directions in 3D space. It sets itself apart from siblings like [Direct Volume Rendering with Phong Illumination](3686-direct-volume-rendering-with-phong-shading.md) by explicitly accounting for the geometric attenuation and PDF conversion required when the light source is a volume rather than a surface, avoiding the pitfalls of naive solid-angle-to-volume PDF conversions.

### [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md)
Direct Volume Rendering with Phong Shading is a technique that projects 3D scalar volume data onto a 2D image plane using user-defined opacity functions to control visibility, combined with Phong illumination models to provide realistic depth perception. It solves the problem of visual ambiguity in semi-transparent volumetric data by highlighting occluded regions and surface-like structures through specular and diffuse lighting cues. Unlike Cel shading, which fails to highlight encircled or occluded regions appropriately, Phong shading provides superior visualization of internal structures and depth. It builds on the standard direct volume ray casting loop [19-direct-volume-ray-casting.md] by modulating the base color derived from transfer functions with a lighting term computed from local gradients [8281-lighting-and-shading-techniques.md].

### [Illumination Caching](7548-illumination-caching.md)
Illumination Caching is a rendering technique that pre-computes and stores the lighting distribution inside a volume in an illumination cache, drastically improving rendering performance. It solves the problem of expensive per-pixel lighting calculations in participating media by exploiting the assumption that volume material scatters light isotropically, allowing the lighting distribution to be decoupled from the ray-marching step. Unlike sibling methods like [Radiance Caching (Jarosz et al.)](8493-radiance-caching-jarosz-et-al.md) which model directional radiance distributions, or [Light Cache Rendering Pipeline](9159-light-cache-rendering-pipeline.md) which uses 3D Summed Area Tables for aggregate queries, standard illumination caching typically stores scalar irradiance values derived from isotropic scattering assumptions.

### [Extinction-based Shading and Illumination](9147-extinction-based-shading-and-illumination.md)
Extinction-based Shading and Illumination is a GPU-accelerated technique that computes ambient occlusion, color bleeding, and directional soft shadows in direct volume ray casting by aggregating exponential extinction coefficients using a 3D Summed Area Table (SAT). Unlike traditional ray-marching shadowing or opacity-based occlusion methods, it approximates spherical and conical neighborhoods with axis-aligned cuboid shells, enabling distance-weighted aggregation in constant time per sample. It sets itself apart from siblings like [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md) by providing global illumination effects (occlusion, bleeding) rather than just local surface reflection, and from [Extinction-based Shadow and Scattering Approximation](9149-extinction-based-shadow-and-scattering-approximation.md) by unifying these effects into a single SAT-based framework with specific cuboid-shell approximation strategies.

### [Extinction-based Shadow and Scattering Approximation](9149-extinction-based-shadow-and-scattering-approximation.md)
A technique to approximate directional shadows and light scattering by computing the extinction of light intensity for voxels using a 3D Summed Area Table (SAT), avoiding the propagation of illumination from outside the volume. It differs from half-angle slicing [14] by handling any light type and multiple sources, and from Ropinski et al.'s method by computing voxel extinction rather than propagating external illumination.

### [Image Plane Sweep Volume Illumination](10705-image-plane-sweep-volume-illumination.md)
Image Plane Sweep Volume Illumination is a GPU-based volume ray-casting technique that reduces illumination computation complexity by exploiting the plane sweep paradigm. It enables interactive frame rates while supporting advanced effects such as scattering and shadowing. Unlike standard ray casting which casts viewing rays directly from the viewpoint, this method computes illumination for a screen-space line first, then casts viewing rays line by line.

### [ShadedVolumeStyle](10856-shadedvolumestyle.md)
ShadedVolumeStyle is a rendering style node that calculates and applies lighting effects to the visual output of a volume. It operates within an emission-absorption model using a single scattering approximation. It is distinct from its siblings in that it is implemented as a composable style node (often within a `ComposedVolumeStyle`), allowing it to be easily combined with other styles such as `OpacityMapVolumeStyle` or `BoundaryEnhancementVolumeStyle` in a single rendering pass.

### [Kaufman's Shaded Volume Rendering](11206-kaufman-s-shaded-volume-rendering.md)
Kaufman's Shaded Volume Rendering is a direct volume ray casting technique designed to produce informative, rather than physically accurate, representations of volumetric data for scientific and engineering interpretation. It distinguishes itself from sibling methods like [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md) by explicitly decoupling the shading model from physical realism, allowing users to modify shading parameters to highlight specific features through a simplified intensity model that incorporates solid texture maps.

### [Color Bleeding via SH Projection](12232-color-bleeding-via-sh-projection.md)
This method computes indirect illumination (color bleeding) in direct volume rendering by projecting the accumulated radiance along random rays onto Spherical Harmonics (SH) basis functions. It solves the problem of capturing chromaticity effects—where the color of the medium changes as light travels through it—which simpler shading models ignore. Unlike sibling methods that rely solely on local gradient estimation or absorption-only models, this approach evaluates the full volume rendering integral to capture how the medium's color influences the scattered light.

### [On-the-fly pre-shading](13604-on-the-fly-pre-shading.md)
On-the-fly pre-shading is a fast technique that pre-shades a volume on the fly and then slices and composites an RGB volume to obtain an image with shading effects. It is distinguished from sibling methods like [Illumination Caching](7548-illumination-caching.md) by computing shading dynamically during the slicing process rather than caching it in a separate data structure, and from [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md) by deferring the accumulation of shading into a pre-shaded RGB texture space before compositing.

#### Interactive Directional Ambient Occlusion and Shadow for Volume Ray Casting
- [S1] A strategy to compute directional ambient occlusion and shadows for volume ray casting of structured datasets by evaluating Gaussian integrals at discrete samples along the cone axis.
- [S1] Given the extinction coefficient data volume, create one extra volume computing representative amplitudes of Gaussian functions.
- [S1] Use mipmapping to effectively evaluate Gaussian integrals with different sizes placed along the cone axis, adapting a circle packing approach.
- [S1] Use the computed occlusion to add directional ambient occlusion effects and generate shadows.
- [S1] Ray tracing implementation uses GLSL compute shader.
- [S1] Delivers a better balance between quality results and performance compared to previous specialized procedures, combining directional ambient occlusion and shadow generation under the same framework.

#### Direct Diffuse Illumination for Semi-Transparent Volumes
- [S2, unofficial] A method to enable direct diffuse illumination for semi-transparent volume rendering.
- [S2, unofficial] Multiple passes through rasterization hardware lead to significant loss in rendering performance.

#### One-pass shaded volume rendering
- [S2, unofficial] A one-pass approach employing 3D texture hardware interpolation together with software shading and classification.

#### Lighting Calculation
- [S1] A lighting technique that calculates illumination for volume data using two types of light sources: floodlights (directional, uniform) and point lights (omnidirectional from a specific location).
- [S1] Floodlight Calculation: For each point in the volume, march in the direction of the light. Track remaining light intensity. If an illuminated area is hit, reduce intensity based on opacity. Stop when light intensity reaches zero.
- [S1] Point Light Calculation: For each point in the volume, march towards the point light source. Check for occlusions along the path to determine if the point is illuminated.
- [S1] Light Shader: Contains intensity calculators for floodlights and point lights. Takes light parameters (direction for flood, position for point) as input.
- [S1] Implemented in the 'vrc light shader' or 'lighting texture' creation step of the library.

#### Composite with shading
- [S4] A rendering technique that displays the volume as a shaded surface. This is the default technique.

## Background
The use of shading in volume rendering originated from the need to improve depth perception in semi-transparent data, which often appears flat or "cloud-like" without surface cues [G]. Early approaches did not provide any shading, but VanGelder and Kim introduced a fast technique to pre-shade the volume on the fly and then slice and composite a RGB volume to obtain an image with shading effects [S2, unofficial]. Meißner et al. provided a method to enable direct diffuse illumination for semi-transparent volume rendering, though it suffered from performance losses due to multiple passes through rasterization hardware [S2, unofficial]. Dachille et al. proposed a one-pass approach employing 3D texture hardware interpolation together with software shading and classification [S2, unofficial].

The Phong illumination model, widely adopted in these techniques, was originally developed for surface rendering but adapted for volumes by estimating normals from scalar field gradients [G]. This adaptation allows for the visualization of internal structures and surface-like features in volumetric data [G].

Modern implementations leverage GPU acceleration, with techniques like Illumination Caching [7548-illumination-caching.md] and Extinction-based Shading [9147-extinction-based-shading-and-illumination.md] pre-computing lighting to achieve real-time frame rates [G]. Spectral methods like Doppler Volume Rendering [3536-doppler-volume-rendering.md] extend these principles to the spectral domain for scientific visualization of moving media [S1].

## Sources
[S1] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S2] Overview of Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] WebGL for Numerical PDEs (Session 11): Volume Ray Casting, https://www.youtube.com/watch?v=X095f9QzSBo, fetched 2026-10-27 (unofficial)
[S4] Volume rendering — 3D Slicer documentation, https://slicer.readthedocs.io/en/5.0/user_guide/modules/volumerendering.html, fetched 2026-10-27
