# Interpolation and Sampling Techniques

Also called: Trilinear Interpolation (DRR), Trilinear rendering

## Overview
This family defines how discrete voxel data is reconstructed into a continuous scalar field and how sample positions are determined along a ray to minimize aliasing and discretization artifacts. It solves the problem of evaluating density or opacity at arbitrary ray positions that do not align with voxel grid centers, balancing computational cost against spatial smoothness [S1, unofficial][S2][S3]. It sets itself apart from sibling methods like [Supersampling](3493-supersampling.md) by defining the *reconstruction kernel* (the shape of the filter) rather than the *sampling pattern* (the number of rays), and from [Frequency Domain Volume Rendering with Sinc Resampling](826-frequency-domain-volume-rendering-with-sinc-resampling.md) by operating in the spatial domain using fixed local kernels rather than global Fourier transforms [S2, S3].

## Shared principles
All interpolation schemes in this family map a query position $\mathbf{p}$ in voxel space to a scalar value $V(\mathbf{p})$ by combining a finite set of neighboring voxel values $V_i$ using a set of weights $w_i(\mathbf{p})$:
$$ V(\mathbf{p}) = \sum_{i} w_i(\mathbf{p}) V_i $$
The weights are derived from a reconstruction kernel $h(\mathbf{z})$, where $\mathbf{z}$ is the distance from the query point to the sample center [G]. The choice of kernel defines the trade-off between computational complexity, spatial extent (number of neighbors), and frequency response (aliasing vs. blurring) [S1, unofficial][S2].

## Subtopics

### [Interpolation Schemes for Volume Rendering](817-interpolation-schemes-for-volume-rendering.md)
Interpolation schemes for volume rendering reconstruct a continuous scalar field from discrete voxel samples to evaluate density or opacity at arbitrary ray positions. This family solves the problem of discretization artifacts (aliasing, blockiness) inherent in nearest-neighbor sampling while balancing computational cost against spatial smoothness. It sets itself apart from sibling methods like [Supersampling](3493-supersampling.md) by defining the *reconstruction kernel* (the shape of the filter) rather than the *sampling pattern* (the number of rays), and from [Frequency Domain Volume Rendering with Sinc Resampling](826-frequency-domain-volume-rendering-with-sinc-resampling.md) by operating in the spatial domain using fixed local kernels rather than global Fourier transforms.

### [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md)
Multi-Sample Anti-Aliasing via Ray Distribution reduces jagged edges and aliasing artifacts in volume rendering by casting multiple rays per pixel, distributed within the pixel area, and averaging the results. It sets itself apart from standard single-ray casting by explicitly addressing the coarse granularity of per-pixel sampling, which causes abrupt intensity changes along object silhouettes. Compared to [Supersampling](3493-supersampling.md), which generally implies a uniform grid increase in resolution, this method often employs specific offset patterns (such as parity-based or fixed vector offsets) to distribute samples efficiently within the pixel footprint.

### [Depth to Distance Conversion](2428-depth-to-distance-conversion.md)
Depth to Distance Conversion transforms sampled depth values ($z_{vals}$) along a ray into segment lengths ($dists$) required for volume rendering integration. It accounts for the viewing angle by scaling the depth differences with the cosine of the angle between the ray direction and the viewing direction. This method is distinct from [Interpolation Schemes for Volume Rendering](817-interpolation-schemes-for-volume-rendering.md) as it focuses on metric length calculation rather than value estimation, and from [Sampling and Jittering](2516-sampling-and-jittering.md) as it operates on already sampled points.

### [Alpha Grid Mask](2430-alpha-grid-mask.md)
Alpha Grid Mask is a module for sampling alpha (opacity) values from a precomputed 3D alpha volume using trilinear interpolation. It solves the problem of incorporating spatially varying opacity constraints into volume rendering pipelines, allowing external supervision or geometry masks to influence the final image. It differs from direct intensity sampling by applying a linear remapping to sampled values to ensure they fall within a valid $[min\_opacity, max\_opacity]$ interval, preventing complete occlusion or invisibility artifacts.

