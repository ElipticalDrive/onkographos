# Acceleration

Also called: Volume rendering optimization, Rendering speedup

## Overview
Acceleration techniques in volume rendering reduce the computational cost of generating images by minimizing redundant sampling, skipping invisible data, or parallelizing work. This family solves the bottleneck of evaluating the volume rendering integral for every ray or voxel, which is often prohibitively expensive for high-resolution datasets or neural fields. Unlike [Image-order rendering](16-image-order-rendering.md) which defines the traversal order (eye-space), acceleration methods optimize the *content* of that traversal—whether by skipping empty space, terminating rays early, or adapting resolution—without changing the fundamental image-order nature of the parent.

## Shared principles
The core mechanism shared by all acceleration methods is the reduction of work units (samples, voxels, or pixels) processed per frame while maintaining acceptable visual fidelity. This is achieved through three primary strategies:
1.  **Culling/Skipping:** Identifying regions of the volume or ray path that do not contribute to the final image (e.g., empty space, occluded voxels) and bypassing their evaluation [S1, S2, S3, unofficial][S4][S5, S6, unofficial].
2.  **Termination:** Halting the integration loop when the accumulated contribution becomes negligible (e.g., high opacity, low transmittance) [S1, S2, S3, unofficial][S4][S5, S6, unofficial].
3.  **Adaptation:** Varying the computational effort based on local scene complexity or global performance budgets, such as adjusting sample density, step size, or output resolution [S1, S2, S3, unofficial][S4][S5, S6, unofficial].

Mathematically, these methods modify the standard front-to-back compositing loop:
$$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
$$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
by introducing conditions that break the loop early, skip iterations, or reduce the number of $C_{sample}$ queries [S1, S2, S3, unofficial][S4][S5, S6, unofficial].

## Subtopics

### [GL-NeRF](803-gl-nerf.md)
A training-free acceleration for Neural Radiance Fields that replaces stochastic sampling with deterministic Gauss-Laguerre quadrature, reducing MLP calls by selecting points based on accumulated optical depth roots.

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
An acceleration for texture-based rendering that skips occluded sub-volumes using a low-resolution opacity map orthogonal to the view direction, operating at the sub-volume level rather than per-voxel.

### [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md)
A family of techniques that dynamically adjust sample count, step size, or image resolution based on local metrics (e.g., density, variance) or global performance budgets to balance fidelity and cost.

### [Kernel Selection](1647-kernel-selection.md)
A pre-processing strategy for Gaussian ellipsoid rendering that rasterizes bounding boxes to select a fixed number of nearest, high-density kernels per pixel, discarding distant primitives before tracing.

### [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md)
Techniques that halt ray marching or path integration when opacity thresholds are met, distance bounds are exceeded, or probabilistic weights indicate negligible contribution.

### [Adaptive Refinement for Volume Rendering](1892-adaptive-refinement-for-volume-rendering.md)
An iterative image-order technique that refines image resolution based on local visual error (e.g., FEM-based error control), starting coarse and adding rays only where variation is high.

### [Conservative Early Ray Termination](1949-conservative-early-ray-termination.md)
An optimization for scan conversion that prunes cell processing by determining if a ray is sufficiently opaque using a subscreen-based lookup table (ERT-table), allowing parallel visibility decisions.

### [Inference Ray Marching and Integration](2148-inference-ray-marching-and-integration.md)
A GPU-accelerated pipeline for real-time NeRF inference that uses density bitfields/occupancy grids for early termination and fixed step sizes, disabling gradients to prioritize latency.

### [Ray-guided volume rendering with missing brick reporting](2815-ray-guided-volume-rendering-with-missing-brick-reporting.md)
An out-of-core technique that dynamically determines the working set by streaming only bricks accessed by rays, reporting missing bricks on the GPU for subsequent frames.

### [Intra-ray Parallelism](3135-intra-ray-parallelism.md)
A scheme that partitions each ray into depth bins to enable parallel accumulation of primitive contributions.

### [Acceleration Structure Rebuild](3266-acceleration-structure-rebuild.md)
Not given in the sources.

### [Spatiotemporal Importance Sampling](4696-spatiotemporal-importance-sampling.md)
A strategy for adaptive sampling that assigns higher probabilities to pixels occluded by tools or exhibiting motion, and lower probabilities to stationary tissues, to speed up convergence in surgical scenes.

### [Visibility Culling](6226-visibility-culling.md)
Identifies and skips processing of data regions or rays that do not contribute to the final image, using geometric or scalar-based determination of visibility before integration.

### [Parallelism and Distribution](6265-parallelism-and-distribution.md)
Distributes ray casting, compositing, or voxel processing across multiple processors or nodes to increase throughput, addressing computational bottlenecks via spatial decomposition and task scheduling.

### [Adaptive Multi-Step Empty Voxel Skipping](6289-adaptive-multi-step-empty-voxel-skipping.md)
Combines empty voxel skipping (space leaping) with adaptive multi-step length sampling to reduce redundant computation on empty regions.

### [Performance Optimization Techniques](6301-performance-optimization-techniques.md)
Not given in the sources.

### [Temporal Coherence and Caching](6372-temporal-coherence-and-caching.md)
Not given in the sources.

### Parsimonious Ray Sampling and Fused Rendering
- **Definition**: A rendering optimization that infers ray-BBox intersection to sample query points only within the scene bounding box, applicable only to bounded scenes [S1, unofficial]. It also fuses approximately ten point-wise operations for the forward and backward pass of the density-to-alpha function conversion [S1, unofficial].
- **Step**: In the volume rendering accumulation, halt tracking a ray once the accumulated transmittance falls below $10^{-3}$ [S1, unofficial].

