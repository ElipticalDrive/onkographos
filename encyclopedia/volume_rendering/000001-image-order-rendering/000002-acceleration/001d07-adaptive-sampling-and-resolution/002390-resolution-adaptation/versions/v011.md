# Resolution Adaptation

Also called: Adaptive Resolution, Dynamic Resolution, Resolution Scaling

## Overview
Resolution Adaptation is a family of image-order rendering techniques that modulate the spatial granularity of the rendering process—such as voxel grid resolution, ray step size, or screen-space viewport dimensions—to balance visual fidelity against computational cost [S1][S2, unofficial][S3]. It solves the problem of performance instability in volume rendering, where minor parametric changes or complex geometry can cause significant frame rate dips, by dynamically adjusting the discretization of the rendering integral based on real-time metrics like render time, visual error, or system constraints [S1][S2, unofficial][S3]. What sets this family apart from static hierarchical structures (like fixed MIP-maps) is the real-time, feedback-driven adaptation to user interaction or training progress, trading accuracy for speed to ensure interactivity or stable frame rates [S3].

## Shared principles
All methods in this family share the core mechanism of decoupling the rendering resolution from a fixed global constant, instead treating it as a variable state $R$ that is updated per-frame or per-training-step. The general loop is:
1.  **Metric Evaluation:** Compute a scalar metric $M$ derived from the previous frame's performance (e.g., render time $T_{render}$ vs. target time $T_{target}$) or visual quality (e.g., error estimate $\epsilon$).
2.  **Adjustment:** Update the resolution parameter $R$ (or a reduction factor $n$) based on $M$. If $T_{render} > T_{target}$, $R$ is decreased (coarser resolution); if $T_{render} < T_{target}$, $R$ is increased (finer resolution).
3.  **Rendering:** Perform the volume integration (ray marching, texture slicing, or neural field evaluation) using the adjusted resolution $R$.

Common mathematical forms for adjustment include:
*   **Proportional Control:** $R_{new} = R_{prev} \times \frac{T_{render}}{T_{target}}$ [S1].
*   **Threshold Hysteresis:** $R$ changes only if the metric crosses predefined upper/lower bounds $[S_{min}, S_{max}]$ to prevent oscillation [S4, unofficial].
*   **Scheduled Upscaling:** In training contexts, resolution is increased at discrete iteration steps $k$ according to a predefined list, often accompanied by grid interpolation and optimizer reset [S1].

The shared goal is to maintain a target performance metric (FPS or latency) while maximizing the resolution $R$ such that the visual error remains within an acceptable threshold $\epsilon$.

## Subtopics

### [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md)
LOD methods maintain responsive frame rates by generating reduced-resolution approximations of the original data during viewpoint interaction. They solve the problem of computational intensity in volume rendering by dynamically adjusting the discretization (step size, sample count, or spatial resolution) based on a metric of visual error or performance. Unlike static hierarchical structures, these methods adapt in real-time to user manipulation, trading accuracy for speed to ensure interactivity.