### [Sampling and Jittering](2516-sampling-and-jittering.md)
Sampling and Jittering is a direct volume rendering technique that casts rays from the camera through a 3D voxel grid, samples voxel data with jittered step offsets to reduce banding artifacts, and composites the result using front-to-back alpha blending. It distinguishes itself from uniform ray sampling by perturbing sample positions to break up regular grid patterns, and from stochastic methods by using a deterministic, pre-computed jitter texture rather than runtime random number generation.

### [Supersampling](3493-supersampling.md)
Supersampling is a technique to minimize aliasing artifacts by casting multiple rays per pixel and averaging the results, effectively sampling the scene at a higher spatial frequency than the displayed resolution. It addresses the coarse granularity of per-pixel sampling, which causes visible artifacts such as jagged edges (staircase artifacts) and moiré patterns, particularly along object silhouettes where intensity changes abruptly. Unlike standard single-sampling, supersampling provides a more accurate color representation for each pixel by integrating the signal over the pixel area.

### [Epipolar Sampling](3899-epipolar-sampling.md)
Epipolar sampling is a technique for aggregating features from multiple views by sampling along epipolar lines rather than along camera rays in 3D space. It solves the problem of efficiently correlating information between a reference view and target views without performing full 3D volume rendering or explicit 3D reconstruction during the encoding phase. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths. This allows the encoder to memorize the correct depth consistent with the scene's arbitrary scale, which is then used for subsequent parameter prediction.

### [Voronoi-Based Foveated Volume Rendering](6463-voronoi-based-foveated-volume-rendering.md)
Voronoi-Based Foveated Volume Rendering is a method that adapts sampling density and reconstruction to the human visual system's acuity fall-off, using gaze tracking to shift a pre-computed sampling mask and reconstructing images via Voronoi-based natural neighbor interpolation. It sets itself apart from standard foveated techniques by mathematically aligning the sampling strategy (Voronoi cells) with the reconstruction kernel (natural neighbor interpolation), ensuring symmetry between how rays are placed and how values are interpolated, while explicitly addressing the undersampling artifacts inherent in sparse peripheral sampling through temporal smoothing.

### Slab-based Ray Sampling
Defines each slab of thickness $2d$ by three successive samples ($s_f, s_m, s_b$). The next slab is indexed by $(s_f, s_m, s_b)$ with the correlation $s_f = s_b$ (front sample of next slab equals back sample of current sample) [S2]. The method retrieves scalar values for $s_f, s_m, s_b$ from the 3D volume and uses a 3D texture-dependent fetch in a 3D pre-integration table to determine color and opacity values, blending successive slabs via front-to-back compositing [S2].

### Space Warping (Log+Warp)
Warps 3D space towards the view cell center using radial distortion to bring background closer for positional encoding, dampening high frequencies in the background while preserving foreground detail [S3]. The warping function is defined as $W(\mathbf{p}) = \frac{1}{\sqrt{|\mathbf{p}| \cdot d_{max}}}$, where $\mathbf{p}$ is the position relative to the view cell center [S3]. The warped feature vector is computed as $\mathbf{f} = \text{encode}((\mathbf{x}(\tilde{d}_i) - \mathbf{c}) \cdot W(\mathbf{x}(\tilde{d}_i) - \mathbf{c}))$, where $\tilde{d}_i$ is the logarithmic depth [S3].

### Space-covering volume sampling
A technique to extend pre-integrated volume rendering to multidimensional transfer functions [S4].

## Background
Trilinear interpolation is widely used in Digital Radiographic Reconstruction (DRR) for its speed advantage over exact intersection methods like Siddon's method [S1, unofficial]. In benchmarks, trilinear rendering with 25 points takes ~737 μs, while Siddon's method takes ~25 ms; trilinear rendering matches Siddon's cost only when the sample count reaches ~2,000 points [S1, unofficial]. The technique is implemented in libraries such as `diffdrr` [S1, unofficial]. Slab-based rendering with pre-integration tables was introduced to accelerate volume rendering by reducing the number of texture fetches and transfer function evaluations [S2]. Space warping techniques have been proposed in neural rendering contexts (e.g., DONeRF) to handle frequency discrepancies between foreground and background [S3]. Space-covering volume sampling extends pre-integration to multidimensional transfer functions [S4].

## Sources
[S1] Trilinear rendering - diffdrr, https://vivekg.dev/DiffDRR/tutorials/trilinear.html, fetched 2026-10-28 (unofficial)
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
[S3] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
[S4] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
