# Resolution Adaptation

Also called: Adaptive Resolution, Dynamic Resolution, Resolution Scaling

## Overview
Resolution Adaptation is a family of techniques that adjust the spatial discretization of the rendering process—such as voxel grid resolution, sampling step size, or viewport dimensions—to balance computational cost and visual fidelity. It solves the problem of performance instability in volume rendering, where static sampling rates often fail to maintain target frame rates under varying scene complexity or user interaction. Unlike static hierarchical structures, these methods adapt in real-time, trading accuracy for speed to ensure interactivity or stability [S1][S2, unofficial][S3].

## Shared principles
The core mechanism involves a feedback loop or metric evaluation that determines a resolution parameter $R$ (e.g., grid size, step size $\Delta t$, or viewport scale) and applies it to the rendering integral.

1.  **Metric Evaluation:** Compute a metric to determine the required level of detail. Common metrics include:
    *   **Performance-based:** The ratio of desired frame time to actual render time [S1].
    *   **Visual Error-based:** Metrics such as the ratio of "lone" or "tooth" texels to total texels in an auxiliary texture [S4, unofficial], or frequency content analysis [S1][S2, unofficial].
2.  **Discretization Adjustment:** Modify rendering parameters based on the metric. This may involve:
    *   Changing the voxel grid resolution $reso$ [S1].
    *   Adjusting the step size $\Delta t$ or sample count per ray [S2, unofficial].
    *   Scaling the viewport dimensions by a reduction factor $n \in (0.0, 1.0]$ [S1].
3.  **Integration:** Compute the volume rendering integral using the adjusted discretization. For sample-based simplification, the original geometry is used with approximate integration; for domain-based simplification, the domain geometry is approximated first [S1].
4.  **Stability:** Methods often employ smoothing, hysteresis, or clamping to prevent oscillation in resolution levels, which can cause visual artifacts and performance instability [S1].

## Subtopics

### [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md)
Techniques for interactive visualization of large scalar fields that maintain responsive frame rates by generating reduced-resolution approximations of the original data during viewpoint interaction. They solve the problem of computational intensity by dynamically adjusting discretization based on visual error or performance metrics. Unlike static hierarchical structures, these methods adapt in real-time to user manipulation, trading accuracy for speed.

### [Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md)
This method dynamically adjusts the voxel grid resolution and the active spatial bounding box of a neural volume representation during training to balance memory usage and rendering fidelity. It solves the problem of inefficiently allocating resources to empty space or overly dense regions by periodically shrinking the volume to its occupied core and upscaling the grid density at predefined intervals. It differs from [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md) by explicitly managing the *training* state (optimizer, learning rate, bounding box) rather than just the inference-time sampling rate, and from [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by modifying the underlying neural field's parameter space rather than selecting precomputed textures.

### [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md)
Dynamic Resolution Adaptation is a runtime performance management technique that adjusts the render viewport reduction factor based on previous frame render time and an allocated time budget to maintain a target frame rate. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors.

### [Mixed-Resolution Multi-Volume Rendering](3193-mixed-resolution-multi-volume-rendering.md)
Mixed-resolution multi-volume rendering is a technique where different channels of a multi-volume dataset are rendered at different spatial resolution levels to save memory and reduce sample evaluation costs. It addresses the inefficiency of rendering all channels at the highest resolution when some channels have lower frequency content or lower visual importance.

### [Volume Level-of-Detail (LOD) Rendering](4614-volume-level-of-detail-lod-rendering.md)
Volume Level-of-Detail (LOD) Rendering is a method that maximizes image quality within user-specified frame rate and system constraints (RAM, texture memory, bandwidth) by adaptively modifying the set of rendered octree cells. It differs from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by explicitly maintaining an "active cut" of nodes that cover the model space exactly once, rather than relying solely on geometric error metrics or fixed hierarchy levels. It builds on [Resolution Adaptation](3063-resolution-adaptation.md) by adjusting the discretization of the volume data itself (via octree traversal depth) rather than just the sampling rate or viewport size.

### [Focus and Context Multiresolution Rendering](8163-focus-and-context-multiresolution-rendering.md)
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by explicitly decoupling the rendering parameters (resolution, lighting, sampling) of two distinct spatial regions rather than adjusting a single global level of detail. The goal is not seamless integration but a tradeoff between quality and speed.

### [Multiresolution Atlas Buffer (MAB)](8219-multiresolution-atlas-buffer-mab.md)
The Multiresolution Atlas Buffer (MAB) is an acceleration wrapper for Direct Volume Rendering (DVR) in Virtual Reality (VR) that reduces ray density adaptively based on peripheral vision and Head-Mounted Display (HMD) lens distortion, then reconstructs the full-resolution image via compositing. It differs from standard foveated rendering by not requiring high-speed eye-tracking hardware, relying instead on the fixed nature of HMD lenses and the human visual system's (HVS) non-uniform resolution perception to define static gaze-dependent resolution zones. Unlike Mip-Maps which use fixed power-of-two scaling, MAB supports arbitrary resolution ratios between levels to minimize the visual difference between resolution steps.

#### Mixed Resolution Scene Rendering
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

## Background
Resolution adaptation in volume rendering has evolved from static hierarchical structures (like Mip-maps) to dynamic, feedback-driven systems. Early methods focused on precomputed LODs for interactive visualization of large scalar fields [S1][S2, unofficial]. With the advent of neural volume representations (e.g., NeRF), methods like Dynamic Volume Resolution & Alpha Mask Update introduced training-time adaptation of grid density and bounding boxes [S1]. In real-time applications, particularly VR and cloud rendering, techniques like Dynamic Resolution Adaptation and MAB have been developed to manage performance budgets and leverage human visual system properties [S1][S2, unofficial].

## Sources
[S1] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
