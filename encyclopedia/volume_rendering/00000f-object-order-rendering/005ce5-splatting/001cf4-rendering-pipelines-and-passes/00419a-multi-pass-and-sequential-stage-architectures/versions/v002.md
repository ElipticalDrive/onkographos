# Multi-Pass and Sequential Stage Architectures


## Overview
Multi-Pass and Sequential Stage Architectures decompose volume rendering into distinct, ordered execution phases (e.g., data traversal, shading, compositing) to manage complexity, optimize memory access, or integrate with broader rendering pipelines. Unlike single-pass ray marching, these architectures separate geometric projection, attribute computation, and image synthesis into discrete steps, often leveraging intermediate framebuffers or data structures [S1, unofficial][S2, unofficial][S3, unofficial]. They are chosen when decoupling shading from geometry traversal, handling complex lighting/shadowing, or managing large datasets that exceed single-pass memory limits [S2, unofficial][S10584, unofficial].

## Shared principles
The family shares a sequential execution model where the output of one stage serves as the input for the next. Common mechanisms include:
1.  **Stage Decomposition:** The rendering process is split into logical units (e.g., Clear, Construction, Insertion, Evaluation) or functional passes (e.g., Depth, Position, Shading) [S1, unofficial][S10584, unofficial].
2.  **Intermediate Storage:** Data is persisted between stages using framebuffers, textures, or in-memory arrays (e.g., fragment lists, shading arrays) [S1, unofficial][S3, unofficial].
3.  **Sequential Dependency:** Stages must execute in a specific order to maintain data consistency (e.g., depth must be computed before shading; geometry must be inserted before evaluation) [S1, unofficial][S10584, unofficial].
4.  **Hardware Acceleration:** Most implementations utilize GPU pipelines, often via multiple draw calls or compute dispatches, to exploit parallelism within each stage [S1, unofficial][S10584, unofficial].

## Subtopics

### [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md)
A standard direct volume rendering structure that decomposes image formation into data traversal, interpolation, gradient computation, classification, shading, and compositing. It solves the problem of projecting 3D scalar fields to 2D images by treating samples as semi-transparent clouds, operating directly on volumetric data streams rather than explicit primitives. It is distinguished from fragment-list pipelines by its lack of explicit primitive construction and from deferred pipelines by its direct ray-casting compositing.

### [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md)
A four-stage object-order method (Clear, Construction, Insertion, Evaluation) for real-time smoke visualization. It circumvents raytracing artifacts by explicitly separating fragment storage (Insertion) from image synthesis (Evaluation) using per-pixel fragment lists. It is chosen when high accuracy in self-intersection handling is required for complex geometry.

### [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md)
A multi-pass sequence that computes depth, position/normal, materials, shadows, and SSAO in separate framebuffer objects. It separates geometric projection from shading logic, allowing independent optimization of appearance and traversal. It is chosen when complex lighting and post-processing effects are needed without re-traversing geometry.

### Eye/Light/Composite Pass Pipeline
A three-pass architecture for volumetric shadowing and lighting [S1, unofficial].
- **Definition:** Renders volumetric data in sequential eye, light, and composite passes to separate direct lighting from opacity accumulation [S1, unofficial].
- **Steps:**
    1.  **Eye Pass:** Renders the volume with direct lighting into `m_eyeBuffer` using `m_eyePipeline` and `m_eyeSet` [S1, unofficial].
    2.  **Light Pass:** Accumulates opacity into `m_lightBuffer` using `m_lightPipeline` and `m_volSet`, transitioning layout from `ColorAttachmentOptimal` to `ShaderReadOnlyOptimal` [S1, unofficial].
    3.  **Composite Pass:** Binds `m_compositePipeline` and `m_compSet`, drawing a full-screen quad using `m_eyeBuffer` as a sampled texture to blend results [S1, unofficial].
- **Layout:** Uses `m_eyeBuffer` (eye output), `m_lightBuffer` (opacity accumulation), `m_volumeTex` (3D volume, R8G8Unorm), and `m_tfTex` (transfer function) [S1, unofficial].

## Background
The concept of decomposing rendering into stages originates from early graphics pipelines and has evolved to address specific volume rendering challenges such as memory bandwidth, lighting complexity, and geometric accuracy. The Volume Rendering Pipeline formalizes the ray-casting process into distinct computational steps [S3, unofficial]. Fragment list-based approaches emerged to handle complex, non-grid-based volumes like smoke in real-time [S1, unofficial]. Deferred rendering techniques were adapted from surface rendering to volume visualization to support advanced lighting and post-processing effects [S10584, unofficial]. The Eye/Light/Composite pattern is a specialized variant for efficient volumetric shadowing [S1, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
