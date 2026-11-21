# Multi-Pass and Sequential Stage Architectures


## Overview
Multi-pass and sequential stage architectures decompose volume rendering into distinct, ordered computational phases to manage complexity, memory bandwidth, or specific visual effects that cannot be achieved in a single rasterization pass. Unlike single-pass splatting or ray-marching methods, these architectures explicitly separate data traversal, shading, compositing, or post-processing into separate GPU passes, often utilizing intermediate Framebuffer Objects (FBOs) or fragment lists [S1, unofficial][S2]. They are chosen when the rendering workload requires decoupling geometry processing from shading logic, managing complex fragment storage for accurate transparency, or integrating screen-space effects like ambient occlusion [S1, unofficial][S2].

## Shared principles
The family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [G].

1.  **Primitive Representation:** The volume is decomposed into primitives [G]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening .
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane [G]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
4.  **Compositing:** Contributions are accumulated to form the final image [G]. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, unofficial][S2]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) .
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, unofficial][S2].

## Subtopics

### [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md)
A standard implementation structure for direct volume rendering that processes scalar field data through a sequence of stages to produce a 2D image without fitting geometric primitives. It solves the problem of visualizing 3D volumetric data by discretizing the continuous volume rendering integral along light rays, mapping scalar values to optical properties, and compositing the results. What sets it apart from its siblings, such as the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) or the [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md), is its explicit separation of classification (opacity assignment) and shading (color assignment) into distinct, often parallel, steps before ray casting, allowing for independent control of surface appearance and transparency.

### [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md)
A multi-pass architecture for real-time smoke volume rendering that processes geometry through four distinct stages to store per-pixel fragment data in GPU memory structures. It is chosen to maintain high accuracy and circumvent the self-intersection and global intersection problems inherent in raytracing approaches. Unlike the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) which typically evaluates rays directly, or the [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md) which separates shading from geometry, this method explicitly decouples fragment construction and insertion from their final evaluation to manage complex storage requirements.

### [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md)
A multi-pass deferred rendering sequence that computes depth, position/normal, materials, filtered normals, shadows, SSAO, blurred AO, and final shading in separate framebuffer objects. It sets itself apart from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by decoupling geometry traversal from shading logic, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by relying on screen-space post-processing for effects like ambient occlusion rather than explicit fragment lists.

### Eye/Light/Composite Pass Pipeline
- **Definition:** Renders volumetric data in three sequential passes: eye pass (direct lighting/color), light pass (opacity accumulation for shadows), and composite pass (blends results) [S1, unofficial].
- **Steps:**
    1.  **Eye pass:** Render volume to `m_eyeBuffer` with direct lighting, using `m_eyePipeline` and `m_eyeSet` [S1, unofficial].
    2.  **Light pass:** Accumulate opacity into `m_lightBuffer` using `m_lightPipeline` and `m_volSet`, transitioning layout from `ColorAttachmentOptimal` to `ShaderReadOnlyOptimal` [S1, unofficial].
    3.  **Composite pass:** Bind `m_compositePipeline` and `m_compSet`, draw 3 triangles (full-screen quad) using `m_eyeBuffer` as a sampled texture [S1, unofficial].
- **Layout:** Buffers include `m_eyeBuffer` (eye pass output), `m_lightBuffer` (light pass opacity accumulation), `m_volumeTex` (3D volume data, R8G8Unorm), and `m_tfTex` (transfer function) [S1, unofficial].

### Jrender Rendering Pipeline
- **Definition:** A modular differentiable rendering pipeline containing four stages: vertex processing, rasterization, fragment shading, and post-processing [S2].
- **Steps:** Supports various rasterization algorithms and shading models through efficient interpolation between Jittor and CUDA [S2].
- **Usage:** Supports surface rendering for 3D meshes and volume rendering for 3D volumes. Supports PBR materials and shading effects such as soft shadows, ambient occlusions, subsurface scattering, and global illumination [S2].

## Background
The concept of multi-pass rendering in volume visualization evolved from the need to handle complex lighting and transparency effects that single-pass rasterization could not efficiently support. Early direct volume rendering methods, such as the Shading and Classification Pipeline, established the separation of opacity and color assignment to improve visual fidelity [S2]. As GPU architectures advanced, fragment list-based approaches emerged to handle complex smoke and fire simulations by decoupling fragment storage from evaluation, addressing issues like self-intersection in ray-tracing [S1, unofficial]. Deferred rendering techniques, originally developed for surface rendering, were adapted for volume rendering to integrate screen-space effects like ambient occlusion and shadows without the performance penalty of explicit ray casting [S1, unofficial]. The rise of differentiable rendering has further refined these pipelines, allowing for gradient computation through the entire multi-stage process for applications in computer vision and optimization [S2].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
