# Ray Traversal and Integration Schemes

Also called: Ray Casting Integration, Volume Ray Casting Algorithms

## Overview
Ray Traversal and Integration Schemes constitute the core computational engine of direct volume ray casting, responsible for discretizing the viewing ray, sampling the scalar field, and accumulating optical properties to form the final pixel color [S1, S10, 4324, 4807, 4980, 6890, 9441]. This family solves the problem of efficiently evaluating the volume rendering integral along a line of sight while handling diverse data structures (voxels, neural fields, physical quantities) and rendering requirements (transparency, scattering, multi-material). What sets this family apart from its siblings is its focus on the *loop structure* and *aggregation logic*: while [Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md) defines how values are retrieved, and [Transfer Function and Opacity Mapping](2789-transfer-function-and-opacity-mapping.md) defines how values are mapped to color, the traversal schemes define *where* and *how often* to sample and *how* to combine those samples (e.g., deterministic front-to-back, Monte Carlo path tracing, or physical LOS integration).

## Shared principles
All methods in this family implement the emission-absorption optical model, computing the light intensity reaching the eye by integrating radiative energy along a ray through the scalar volume [S1, S5, S9, unofficial]. The process follows a strict pipeline for each pixel: ray generation, sampling, classification (transfer function application), shading, and compositing [S4, S34, unofficial].

### Ray Generation and Discretization
For each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the direction [S1, S2, S3, S4, S5, S6, S7, S8, S9, S10, S11, S12, S13, S14, S15, S16, S17, S18, S19, S20, S21, S22, S23, S24, S25, S26, S27, S28, S29, S30, S31, S32, S33, S34, S35, S36, S37, S38, S39, S40, S41, S42, S43, S44, S45, S46, S47, S48, S49, S50, S51, S52, S53, S54, S55, S56, S57, S58, S59, S60, S61, S62, S63, S64, S65, S66, S67, S68, S69, S70, S71, S72, S73, S74, S75, S76, S77, S78, S79, S80, S81, S82, S83, S84, S85, S86, S87, S88, S89, S90, S91, S92, S93, S94, S95, S96, S97, S98, S99, S100]. The ray is intersected with the volume domain (typically an axis-aligned bounding box) to determine the entry parameter $t_{start}$ and exit parameter $t_{end}$ [S1, S7, S15, S22, S33, S55, unofficial]. The segment $[t_{start}, t_{end}]$ is discretized into $N$ samples at equidistant intervals $\Delta t$ [S4, S8, S9, S16, S33, S40, unofficial]. The step size $\Delta t$ is often chosen to ensure at least one sample per voxel along the fastest axis to minimize aliasing, though adaptive stepping is also used [S36, unofficial][S46, abstract only, unofficial].

### Sampling and Interpolation
At each sample point $\mathbf{p}_i = r(t_i)$, the scalar volume value $V(\mathbf{p}_i)$ is reconstructed from the discrete voxel grid [S6, S7, S33, unofficial][S44, book]. Trilinear interpolation is the standard reconstruction filter for high-quality rendering, combining the eight surrounding voxels [S6, S7, S33, S40, unofficial]. Nearest-neighbor interpolation is used for performance or specific artifact styles [S43, unofficial][S44, book]. The interpolated value represents the local density or material property [S9, S18, unofficial].

### Classification (Transfer Function)
The scalar value $d_i = V(\mathbf{p}_i)$ is mapped to optical properties: color $C_i$ and opacity $\alpha_i$ [S2, S5, S11, S12, S25, S29, S31, S34, unofficial]. This mapping is defined by a transfer function $T(d_i)$ [S2, S5, S2789]. Pre-classification maps scalar values to RGBA before interpolation, while post-classification interpolates scalars first and then applies the transfer function [S4, unofficial][S44, book]. The opacity $\alpha_i$ determines the fraction of light absorbed or scattered at the sample [S18, S23, S51, unofficial].

### Compositing (Front-to-Back Integration)
The final pixel color $C_{final}$ is computed by integrating contributions from all samples along the ray from front to back ($t_{start}$ to $t_{end}$) [S1, S12, S16, S23, S24, S43, S50, S51, unofficial]. The standard compositing equation (over operator) updates the accumulated color $C'$ and accumulated opacity $\alpha'$ for each sample $i$:

$$
C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
$$
$$
\alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
$$

