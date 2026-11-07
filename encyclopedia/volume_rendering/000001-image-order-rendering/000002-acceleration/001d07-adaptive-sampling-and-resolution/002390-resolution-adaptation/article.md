# Resolution Adaptation

Also called: Adaptive Resolution, Dynamic Resolution, Variable Resolution Rendering

## Overview
Resolution Adaptation encompasses techniques that dynamically adjust the spatial resolution of the rendered image or the underlying volumetric data to balance visual fidelity and computational performance. It solves the problem of fixed-cost rendering in scenarios where frame rate stability or memory bandwidth is constrained. Unlike [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md) which may adjust sample counts along rays, this family primarily focuses on changing the discretization density of the output image or the input data structure itself.

## Shared principles
The core mechanism involves a feedback loop or pre-defined schedule that modifies the resolution parameter $R$ (either image-space or data-space) based on a metric $M$.

1.  **Metric Evaluation:** Compute a metric $M$ derived from performance data (e.g., frame time $T_{draw}$), visual error (e.g., texel density, frequency content), or scene occupancy (e.g., alpha mask sparsity).
2.  **Resolution Adjustment:** Map $M$ to a new resolution level or factor $R_{new}$. This may involve:
    *   Scaling the viewport dimensions.
    *   Selecting a level-of-detail (LOD) index for data structures.
    *   Adjusting the voxel grid resolution of a neural field or voxel volume.
3.  **Rendering/Integration:** Render the scene using the adjusted resolution. This may require:
    *   Re-sampling the volume data.
    *   Rendering to a smaller framebuffer and upscaling.
    *   Selecting coarser data structures (e.g., lower mip levels, coarser octree nodes).
4.  **Stability/Coherence:** Ensure transitions between resolution levels are smooth or temporally coherent to avoid visual popping. This often involves clamping the rate of change or using temporal filtering.

## Subtopics

### [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md)
Level-of-detail (LOD) methods are techniques for interactive visualization of large scalar fields that maintain responsive frame rates by generating reduced-resolution approximations of the original data during viewpoint interaction. They solve the problem of computational intensity in volume rendering by dynamically adjusting the discretization of the rendering process—such as step size, sample count, or spatial resolution—based on a metric of visual error or performance. Unlike static hierarchical structures, these methods adapt in real-time to user manipulation, trading accuracy for speed to ensure interactivity.

### [Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md)
This method dynamically adjusts the spatial resolution of a volumetric representation and regenerates an alpha mask to prune empty space during training. It solves the problem of inefficient computation in sparse or evolving scenes by reducing the voxel grid size when the occupied volume is small and increasing it as detail is learned. It sets itself apart from static resolution methods by coupling grid upscaling with alpha mask culling and optimizer reinitialization, ensuring that computational resources are focused only on the active region of the scene.

### [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md)
Dynamic Resolution Adaptation is a runtime technique that adjusts the spatial resolution of the rendered viewport to maintain a target frame rate. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors.

### [Mixed-Resolution Multi-Volume Rendering](3193-mixed-resolution-multi-volume-rendering.md)
Mixed-resolution multi-volume rendering is a technique where different channels of a multi-volume dataset are rendered at different spatial resolution levels to save memory and reduce sample evaluation costs. It addresses the inefficiency of rendering all channels at the highest resolution when some channels have lower frequency content or lower visual importance. It is distinct from sibling methods like Level-of-detail (LOD) methods, which typically adjust resolution based on distance for the entire volume, and Dynamic Volume Resolution & Alpha Mask Update, which adjusts resolution based on occupancy or alpha thresholds [1007-level-of-detail-lod-methods.md][2539-dynamic-volume-resolution-alpha-mask-update.md].

### [Focus and Context Multiresolution Rendering](8163-focus-and-context-multiresolution-rendering.md)
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by explicitly decoupling the rendering pipeline into two distinct passes with different resolutions, sampling rates, and shading models, rather than dynamically adjusting a single ray's step size based on local metrics. The goal is not seamless integration but a tradeoff between quality and speed.

