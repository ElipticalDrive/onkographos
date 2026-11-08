# Resolution Adaptation
Also called: Adaptive Resolution, Dynamic Resolution, Variable Resolution Rendering

## Overview
Resolution Adaptation refers to a family of techniques in volume rendering that dynamically adjust the spatial discretization of the rendering process—such as voxel grid resolution, step size, or framebuffer dimensions—to balance computational cost and visual fidelity [S1][S2, unofficial][S3]. It solves the problem of fixed computational budgets in interactive or real-time rendering, where static resolution leads to either wasted resources in empty space or insufficient detail in complex regions [S2, unofficial][S4, unofficial]. Unlike static hierarchical structures, these methods adapt in real-time to user manipulation, performance metrics, or training schedules, trading accuracy for speed to ensure interactivity or stability [S3][S1].

## Shared principles
The core mechanism of Resolution Adaptation involves a feedback loop or scheduled trigger that modifies the resolution parameters of the rendering pipeline. The shared mathematical principle is the mapping of a metric $M$ (performance, error, or iteration count) to a resolution parameter $R$ (grid size, step size, or viewport scale).

1.  **Metric Evaluation:** A metric is computed to determine the required level of detail.
    *   *Performance-based:* The ratio of desired frame time to actual render time [S1].
    *   *Visual Error-based:* Metrics such as the ratio of "lone" and "tooth" texels to total texels in an auxiliary texture [S4, unofficial].
    *   *Scheduled:* Iteration indices in training pipelines [S1].

2.  **Discretization Adjustment:** Rendering parameters are adjusted based on the metric.
    *   *Grid Resolution:* The voxel grid dimensions $(R_x, R_y, R_z)$ are upsampled or downsampled [S1].
    *   *Step Size:* The ray marching step size $\Delta t$ is increased or decreased [S2, unofficial].
    *   *Viewport Scale:* The render viewport reduction factor $R$ is adjusted [S1].

3.  **Integration/Rendering:** The volume rendering integral is computed using the adjusted discretization.
    *   For sample-based simplification, the original geometry is used with approximate integrals [S1].
    *   For domain-based simplification, the domain geometry is approximated first [S1].

4.  **Coherence and Stability:** Mechanisms are employed to prevent flickering or popping, such as clamping resolution changes, using auxiliary textures analyzed in parallel, or maintaining an "active cut" in hierarchical structures [S1][S4, unofficial].

## Subtopics

### [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md)
Level-of-detail (LOD) methods maintain responsive frame rates by generating reduced-resolution approximations of the original data during viewpoint interaction. They solve the problem of computational intensity in volume rendering by dynamically adjusting the discretization of the rendering process—such as step size, sample count, or spatial resolution—based on a metric of visual error or performance. Unlike static hierarchical structures, these methods adapt in real-time to user manipulation, trading accuracy for speed to ensure interactivity.

### [Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md)
This method dynamically adjusts the voxel grid resolution and prunes empty space in a neural volume representation (such as a Neural Radiance Field) during training to balance computational cost and geometric accuracy. It distinguishes itself from static resolution methods by coupling periodic resolution upscaling with alpha mask regeneration, ensuring that the active volume data structure shrinks to fit the occupied space. Unlike sibling methods that rely on spatial metrics for adaptive sampling, this approach follows a pre-defined schedule of iterations to trigger structural changes.

### [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md)
Dynamic Resolution Adaptation is a runtime performance management technique that adjusts the render viewport reduction factor based on previous frame render time and an allocated time budget to maintain a target frame rate. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors.

### [Mixed-Resolution Multi-Volume Rendering](3193-mixed-resolution-multi-volume-rendering.md)
Mixed-resolution multi-volume rendering is a technique where different channels of a multi-volume dataset are rendered at different spatial resolution levels to save memory and reduce sample evaluation costs. It addresses the inefficiency of rendering all channels at the highest resolution when some channels have lower frequency content or lower visual importance.

### [Focus and Context Multiresolution Rendering](8163-focus-and-context-multiresolution-rendering.md)
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications. It differs from seamless integration methods by explicitly accepting visual discontinuities to achieve a better tradeoff between achievable quality and speed. It builds on [Resolution Adaptation](3063-resolution-adaptation.md) by decoupling the rendering parameters of the focus and context regions rather than adapting a single global resolution.

