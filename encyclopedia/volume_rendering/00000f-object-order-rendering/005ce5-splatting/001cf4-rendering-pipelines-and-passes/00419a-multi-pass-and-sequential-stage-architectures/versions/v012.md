# Multi-Pass and Sequential Stage Architectures


## Overview
Multi-pass and sequential stage architectures decompose volume rendering into distinct, ordered computational phases to manage complexity, memory bandwidth, or differentiability. Unlike single-pass methods that attempt to traverse, shade, and composite in one kernel, these architectures decouple geometric traversal, data preparation, and optical compositing [S1, S2, S3, unofficial]. This separation allows for specialized optimization of each stage, such as deferred shading or fragment list management, at the cost of increased memory traffic between passes.

## Shared principles
The family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1, S3, S4, unofficial].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, S3, S4, unofficial]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6, unofficial][S10, abstract only, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6, unofficial][S10, abstract only, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2, S3, S4, unofficial]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2][S6, unofficial][S10, abstract only, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, abstract only, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, abstract only, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].

## Subtopics

### [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md)
A conceptual structure decomposing rendering into a fixed sequence of stages where the output of one serves as the input for the next. It solves the mapping of 3D scalar fields to 2D images by separating geometric traversal from optical property assignment. Unlike primitive-fitting methods, it directly shades samples using local gradients as normals and separate classification operators. It is distinct from [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by its explicit separation of shading/classification for undistorted visualization, and from [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md) by applying shading directly to volume samples rather than deferred geometry. Choose this for standard ray-casting with explicit gradient-based shading.

### [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md)
A multi-pass architecture for accurate real-time smoke rendering, circumventing self-intersection and global intersection artifacts of raytracing. It decouples geometric construction from shading using fragment lists to store per-pixel data, avoiding re-traversal. Unlike the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) which integrates shading into traversal, or the [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md) which focuses on opaque geometry, it explicitly manages fragment accumulation in intermediate buffers for dynamic properties like density and flow tangents. Choose this for dynamic smoke with complex per-pixel state management.

### [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md)
A multi-pass sequence decoupling geometric traversal/rasterization from final shading by storing intermediate state in Framebuffer Objects (FBOs). It efficiently applies complex lighting (shadows, AO) to large datasets by reusing screen-space data, avoiding redundant traversal. It differs from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by separating depth/position from shading, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by using screen-space buffers rather than explicit fragment lists. Choose this for complex lighting models on large volumes.

### Eye/Light/Composite Pass Pipeline
- **Definition:** Renders volumetric data in three sequential passes: eye pass (direct lighting/color), light pass (opacity accumulation for shadows), and composite pass (blends results) [S1, unofficial].
- **Steps:** Eye pass renders volume to `m_eyeBuffer` with direct lighting using `m_eyePipeline` and `m_eyeSet` [S1, unofficial]. Light pass accumulates opacity into `m_lightBuffer` using `m_lightPipeline` and `m_volSet`, transitioning layout from `ColorAttachmentOptimal` to `ShaderReadOnlyOptimal` [S1, unofficial]. Composite pass binds `m_compositePipeline` and `m_compSet`, drawing 3 triangles (full-screen quad) using `m_eyeBuffer` as a sampled texture [S1, unofficial].
- **Layout:** Buffers include `m_eyeBuffer` (eye pass output), `m_lightBuffer` (light pass opacity accumulation), `m_volumeTex` (3D volume data, R8G8Unorm), and `m_tfTex` (transfer function) [S1, unofficial].

### Jrender Rendering Pipeline
- **Definition:** A modular differentiable rendering pipeline containing four stages: vertex processing, rasterization, fragment shading, and post-processing [S2].
- **Steps:** Supports various rasterization algorithms and shading models through efficient interpolation between Jittor and CUDA [S2].
- **Usage:** Supports surface rendering for 3D meshes and volume rendering for 3D volumes [S2]. Supports PBR materials and shading effects such as soft shadows, ambient occlusions, subsurface scattering, and global illumination [S2].

## Background
The concept of decomposing rendering into stages originates from early volume visualization systems that required explicit handling of gradient computation and classification separate from ray casting [S1, S2, S3, unofficial]. The Volume Rendering Pipeline was formalized to provide a structured approach to mapping scalar fields to images [S1, S2, S3, unofficial]. Fragment list-based approaches emerged to address specific artifacts in smoke rendering [S1, unofficial]. Deferred rendering techniques, adapted from surface rendering, were applied to volumes to handle complex lighting efficiently [S1, unofficial]. Modern differentiable rendering libraries like Jrender integrate these stages into unified computational graphs for gradient-based optimization [S2].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