where $C'_0 = 0$ and $\alpha'_0 = 0$ [S12, S43, S50, unofficial]. The term $(1 - \alpha'_{i-1})$ represents the transmittance (light reaching the current sample) [S1, S51, unofficial]. If $\alpha'$ reaches 1 (or a threshold close to it), the ray can be terminated early, as subsequent samples are fully occluded [S9, S12, S27, unofficial].

### Shading
Shading models are applied to enhance depth perception. Common techniques include Phong or Blinn-Phong shading, which use the gradient of the volume data $\nabla V$ to compute surface normals for lighting calculations [S14, S15, S30, unofficial][S54]. The gradient is typically estimated using central differences or Sobel operators on the interpolated scalar field [S14, S30, unofficial][S54].

## Subtopics

### [Ray Marching and Discretization](2788-ray-marching-and-discretization.md)
An iterative direct volume rendering technique that approximates the volume rendering integral by casting rays from the camera through a 3D volume and accumulating optical properties at discrete sample points. It solves the problem of visualizing semi-transparent scalar fields without extracting explicit geometric surfaces by mapping data values to color and opacity via a transfer function and integrating the emission-absorption model along the viewing ray. Unlike fixed-step traversal methods, iterative variants advance rays using variable step sizes guided by distance estimators, enabling efficient handling of sparse, implicit, or procedural data. It processes the image pixel-by-pixel in a fragment shader, ensuring view-independent quality and supporting early ray termination.

### [Front-to-Back Volume Ray Casting with HSV Shading and SOS/Attenuation/Reflection Filtering](4324-front-to-back-volume-ray-casting-with-hsv-shading-and-sos-at.md)
This method is a direct volume rendering technique that casts rays from the camera through each pixel into a 3D volume, samples voxel values along the ray, and composites them front-to-back to produce the final pixel color. It distinguishes itself by applying filtering based on Scalar Output Scalar (SOS), Attenuation, and Reflection values, converting volume data to RGB using HSV color space, and using specific thresholding to discard irrelevant samples. It builds on [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md) by implementing a fixed-step front-to-back integration loop with early termination.

### [LOS-integration for visualization](4807-los-integration-for-visualization.md)
LOS-integration for visualization computes scalar physical quantities (electron column density, integrated emissivity, or emission-weighted temperature) along camera rays by integrating sampled neural network or ground truth field outputs. It differs from standard ray marching by explicitly handling unit conversions to physical consistency (cm, ergs, photons) and supporting spectral channel selection via a lookup table (LUT) for emissivity coefficients. It builds on [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md).

### [Monte Carlo Path Tracing for Volume Rendering](4980-monte-carlo-path-tracing-for-volume-rendering.md)
A volume rendering technique that uses Monte Carlo path tracing to simulate light scattering within participating media, calculating transmittance and accumulating results over multiple frames to reduce noise. It distinguishes itself from deterministic ray marching by using stochastic distance sampling and exponential moving average accumulation, trading convergence speed for unbiased scattering behavior.

### [Ray Traversal Schemes](6890-ray-traversal-schemes.md)
Ray traversal schemes aggregate scalar or color values along a ray to produce a 2D image, serving as the final compositing step in direct volume ray casting. Unlike [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) which focuses on sample placement, or [LOS-integration for visualization](4807-los-integration-for-visualization.md) which handles physical light transport, these schemes define the mathematical aggregation function (e.g., max, average, or alpha compositing) applied to the discrete samples. They are distinguished by their specific aggregation logic, ranging from simple extrema extraction to physically-based opacity accumulation.

### [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md)
Multi-layer volume ray casting is a unified rendering framework that treats volume datasets and geometric objects (clipping geometry, embedded opaque/semi-transparent objects, editing masks, MPRs, non-empty-space boundaries) as Volume-of-Interests (VOIs). It solves the problem of rendering complex, overlapping spatial regions with different material properties and priorities by encoding all VOI information into a pre-generated **ray-layer buffer**. Unlike standard ray marching which processes a single continuous ray or polygon-assisted methods that store full 3D coordinates, this method iterates through non-overlapping ray segments stored in the buffer, allowing material-specific transfer functions and shading per segment while maintaining bricking compatibility.

#### Spherical Volume Rendering with Adaptive Ray Casting
- **Definition**: A method for expressing ocean scalar fields in a 3D digital Earth environment using a minimum bounding volume based on spherical mosaic as proxy geometry and adaptive sampling [S1, abstract only, unofficial].
- **Steps**:
    1. Construct a minimum bounding volume based on spherical mosaic as the proxy geometry [S1, abstract only, unofficial].
    2. Apply the depth texture of the seabed terrain to determine the position of sampling points in the spatial interpolation process, realizing the fusion of ocean scalar field and seabed terrain [S1, abstract only, unofficial].
    3. Propose an adaptive sampling step algorithm based on heterogeneous depth distribution and data change rate to improve ray-casting efficiency [S1, abstract only, unofficial].
    4. Apply a nonlinear color-mapping enhancement scheme based on dataset skewness characteristics [S1, abstract only, unofficial].
    5. Design a transparency transfer function to realize volume rendering and local feature structure extraction of ocean scalar field data [S1, abstract only, unofficial].

## Background
The foundational concept of ray casting for volume rendering was introduced by Levoy in 1989, who proposed the direct volume rendering algorithm that samples along rays and composites front-to-back [S1, S10, unofficial]. The shear-warp factorization, mentioned in [S1, abstract only, unofficial], provides a fast method for rendering rotated 3D volumes by decomposing the viewing transform into shear and warp operations, though modern GPU implementations often bypass this for direct fragment shader execution. The family has evolved to include Monte Carlo methods for participating media [4980], physical LOS integration for scientific visualization [4807], and multi-layer buffering for complex scene composition [9441].

## Sources
[S1] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
