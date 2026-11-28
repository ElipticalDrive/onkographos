# Multi-Pass and Sequential Stage Architectures

Also called: Multi-Pass Rendering, Sequential Stage Pipelines

## Overview
Multi-pass and sequential stage architectures decompose volume rendering into distinct, ordered computational phases to manage complexity, memory bandwidth, or specific optical effects that cannot be handled in a single rasterization pass. Unlike single-pass splatting or ray-marching, these architectures decouple geometric traversal, attribute computation, and final compositing into separate rendering passes [S1, S3, 1318-volume-rendering-pipeline, 3488-fragment-list-based-rendering-pipeline, 10584-deferred-rendering-pipeline]. They are chosen when the rendering workload requires intermediate data storage (e.g., fragment lists, G-buffers) or when lighting and opacity accumulation must be separated to support effects like volumetric shadows or deferred shading [S1, 1318-volume-rendering-pipeline, 3488-fragment-list-based-rendering-pipeline, 10584-deferred-rendering-pipeline, unofficial].

## Shared principles
The family shares the mechanism of executing a sequence of rendering commands (draw calls or compute dispatches) where the output of one stage serves as the input for the next. This requires explicit management of intermediate resources, typically Framebuffer Objects (FBOs) or buffer objects, to store per-pixel or per-fragment data between passes [S1, 3488-fragment-list-based-rendering-pipeline, 10584-deferred-rendering-pipeline, unofficial].

1.  **Stage Decomposition:** The rendering process is split into logical stages (e.g., Clear, Construction, Insertion, Evaluation; or Depth, Position, Shading) [S1, 3488-fragment-list-based-rendering-pipeline, 10584-deferred-rendering-pipeline, unofficial]. Each stage performs a specific transformation or accumulation.
2.  **Intermediate Storage:** Data generated in one pass (e.g., depth, normals, fragment lists, opacity accumulators) is written to GPU memory (textures or buffers) and bound as input for subsequent passes [S1, 3488-fragment-list-based-rendering-pipeline, 10584-deferred-rendering-pipeline, unofficial].
3.  **Sequential Execution:** Stages are executed in a strict order defined by data dependencies. For example, geometry must be rasterized before fragments can be evaluated, or depth must be computed before world-space reconstruction [S1, 3488-fragment-list-based-rendering-pipeline, 10584-deferred-rendering-pipeline, unofficial].
4.  **Resource Management:** The implementation must handle buffer clearing, state binding (shaders, textures, samplers), and synchronization between passes to ensure data consistency [S1, 3488-fragment-list-based-rendering-pipeline, 10584-deferred-rendering-pipeline, unofficial].

## Subtopics

### [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md)
A sequential pipeline that processes volumetric data by traversing rays, sampling the volume, and compositing results. It is chosen for direct volume rendering where shading and classification are computed per-voxel before ray casting. It differs from splatting by treating samples as semi-transparent clouds directly shaded and projected, and from fragment-list methods by not using dynamic per-pixel data structures [1318-volume-rendering-pipeline].

### [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md)
A multi-pass architecture for real-time smoke rendering that uses fragment lists to store per-pixel data. It is chosen to maintain high accuracy and circumvent self-intersection problems in raytracing. It differs from the Volume Rendering Pipeline by decoupling geometric construction and fragment storage from final pixel evaluation, allowing complex per-pixel data structures [3488-fragment-list-based-rendering-pipeline].

### [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md)
A multi-pass sequence that decouples geometry processing from shading by storing scene attributes (depth, position, normal, material) in separate FBOs before computing lighting and compositing. It differs from the Volume Rendering Pipeline by rasterizing explicit geometric primitives (like an octree) into screen-space buffers rather than integrating density along rays, and from the Fragment List-Based Pipeline by using fixed, sequential passes over full-screen quads rather than dynamic fragment lists [10584-deferred-rendering-pipeline].

### Eye/Light/Composite Pass Pipeline
A three-pass architecture for rendering volumetric data with shadow support. It separates direct lighting, opacity accumulation for shadows, and final compositing. It is chosen when volumetric shadows are required without full ray-traced shadowing. The pipeline consists of:
1.  **Eye Pass:** Renders the volume with direct lighting into `m_eyeBuffer` using `m_eyePipeline` and `m_eyeSet` [S1, unofficial].
2.  **Light Pass:** Accumulates opacity into `m_lightBuffer` using `m_lightPipeline` and `m_volSet`. This pass transitions the buffer layout from `ColorAttachmentOptimal` to `ShaderReadOnlyOptimal` to prepare for sampling [S1, unofficial].
3.  **Composite Pass:** Binds `m_compositePipeline` and `m_compSet`, and draws a full-screen quad (3 triangles) using `m_eyeBuffer` as a sampled texture to blend the direct lighting with shadow information derived from `m_lightBuffer` [S1, unofficial].
The data structures include `m_eyeBuffer` (eye pass output), `m_lightBuffer` (light pass opacity accumulation), `m_volumeTex` (3D volume data, R8G8Unorm), and `m_tfTex` (transfer function) [S1, unofficial].

## Background
The concept of decomposing rendering into multiple passes originates from the need to handle complex lighting and transparency effects that single-pass rasterization cannot efficiently support. The Volume Rendering Pipeline formalizes the steps of data preparation, shading, classification, and compositing for direct volume rendering [1318-volume-rendering-pipeline]. The Fragment List-Based approach was developed to address the limitations of raytracing in real-time smoke rendering by leveraging GPU fragment shaders for dynamic data structures [3488-fragment-list-based-rendering-pipeline]. Deferred rendering techniques, originally developed for surface rendering, were adapted for volume rendering to allow flexible lighting and shadow calculations on volumetric primitives represented as geometry (e.g., octrees) [10584-deferred-rendering-pipeline]. The Eye/Light/Composite pattern is a specific implementation for volumetric shadows, separating the expensive opacity accumulation from the final shading pass [S1, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