### [Multiresolution Atlas Buffer (MAB)](8219-multiresolution-atlas-buffer-mab.md)
The Multiresolution Atlas Buffer (MAB) is an acceleration wrapper for Direct Volume Rendering (DVR) in Virtual Reality (VR) that reduces computational cost by adaptively lowering ray density in peripheral vision and HMD-distorted regions, then reconstructing a full-resolution image via compositing. Unlike foveated rendering systems that rely on high-speed eye-tracking to locate precise gaze points, MAB leverages the fixed nature of HMD lenses and the non-uniform resolution of the human visual system to apply static, gaze-dependent resolution levels without tracking rapid eye movements. It differs from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by operating as a post-processing/compositing bracket around existing raycasters rather than modifying the ray traversal itself, and from [Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md) by adapting the output framebuffer resolution rather than the input volume grid resolution.

### Mixed Resolution Scene Rendering
- [S1] Combines full-resolution scene elements with low-resolution high-density volume data by rendering the volume to off-screen targets and projecting them onto a quad.
- [S1] Setup two off-screen cameras, one for each eye, updating each frame.
- [S1] Setup two low-resolution render targets (e.g., 200x200) for the cameras.
- [S1] Set up a quad that moves in front of the user.
- [S1] Each frame: Draw the low-resolution render targets for each eye (containing volume data and expensive shaders).
- [S1] Each frame: Draw the scene normally at full resolution (meshes, UI).
- [S1] Each frame: Draw a quad in front of the user and project the low-res renders onto it.

### Resolution and Frame Splitting
- [S2, unofficial] High-end: Reduce resolution to 1/2x1/2.
- [S2, unofficial] Mid/Low-end: Use frame splitting (similar to RDR2), simplified for clouds not intersecting other objects.

### Volume Level-of-Detail (LOD) Rendering
- [S3] A rendering process that maximizes image quality within user-specified frame rate and system constraints (RAM, texture memory, bandwidth) by adaptively modifying the set of rendered cells.
- [S3] Adapt/Load Phase: Determine which octree cells to render. Load non-resident textures from RAM or disk to texture memory. Maintain an 'active cut' (nodes covering space exactly once).
- [S3] Active Cut Maintenance: Split queue contains non-leaf nodes on active cut, sorted by decreasing error. To decrease error: Move first split node to merge queue (mark inactive), add its children to split queue (mark active). To increase error (frame rate): Remove first node from merge queue to split queue (mark active), add its parent to merge queue, remove children from split queue (mark inactive).
- [S3] Cull/Render Phase: Traverse octree from root. Perform hierarchical cull. Render flagged nodes using 3D texture mapping and blending in back-to-front order.
- [S3] User parameters: Desired frame rate, transfer function (data to RGBA), view position.
- [S3] System parameters: RAM, texture memory size, bandwidths, polygon and fill performance.

## Background
Resolution adaptation in volume rendering has evolved from static hierarchical structures (like Mip-maps and Octrees) to dynamic, feedback-driven systems. Early work focused on view-dependent LOD for unstructured grids, using active cuts and error metrics to manage memory and rendering cost [S3]. With the advent of neural rendering and VR, techniques have expanded to include dynamic resolution upscaling during training [2539-dynamic-volume-resolution-alpha-mask-update.md] and gaze-dependent rendering in HMDs [8219-multiresolution-atlas-buffer-mab.md]. Modern implementations often combine these approaches, using temporal coherence and frame-time budgets to adjust resolution in real-time [2683-dynamic-resolution-adaptation.md].

## Sources
[S1] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] PDF Interactive Visualization of Unstructured Grids Using Hierarchical 3D ..., https://web.eecs.umich.edu/~jjcorso/pubs/leven02unstructgrid.pdf, fetched 2026-10-27
