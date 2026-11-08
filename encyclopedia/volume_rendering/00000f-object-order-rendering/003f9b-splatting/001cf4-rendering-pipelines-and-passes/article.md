# Rendering Pipelines and Passes

Also called: Render Forward Pass

## Overview
Rendering Pipelines and Passes define the structural organization of volume rendering tasks into distinct, sequential logical blocks or stages, separating geometric projection from visual appearance to allow independent optimization of shading, classification, and compositing [S3, unofficial]. This family solves the problem of managing complex data transformations by decomposing the rendering task into reusable passes (e.g., attribute accumulation, depth occlusion, normalization) that can be combined in various configurations, distinguishing it from monolithic ray-marching or single-shader approaches [S3, unofficial]. It sets itself apart from sibling methods like [Splatting Without the Blur](1157-splatting-without-the-blur.md) by focusing on the orchestration of rendering stages rather than the specific primitive rasterization or filtering techniques.

## Shared principles
The rendering pipeline family operates on the principle of stage separation, where each stage performs a specific transformation on the data or image state. Common mechanisms include:

1.  **Stage Decomposition:** The rendering process is divided into logical stages such as Data Traversal, Interpolation, Gradient Computation, Classification, Shading, and Compositing [S1, S3, unofficial]. This allows for independent optimization of visual appearance (shading/classification) and geometric projection (compositing) [S3, unofficial].
2.  **Intermediate Storage:** Pipelines utilize intermediate framebuffers or data structures (e.g., fragment lists, G-buffers) to store per-pixel or per-voxel attributes (color, normal, depth, opacity) between stages [S1, S2, unofficial]. For example, [Custom Pass Rendering](3629-custom-pass-rendering.md) uses multiple FBOs to store hit positions, normals, and material IDs [S1, unofficial].
3.  **Sequential Execution:** Stages are executed in a defined order. For instance, [Fragment List-Based Rendering](3488-fragment-list-based-rendering-pipeline.md) follows a Clear, Construction, Insertion, and Evaluation sequence [S1, unofficial].
4.  **Accumulation and Blending:** Final image generation involves accumulating contributions from multiple samples or primitives, often using alpha blending or weighted summation [S1, S3, unofficial].

## Subtopics

### [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md)
This method provides a modular interface for efficient ray marching and differentiable volumetric rendering, allowing users to plug any user-defined radiance field into a standardized acceleration pipeline. It distinguishes itself from specialized implementations like Instant-NGP by offering a Python API built on PyTorch, which facilitates easier extension to dynamic and unbounded scenes while maintaining high performance. Choose this when differentiable training of arbitrary radiance fields is required.

### [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md)
The Volume Rendering Pipeline is a standard implementation structure for direct volume rendering that decomposes the image formation process into logical stages: data traversal, interpolation, gradient computation, classification, shading, and compositing. It solves the problem of mapping a 3D scalar field to a 2D image by treating each sample point along a viewing ray as a semi-transparent, reflective, and light-emitting entity. Unlike methods that fit geometric primitives to the data, this pipeline projects shaded samples directly onto the picture plane, allowing independent optimization of visual appearance (shading/classification) and geometric projection (compositing). It is distinguished from [Post-classification Shading Pipeline](7545-post-classification-shading-pipeline.md) by typically performing classification on pre-classified voxel values or resampled arrays, whereas the post-classification variant classifies interpolated samples directly.

### [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md)
The Time-Sliced Pipelined Architecture is a hardware design methodology for volume rendering that achieves real-time performance by time-slicing functional blocks to allow the simultaneous processing of multiple tasks. It solves the latency and throughput bottlenecks of sequential rendering by overlapping execution stages, enabling continuous communication between hardware components and software control. It sets itself apart from standard pipelined approaches by explicitly reusing embedded sub-cores across different pipeline stages via a shared context memory, rather than dedicating fixed hardware to a single stage.

### [Screen Quad Rendering](2435-screen-quad-rendering.md)
Screen Quad Rendering is the final presentation stage in a volume rendering pipeline, responsible for transferring computed volume data from off-screen render targets to the default framebuffer or current context texture for display. It differs from standard blits by explicitly managing multiple draw buffers and allowing fragment shaders to sample intermediate textures for final compositing or debugging overlays.