### [Multiresolution Atlas Buffer (MAB)](8219-multiresolution-atlas-buffer-mab.md)
The Multiresolution Atlas Buffer (MAB) is an acceleration wrapper for Direct Volume Rendering (DVR) in Virtual Reality (VR) that reduces ray density adaptively based on peripheral vision and Head-Mounted Display (HMD) lens distortion, then reconstructs the full-resolution image via compositing. It differs from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by operating as a post-processing/compositing bracket around existing raycasters rather than modifying the traversal itself, and unlike [Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md), it adapts the output framebuffer resolution rather than the input volume data resolution.

#### Mixed Resolution Scene Rendering
- [S1] Combines full-resolution scene elements with low-resolution high-density volume data by rendering the volume to off-screen targets and projecting them onto a quad.
- [S1] Setup two off-screen cameras, one for each eye, updating each frame.
- [S1] Setup two low-resolution render targets (e.g., 200x200) for the cameras.
- [S1] Set up a quad that moves in front of the user.
- [S1] Each frame: Draw the low-resolution render targets for each eye (containing volume data and expensive shaders).
- [S1] Each frame: Draw the scene normally at full resolution (meshes, UI).
- [S1] Each frame: Draw a quad in front of the user and project the low-res renders onto it.

#### Resolution and Frame Splitting
- [S2, unofficial] High-end: Reduce resolution to 1/2x1/2.
- [S2, unofficial] Mid/Low-end: Use frame splitting (similar to RDR2), simplified for clouds not intersecting other objects.

#### Volume Level-of-Detail (LOD) Rendering
- [S3] A rendering process that maximizes image quality within user-specified frame rate and system constraints (RAM, texture memory, bandwidth) by adaptively modifying the set of rendered cells.
- [S3] Adapt/Load Phase: Determine which octree cells to render. Load non-resident textures from RAM or disk to texture memory. Maintain an 'active cut' (nodes covering space exactly once).
- [S3] Active Cut Maintenance: Split queue contains non-leaf nodes on active cut, sorted by decreasing error. To decrease error: Move first split node to merge queue (mark inactive), add its children to split queue (mark active). To increase error (frame rate): Remove first node from merge queue to split queue (mark active), add its parent to merge queue, remove children from split queue (mark inactive).
- [S3] Cull/Render Phase: Traverse octree from root. Perform hierarchical cull. Render flagged nodes using 3D texture mapping and blending in back-to-front order.
- [S3] User parameters: Desired frame rate, transfer function (data to RGBA), view position.
- [S3] System parameters: RAM, texture memory size, bandwidths, polygon and fill performance.

## Background
The concept of Level-of-Detail (LOD) in volume rendering originated in the early 2000s, with significant work on interactive visualization of unstructured grids using hierarchical 3D structures [S3]. The "Volume Level-of-Detail Rendering" process allows users to interactively choose a desired frame rate and other visualization parameters, maximizing image quality within system constraints like RAM and texture memory [S3]. The adapt/load phase modifies the set of rendered cells from the last frame to adapt to new viewing parameters, splitting some cells while merging others [S3].

In the context of Neural Radiance Fields (NeRF) and neural volume representations, dynamic resolution adaptation emerged as a method to manage the computational cost of dense voxel grids during training [S1]. This approach couples resolution upscaling with alpha mask regeneration to prune empty space [S1].

For real-time applications, particularly in Virtual Reality (VR) and Mixed Reality (MR), techniques like the Multiresolution Atlas Buffer (MAB) were developed to handle high HMD resolutions (e.g., 2880 x 1600) and maintain high refresh rates (90-120 Hz) by adapting ray density based on peripheral vision [S1]. Similarly, dynamic resolution adaptation for runtime performance management addresses frame rate instability in interactive volume rendering [S1].

In cloud rendering, resolution adaptation is often combined with temporal upsampling and frame splitting to handle high-density volume data on varying hardware configurations [S2, unofficial].

## Sources
[S1] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] PDF Interactive Visualization of Unstructured Grids Using Hierarchical 3D ..., https://web.eecs.umich.edu/~jjcorso/pubs/leven02unstructgrid.pdf, fetched 2026-10-27
