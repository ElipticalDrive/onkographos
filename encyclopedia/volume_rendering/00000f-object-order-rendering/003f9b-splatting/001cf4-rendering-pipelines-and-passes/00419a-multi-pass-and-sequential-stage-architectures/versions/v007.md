# Multi-Pass and Sequential Stage Architectures

Also called: Multi-Pass Rendering, Sequential Stage Rendering

## Overview
Multi-Pass and Sequential Stage Architectures decompose the volume rendering process into a sequence of distinct rendering passes or logical stages, where the output of one stage serves as the input for the next. This approach solves the problem of managing complex lighting, shadowing, and compositing requirements that cannot be handled in a single rasterization or ray-marching pass. It is distinct from single-pass splatting methods by explicitly separating data traversal, shading, and compositing into independent GPU operations, allowing for the use of intermediate framebuffers (FBOs) to store per-pixel attributes like depth, normals, and opacity before final image synthesis.

## Shared principles
The family shares the fundamental mechanism of treating volume rendering as a pipeline of sequential operations, often leveraging the GPU's render pipeline or compute shaders to process data in stages.

1.  **Stage Decomposition:** The rendering process is broken down into logical or physical passes. Common stages include data preparation, geometry generation, shading/lighting, and final compositing [S1, S3, unofficial].
2.  **Intermediate Storage:** Results from one stage are stored in intermediate buffers (e.g., Framebuffer Objects, Texture Buffers, or Atomic Counters) to be read by subsequent stages [S1, 10584-deferred-rendering-pipeline, unofficial]. This decouples geometry processing from shading [10584-deferred-rendering-pipeline].
3.  **Sequential Execution:** Stages are executed in a defined order. For example, a "Clear" stage may reset buffers, followed by a "Construction" stage to generate primitives, an "Insertion" stage to populate fragment lists, and an "Evaluation" stage to composite the final image [S1, unofficial].
4.  **Resource Management:** The architecture requires careful management of GPU resources, including binding textures, buffers, and shader programs between passes [S1, 10584-deferred-rendering-pipeline, unofficial].

## Subtopics

### [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md)
A conceptual structure that decomposes image formation into data traversal, interpolation, gradient computation, classification, shading, and compositing. It is chosen when a clear mathematical decomposition of the volume rendering integral is required, mapping scalar fields to optical properties. It differs from siblings by focusing on per-sample optical property assignment and back-to-front accumulation.

### [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md)
A multi-pass architecture using fragment lists to store per-pixel data, designed to circumvent self-intersection artifacts in raytracing. It is chosen for accurate real-time smoke rendering. It differs from siblings by using explicit fragment lists and four distinct stages (Clear, Construction, Insertion, Evaluation).

### [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md)
A multi-pass sequence that decouples geometry processing from shading by storing per-pixel attributes in FBOs. It is chosen for efficiently applying complex lighting and post-processing effects (SSAO, shadows). It differs from siblings by relying on screen-space quad passes for post-processing rather than explicit fragment lists.

### Eye/Light/Composite Pass Pipeline
A three-pass pipeline rendering volumetric data in sequential eye (direct lighting), light (opacity accumulation), and composite passes. It is chosen when shadow casting and direct lighting need to be separated for performance or flexibility. It uses specific buffers (`m_eyeBuffer`, `m_lightBuffer`) and transitions layout states between passes [S1, unofficial].

### Jrender Rendering Pipeline
A modular differentiable rendering pipeline with four stages: vertex processing, rasterization, fragment shading, and post-processing. It is chosen for differentiable rendering tasks supporting PBR materials and various shading effects. It leverages efficient interpolation between Jittor and CUDA [S2].

## Background
The concept of decomposing rendering into multiple passes originated with the need to handle complex lighting and transparency effects that single-pass rasterization could not efficiently support. The **Volume Rendering Pipeline** was formalized by Levoy [S3, unofficial] to provide a canonical structure for direct volume rendering, separating the mathematical steps of the rendering integral. The **Fragment List-Based Rendering Pipeline** was developed to address artifacts in real-time smoke rendering, specifically avoiding self-intersection issues inherent in raytracing [S1, unofficial]. The **Deferred Rendering Pipeline** approach in volume rendering adapts techniques from surface rendering to handle complex lighting and post-processing effects like SSAO and shadows by decoupling geometry from shading [10584-deferred-rendering-pipeline]. Modern implementations like **Jrender** extend these concepts to differentiable rendering, allowing gradients to flow through the pipeline for optimization tasks [S2].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