### Inter-Ray Coherency
- **Definition**: A reference implementation that tracks rays from the eye point to first iso-surface intersection or volume exit, using fast voxel traversal and fast ray-surface intersection [S2, unofficial]. It exploits inter-ray coherency using a buffer storing cells encountered along the last ray and a flag indicating if an iso-surface portion is contained there [S2, unofficial].
- **Number**: This technique accelerates rendering by about 60 percent [S2, unofficial].

### Image-Order Volumetric Compositing
- **Definition**: A volume rendering framework proposed by Levoy, implemented on PIXEL-PLANES 5 hardware, using image-order volumetric compositing [S3, unofficial].
- **Step**: Use empty space skipping and occlusion culling to accelerate rendering speed [S3, unofficial].
- **Number**: The system achieved real-time or near real-time performance [S3, unofficial].

### Volume Level-of-Detail (LOD) Rendering
- **Definition**: A rendering process that maximizes image quality within user-specified frame rate and system constraints (RAM, texture memory, bandwidth) by adaptively modifying the set of rendered cells [S4].
- **Step**: 
  - **Adapt/Load Phase**: Determine which octree cells to render. Load non-resident textures from RAM or disk to texture memory. Maintain an 'active cut' (nodes covering space exactly once) [S4].
  - **Active Cut Maintenance**: Split queue contains non-leaf nodes on active cut, sorted by decreasing error. To decrease error: Move first split node to merge queue (mark inactive), add its children to split queue (mark active). To increase error (frame rate): Remove first node from merge queue to split queue (mark active), add its parent to merge queue, remove children from split queue (mark inactive) [S4].
  - **Cull/Render Phase**: Traverse octree from root. Perform hierarchical cull. Render flagged nodes using 3D texture mapping and blending in back-to-front order [S4].
- **Parameter**: User parameters: Desired frame rate, transfer function (data to RGBA), view position. System parameters: RAM, texture memory size, bandwidths, polygon and fill performance [S4].

### Early Ray Termination via Depth Buffer
- **Definition**: A technique to skip rendering fragments for rays that are already saturated (opaque) by testing against the depth buffer before executing the fragment shader for the brick [S5, unofficial].
- **Step**: 
  - Traverse the volume in front-to-back order to enable early termination [S5, unofficial].
  - Before rendering a brick, execute a fragment shader program while drawing a solid bounding box around the brick with back face culling switched on. This shader writes the maximal depth value in the depth buffer for saturated rays [S5, unofficial].
  - When slicing the brick texture, the early z-test prevents fragment operations for rays where the depth test fails (indicating saturation), reducing load on rasterization and fragment shader bottlenecks [S5, unofficial].
- **Formula**: Color update: $C_{i+1} = (1 - A_i) \cdot \alpha_i \cdot c_i + C_i$. Opacity update: $A_{i+1} = (1 - A_i) \cdot \alpha_i + A_i$. Where $C, A$ are current ray color/opacity, $c, \alpha$ are sample color/opacity, and $i$ is the sample index. A ray is saturated when $A_i \approx 1$ [S5, unofficial].
- **Pitfall**: Early ray termination is performed only once per brick, not per octree node or sample, because the overhead of changing fragment shaders and performing the test would annihilate benefits [S5, unofficial].

### Depth-Prior Ray Truncation
- **Definition**: Adjusts ray near/far bounds using a learned depth prior (disparity map) to restrict volumetric sampling to relevant depth ranges, reducing floater artifacts [S6, unofficial].
- **Step**: 
  1. Check `depthPriorLearned` flag. 
  2. If true, set `near_t = rayInfo[0] + groundTruth.disparityDevice[pixelIndex] * 0.5`. 
  3. Set `far_t = rayInfo[1] - 10.0`. 
  4. If false, use raw AABB intersection $t_{min}$ and $t_{max}$ [S6, unofficial].
- **Parameter**: `depthPriorLearned`: boolean flag enabling depth prior usage. `groundTruth.disparityDevice[pixelIndex]`: per-pixel disparity value from a depth prior. `0.5`: scaling factor applied to disparity. `10.0`: fixed offset subtracted from far bound [S6, unofficial].

## Background
Acceleration in volume rendering has evolved from hardware-specific optimizations to general algorithmic strategies. Early work by Levoy on image-order volumetric compositing demonstrated real-time performance using empty space skipping and occlusion culling on specialized hardware like PIXEL-PLANES 5 [S3, unofficial]. Subsequent research focused on software optimizations, such as cell-based first-hit ray casting which exploited inter-ray coherency to achieve significant speedups [S2, unofficial]. The advent of GPU rendering introduced techniques like early ray termination via depth buffers to mitigate fragment shader bottlenecks [S5, unofficial]. More recently, with the rise of Neural Radiance Fields (NeRF), acceleration has shifted towards optimizing neural network inference, including parsimonious ray sampling, fused operations, and deterministic quadrature methods like GL-NeRF [S1, S6, unofficial]. Out-of-core rendering techniques have also emerged to handle massive datasets by streaming data based on ray access patterns [S2, S1, unofficial].

## Sources
[S1] Improved Direct Voxel Grid Optimization for Radiance Fields Reconstruction, https://doi.org/10.48550/arxiv.2206.05085, fetched 2026-10-26 (unofficial)
[S2] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S3] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S4] PDF Interactive Visualization of Unstructured Grids Using Hierarchical 3D ..., https://web.eecs.umich.edu/~jjcorso/pubs/leven02unstructgrid.pdf, fetched 2026-10-27
[S5] PDF Optimizing GPU Volume Rendering - Danny Ruijters, https://dannyruijters.nl/docs/OptimizingGPUVolumeRendering_final.pdf, fetched 2026-10-26 (unofficial)
[S6] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
