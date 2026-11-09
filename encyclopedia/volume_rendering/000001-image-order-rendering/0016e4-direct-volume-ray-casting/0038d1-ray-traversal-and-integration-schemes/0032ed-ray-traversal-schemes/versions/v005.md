# Ray Traversal Schemes

Also called: Accumulate Projection, Depth Projection, Max Intensity Projection, Ray Traversal -Accumulate

## Overview
Ray traversal schemes define the aggregation function $A(\{(C_i, \alpha_i)\}_{i=1}^N)$ that combines optical properties (color $C_i$ and opacity $\alpha_i$) sampled along a ray segment $[t_{start}, t_{end}]$ into the final pixel color $C_{final}$ and opacity $\alpha_{final}$ [S1, S2, S3, unofficial]. While the parent article [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md) establishes the common framework of sampling and mapping, this family distinguishes itself by the specific mathematical operator applied to the sequence of samples, ranging from simple extrema to probabilistic accumulation [S1, S2, S3, unofficial].

## Shared principles
All methods in this family operate on a ray segment $[t_{start}, t_{end}]$ defined by the intersection of the camera ray with the volume domain (typically an axis-aligned bounding box) [S1, S8, S16, unofficial]. For each sample point $\mathbf{p}_i$ along the ray, a scalar value $d_i$ is obtained via interpolation (typically trilinear) from the voxel grid or spatial data structure [S1, S2, S3, S4, unofficial]. This scalar is mapped to optical properties (color $C_i$ and opacity $\alpha_i$) via a transfer function $T(d_i) \rightarrow (C_i, \alpha_i)$ [S2, S2789]. The opacity is often corrected for the step size $\Delta t$ to account for discretization error, using the relation $\alpha'_i = 1 - (1 - \alpha_i)^{\Delta t}$ or similar approximations [S11, S32, unofficial]. The core distinction among the subtopics lies in the aggregation function $A(\{(C_i, \alpha_i)\}_{i=1}^N)$ that combines these samples into the final pixel color $C_{final}$ and opacity $\alpha_{final}$, and in the method used to determine the sample points $\mathbf{p}_i$ [S1, S2, S3, unofficial].

## Subtopics

### Intensity
Intensity projection extracts the scalar value of the sample with the highest density or intensity along the ray, effectively ignoring opacity and color compositing. It is primarily used for visualizing high-density structures where the absolute value is more important than the surface appearance [S1, S2, S3, unofficial].

### Max (Maximum Intensity Projection)
Maximum Intensity Projection (MIP) projects the maximum intensity value encountered along the ray to the pixel. It is widely used in medical imaging, specifically for Magnetic Resonance Angiography (MRA), to visualize bright vascular structures against darker backgrounds [S2, S3, unofficial].

### Average
Average projection computes the mean of the scalar values along the ray, producing an image similar to an X-ray. It provides a global view of the density distribution but lacks depth cues and surface detail [S2, S3, unofficial].

### Accumulate
Accumulate projection performs volumetric compositing by iteratively accumulating opacity and blending colors. This method makes transparent layers visible by weighting samples based on their opacity, allowing for the visualization of internal structures through semi-transparent surfaces. It is the standard method for direct volume rendering and was formalized by Levoy in 1988 [S1, S3, unofficial]. The front-to-back compositing equations are:
$$ C = C_s \cdot \alpha_s \cdot (1 - \alpha) + C $$
$$ \alpha = \alpha_s \cdot (1 - \alpha) + \alpha $$
where $(C_s, \alpha_s)$ are the sample color and opacity, and $(C, \alpha)$ are the accumulated color and opacity [S2, unofficial].

### First
First projection extracts the first sample along the ray that exceeds a certain threshold or represents an iso-surface. This method is used to extract specific iso-surfaces from the volume data [S1, unofficial].

### Depth
Depth projection records the distance $t$ along the ray where a specific condition is met (e.g., the first opaque sample or the maximum intensity sample). It is used to generate depth maps for 3D reconstruction or stereo rendering [S1, S2, S3, unofficial].

### Voxel Aggregation Methods
These variants focus on probabilistic or differentiable aggregation for learned representations.
- **Yan et al.**: Casts a ray and selects the voxel with the highest occupancy probability [S4, unofficial].
- **Tulsiani et al.**: Computes the probability that a ray stops at a certain distance, enabling the rendering of depth maps, color images, and foreground masks [S4, unofficial].
- **Henzler et al.**: Introduces a bidirectional emission-absorption (EA) light radiation model for emitting materials, in addition to visual hull (VH) and absorption-only (AO) models [S4, unofficial].
- **Lombardi et al.**: Iteratively accumulates color and opacity probabilities, discarding remaining voxels when cumulative opacity reaches a maximum value to prevent further impact on color [S4, unofficial].
- **Jiang et al.**: Locates the surface voxel closest to the camera for signed distance functions, computing pixel color as a weighted sum of the surface voxel and neighbors, and shading based on normals derived from voxel values. Note that while locating the surface point is not differentiable, the pixel color is differentiable with respect to surrounding voxels [S4, unofficial].

## Background
Ray traversal schemes are fundamental to Direct Volume Rendering (DVR). The Accumulate method, formalized by Marc Levoy in 1988, established the standard for volumetric compositing [S1, S3, unofficial]. Maximum Intensity Projection (MIP) became a standard clinical tool for angiography [S2, S3, unofficial]. Recent developments in differentiable rendering have introduced probabilistic aggregation methods (e.g., Tulsiani et al., Jiang et al.) to enable gradient-based optimization of 3D scene representations [S4, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/presentation/380929057/10-raycasting, fetched 2026-10-25 (unofficial)
[S3] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
[S4] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
