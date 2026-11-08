# Multi-Pass and Sequential Stage Architectures


## Overview
Multi-Pass and Sequential Stage Architectures decompose volume rendering into discrete, ordered processing steps to manage complexity and optimize resource usage. Unlike monolithic ray-marching kernels, this family separates geometric traversal, optical property assignment, and image formation into distinct passes or stages [S1, S3, unofficial]. This separation allows for independent optimization of visual appearance (shading/classification) and geometric projection (compositing), enabling techniques such as deferred shading, fragment-list accumulation, and hybrid light/eye passes that are difficult to achieve in single-pass implementations [S3, unofficial].

## Shared principles
The rendering pipeline family operates on the principle of stage separation, where each stage performs a specific transformation on the data or image state. Common mechanisms include:

1.  **Stage Decomposition:** The rendering process is divided into logical stages such as Data Traversal, Interpolation, Gradient Computation, Classification, Shading, and Compositing [S1, S3, unofficial]. This allows for independent optimization of visual appearance (shading/classification) and geometric projection (compositing) [S3, unofficial].
2.  **Intermediate Storage:** Pipelines utilize intermediate framebuffers or data structures (e.g., fragment lists, G-buffers) to store per-pixel or per-voxel attributes (color, normal, depth, opacity) between stages [S1, S2, unofficial]. For example, [Custom Pass Rendering](3629-custom-pass-rendering.md) uses multiple FBOs to store hit positions, normals, and material IDs [S1, unofficial].
3.  **Sequential Execution:** Stages are executed in a defined order. For instance, [Fragment List-Based Rendering](3488-fragment-list-based-rendering-pipeline.md) follows a Clear, Construction, Insertion, and Evaluation sequence [S1, unofficial].
4.  **Accumulation and Blending:** Final image generation involves accumulating contributions from multiple samples or primitives, often using alpha blending or weighted summation [S1, S3, unofficial].

## Subtopics

### [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md)
A standard six-stage architectural model for direct volume rendering that structures the transformation of raw scalar field data into a final 2D image through discrete processing steps. It is distinct from fragment-list-based or deferred pipelines by performing shading and classification on a per-sample basis during ray traversal, rather than deferring these calculations to fragment processing or separate buffer passes.

### [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md)
A multi-pass architecture designed for real-time smoke volume rendering that processes geometry through four distinct stages to store and evaluate per-pixel fragment data. It is chosen specifically to maintain high accuracy and circumvent the self-intersection and global intersection problems inherent in raytracing approaches. Unlike the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md), which typically processes voxels or rays directly, this method rasterizes primitives into fragment lists for deferred evaluation.

### [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md)
A multi-pass deferred rendering sequence that computes depth, position/normal, materials, filtered normals, shadows, SSAO, blurred AO, and final shading in separate framebuffer objects. It differs from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by decoupling geometry traversal from shading, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by using screen-space quad passes for post-processing effects rather than fragment lists.

### Eye/Light/Composite Pass Pipeline
*   **Definition:** Renders volumetric data in three sequential passes: eye pass (direct lighting/color), light pass (opacity accumulation for shadows), and composite pass (blends results) [S1, unofficial].
*   **Steps:**
    1.  **Eye Pass:** Render volume to `m_eyeBuffer` with direct lighting, using `m_eyePipeline` and `m_eyeSet` [S1, unofficial].
    2.  **Light Pass:** Accumulate opacity into `m_lightBuffer` using `m_lightPipeline` and `m_volSet`, transitioning layout from `ColorAttachmentOptimal` to `ShaderReadOnlyOptimal` [S1, unofficial].
    3.  **Composite Pass:** Bind `m_compositePipeline` and `m_compSet`, draw 3 triangles (full-screen quad) using `m_eyeBuffer` as a sampled texture [S1, unofficial].
*   **Layout:** Buffers include `m_eyeBuffer` (eye pass output), `m_lightBuffer` (light pass opacity accumulation), `m_volumeTex` (3D volume data, R8G8Unorm), and `m_tfTex` (transfer function) [S1, unofficial].

## Background
The concept of decomposing rendering into stages originates from the canonical volume rendering pipeline defined in foundational literature, which established the six-stage model of Data Traversal, Interpolation, Gradient Computation, Classification, Shading, and Compositing [S1, S3, unofficial]. This model solved the problem of efficiently mapping 3D volumetric data to a 2D screen by separating geometric traversal from optical property assignment [S1, S3, unofficial]. Subsequent developments, such as the Fragment List-Based Rendering Pipeline, adapted these principles to real-time graphics APIs (e.g., OpenGL) to handle complex geometries like smoke without ray-tracing artifacts [S1, unofficial]. The Deferred Rendering Pipeline further extended this by applying screen-space techniques to volumetric and hybrid scenes, allowing for complex lighting and post-processing effects [S1, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