### [Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md)
This method dynamically adjusts the voxel grid resolution and the active spatial bounding box of a volume representation during training to balance rendering fidelity and computational cost. It solves the problem of inefficiently sampling empty space or using excessive resolution for static background regions by periodically pruning the volume and upscaling the grid only when necessary. It differs from [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md) by explicitly managing the 3D voxel grid structure and alpha masks rather than just screen-space resolution, and from [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by using a global grid upscaling strategy rather than hierarchical data structures.

### [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md)
Dynamic Resolution Adaptation is a performance-based image-order rendering technique that modulates the render viewport's reduction factor in real-time to maintain a target frame rate. It solves the problem of performance instability in volume rendering, where minor parametric changes (such as transfer function modifications) can cause significant frame rate dips. It sets itself apart from static resolution schemes by continuously shifting the resolution granularity based on the gradient of recent frame times, rather than using fixed blending factors or static thresholds.

### [Mixed-Resolution Multi-Volume Rendering](3193-mixed-resolution-multi-volume-rendering.md)
Mixed-resolution multi-volume rendering is a technique where different channels of a multi-volume dataset are rendered at different spatial resolution levels to save memory and reduce sample evaluation costs. It addresses the inefficiency of rendering all channels at the highest resolution when some channels have lower frequency content or lower visual importance.

### [Focus and Context Multiresolution Rendering](8163-focus-and-context-multiresolution-rendering.md)
Focus and Context Multiresolution Rendering is a technique that balances processing capacity between a high-detail "focus" region and a lower-detail "context" region to improve interactivity in time-critical volume rendering applications. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by explicitly separating the scene into two distinct rendering passes with different resolutions and rendering models, rather than adapting a single representation based on distance or error metrics. The goal is not seamless integration but a trade-off between achievable quality and speed.

### [Multiresolution Atlas Buffer (MAB)](8219-multiresolution-atlas-buffer-mab.md)
The Multiresolution Atlas Buffer (MAB) is an acceleration wrapper for Direct Volume Rendering (DVR) in Virtual Reality (VR) that reduces ray density adaptively based on peripheral vision and Head-Mounted Display (HMD) lens distortion, then reconstructs the full-resolution image via compositing. Unlike sibling methods that modify the ray traversal itself or use high-speed eye-tracking for foveated rendering, MAB operates as a "bracket" around existing raycasters, decoupling the adaptive sampling logic from the volume traversal algorithm. It is specifically designed to guarantee stable high refresh rates (90 Hz or higher) on high-resolution HMDs to prevent cyber-sickness, leveraging the fixed nature of HMD lenses where the highest resolution is naturally confined to a slight central angle.

#### Mixed Resolution Scene Rendering
*   **Definition:** Combines full-resolution scene elements with low-resolution high-density volume data by rendering the volume to off-screen targets and projecting them onto a quad [S1].
*   **Steps:**
    1.  Setup two off-screen cameras, one for each eye, updating each frame [S1].
    2.  Setup two low-resolution render targets (e.g., 200x200) for the cameras [S1].
    3.  Set up a quad that moves in front of the user [S1].
    4.  Each frame: Draw the low-resolution render targets for each eye (containing volume data and expensive shaders) [S1].
    5.  Each frame: Draw the scene normally at full resolution (meshes, UI) [S1].
    6.  Each frame: Draw a quad in front of the user and project the low-res renders onto it [S1].

#### Resolution and Frame Splitting
*   **Steps:**
    1.  High-end: Reduce resolution to 1/2x1/2 [S2, unofficial].
    2.  Mid/Low-end: Use frame splitting (similar to RDR2), simplified for clouds not intersecting other objects [S2, unofficial].

#### Volume Level-of-Detail (LOD) Rendering
*   **Definition:** A rendering process that maximizes image quality within user-specified frame rate and system constraints (RAM, texture memory, bandwidth) by adaptively modifying the set of rendered cells [S3].
*   **Steps:**
    1.  **Adapt/Load Phase:** Determine which octree cells to render. Load non-resident textures from RAM or disk to texture memory. Maintain an 'active cut' (nodes covering space exactly once) [S3].
    2.  **Active Cut Maintenance:** Split queue contains non-leaf nodes on active cut, sorted by decreasing error. To decrease error: Move first split node to merge queue (mark inactive), add its children to split queue (mark active). To increase error (frame rate): Remove first node from merge queue to split queue (mark active), add its parent to merge queue, remove children from split queue (mark inactive) [S3].
    3.  **Cull/Render Phase:** Traverse octree from root. Perform hierarchical cull. Render flagged nodes using 3D texture mapping and blending in back-to-front order [S3].
*   **Parameters:** User parameters: Desired frame rate, transfer function (data to RGBA), view position. System parameters: RAM, texture memory size, bandwidths, polygon and fill performance [S3].

## Background
The concept of Level-of-Detail (LOD) in volume rendering originated in the early 2000s with work on unstructured grids, where hierarchical voxelization and adaptive octrees were used to maintain interactivity [S3]. The specific technique of "Dynamic Resolution Adaptation" via frame-time feedback is a common pattern in real-time graphics engines, often cited in mixed reality and VR contexts to maintain high refresh rates (90 Hz+) on head-mounted displays [S1]. More recently, with the rise of Neural Radiance Fields (NeRF) and 3D Gaussian Splatting, dynamic resolution adaptation has been adapted for training-time optimization, where voxel grid resolution and alpha masks are updated periodically to manage memory and convergence [S1][S2, unofficial]. The "Focus and Context" approach has roots in interactive visualization systems aiming to provide orientation cues while focusing computational resources on specific data probes [S1].

## Sources
[S1] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] PDF Interactive Visualization of Unstructured Grids Using Hierarchical 3D ..., https://web.eecs.umich.edu/~jjcorso/pubs/leven02unstructgrid.pdf, fetched 2026-10-27