### [Rendering Pipeline Architecture](3059-rendering-pipeline-architecture.md)
Rendering Pipeline Architecture refers to the structural organization of volume rendering stages into distinct, sequential logical blocks that process data or image states. It solves the problem of managing complex data transformations by separating geometric projection from visual appearance, allowing for independent optimization of shading, classification, and compositing. It differs from monolithic ray-marching approaches by explicitly decomposing the rendering task into reusable passes, such as attribute accumulation, depth occlusion, and normalization, which can be combined in various configurations like the [Splat Attributes Pass](1066-splat-attributes-pass.md) or the [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-pipeline.md).

### [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md)
The Fragment List-Based Rendering Pipeline is a four-stage object-order rendering method designed for real-time smoke volume visualization. It processes geometry through Clear, Construction, Insertion, and Evaluation stages to maintain high accuracy while circumventing the self-intersection and global intersection artifacts common in raytracing approaches. It is distinguished from sibling pipelines by its explicit separation of fragment storage (Insertion) from final image synthesis (Evaluation), using fragment lists to accumulate per-pixel data before compositing.

### [Custom Pass Rendering](3629-custom-pass-rendering.md)
Custom Pass Rendering is a multi-pass, object-order rendering pipeline designed for high-performance visualization of large-scale volumetric data (specifically NanoVDB) and octree-based geometry within a modern graphics API context. It solves the problem of efficiently computing surface attributes (position, normal, material ID) and lighting effects (SSAO, shadows) for sparse volumetric datasets by decoupling geometric traversal from shading. Unlike monolithic ray-marching shaders, this method uses a sequence of specialized FullScreenPass stages to build intermediate G-buffers (hit position, filtered normals, ambient occlusion) before a final shading pass. It is distinct from [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md) by its explicit separation of traversal and lighting into discrete render passes, and from [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by its reliance on screen-space fragment buffers rather than explicit primitive lists.

### [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md)
DirectXSplat is a C++/Direct3D 12 library for rendering 3D Gaussian Splats, providing a rasterization pipeline that handles scene upload, preparation, rendering, and frame submission with explicit resource management. It differs from general-purpose ray marching or CPU-based splitters by offering a host-owned D3D12 architecture where the renderer manages GPU memory residency and LOD culling while the host controls command list recording and queue submission. It is designed for high performance, claiming up to 2.4x faster rendering than `gsplat` in matched-quality benchmarks on NVIDIA hardware.

### [Ray-Centric and Hardware-Accelerated Rendering](8936-ray-centric-and-hardware-accelerated-rendering.md)
Ray-centric rendering computes the color of each pixel by tracing a ray from the camera through the volume and accumulating contributions from sampled points along the ray path. It differs from object-order splatting by avoiding the sorting and blending overheads required for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G]. It preserves internal volume information, distinguishing it from isosurface extraction methods like [Marching Cubes](53-marching-cubes.md).

#### Eye/Light/Composite Pass Pipeline
*   **Definition:** Renders volumetric data in three sequential passes: eye pass (direct lighting/color), light pass (opacity accumulation for shadows), and composite pass (blends results) [S2, unofficial].
*   **Steps:**
    1.  **Eye pass:** Render volume to `m_eyeBuffer` with direct lighting, using `m_eyePipeline` and `m_eyeSet` [S2, unofficial].
    2.  **Light pass:** Accumulate opacity into `m_lightBuffer` using `m_lightPipeline` and `m_volSet`, transitioning layout from `ColorAttachmentOptimal` to `ShaderReadOnlyOptimal` [S2, unofficial].
    3.  **Composite pass:** Bind `m_compositePipeline` and `m_compSet`, draw 3 triangles (full-screen quad) using `m_eyeBuffer` as a sampled texture [S2, unofficial].
*   **Layout:** Buffers include `m_eyeBuffer` (eye pass output), `m_lightBuffer` (light pass opacity accumulation), `m_volumeTex` (3D volume data, R8G8Unorm), and `m_tfTex` (transfer function) [S2, unofficial].

## Background
The concept of decomposing rendering into stages originates from early direct volume rendering implementations that sought to separate the visualization of scalar fields from the geometric projection [S3, unofficial]. The specific "Render Forward Pass" or "msplat_render" terminology is associated with modern 3D Gaussian Splatting implementations, such as the `msplat` project, which aims for efficient training and rendering on native hardware (e.g., Metal) [S1, unofficial]. The family encompasses diverse approaches, from hardware-level time-slicing [S1, unofficial] to high-level API abstractions like DirectXSplat [S1, unofficial] and Python-based differentiable pipelines [S1, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
