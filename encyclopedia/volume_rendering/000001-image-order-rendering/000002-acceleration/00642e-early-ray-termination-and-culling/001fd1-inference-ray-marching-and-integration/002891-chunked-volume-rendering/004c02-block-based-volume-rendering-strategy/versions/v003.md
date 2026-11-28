# Block-based Volume Rendering Strategy

Also called: Multi-shader Block Accumulation, Sequential Block Rendering

## Overview
A rendering strategy for large scenes partitioned into blocks, where each block uses a separate shader due to web texture unit limits [S1]. It solves the problem where standard volume rendering fails because sampling points across a ray may belong to different blocks loaded by different shaders, breaking 3D consistency and correct occlusion [S1]. It differs from [Chunked Volume Rendering with Occupancy Grid Estimator](4652-chunked-volume-rendering-with-occupancy-grid-estimator.md) by using sequential block rendering with separate shaders rather than occupancy grids to skip empty space, and from [Partitioned Volume Rendering Derivation](10384-partitioned-volume-rendering-derivation.md) by enforcing strict back-to-front sorting and post-hoc accumulation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $M$ scene blocks, each associated with a dedicated shader instance containing its specific texture resources [S1].
*   **Output:** The final composite color $C_r$ for the pixel [S1].

**Data Structures:**
*   **Block List:** A list of $M$ blocks, each containing a reference to its shader and its spatial position [S1].
*   **Shader State:** Each shader instance holds the texture units required to render its specific block [S1].

**Steps:**
1.  **Depth Sorting:** Sort all $M$ scene blocks relative to the camera viewpoint to determine the rendering order from back to front [S1].
2.  **Sequential Rendering:** Iterate through the sorted blocks. For each block $k$:
    *   Activate the dedicated shader for block $k$ [S1].
    *   Render the block to obtain its local color $c_k$ and opacity $\alpha_k$ [S1].
    *   Store $c_k$ and $\alpha_k$ for accumulation [S1].
3.  **Accumulation:** Combine the results using the volume rendering equation [S1]:

$$
C_r = \sum_{k=1}^{M} \left( \prod_{j=1}^{k-1} (1 - \alpha_j) \right) c_k
$$

Where:
*   $C_r$ is the final composite color [S1].
*   $M$ is the total number of blocks [S1].
*   $c_k$ is the color output by the shader for block $k$ [S1].
*   $\alpha_k$ is the opacity output by the shader for block $k$ [S1].
*   The term $\prod_{j=1}^{k-1} (1 - \alpha_j)$ represents the transmittance from all preceding blocks $j < k$ [S1].

**Parameters:**
*   **Texture Unit Limit:** Typically no more than 16 texture units can be loaded into a single web shader [S1]. This constraint necessitates the separation of blocks into different shaders [S1].

**Edge Cases and Pitfalls:**
*   **Web Platform Limits:** Web platforms limit texture units to ~16, preventing loading all block resources into a single shader [S1].
*   **3D Consistency:** Using different shaders per block breaks standard volume rendering because sampling points across a ray may belong to different blocks/shaders [S1]. This method ensures correctness by rendering sequentially and combining results [S1].
*   **Memory Usage:** Loading all blocks for rendering is impractical due to excessive memory usage on standard consumer devices [S1].

## Subtopics

### [Training Pipeline Consistency](13141-training-pipeline-consistency.md)
Training Pipeline Consistency is a strategy to minimize the discrepancy between the training and inference phases of block-based volume rendering by enforcing the identical rendering pipeline during both stages. It solves the problem where training with a continuous, global coordinate system leads to artifacts or quality degradation when the model is deployed in a constrained, block-based rendering environment. It sets itself apart from standard neural rendering methods by explicitly simulating the discrete, hardware-bound constraints of the target rendering hardware (such as texture unit limits and quantization) during the optimization process.

### [Patch-based Volume Rendering](13945-patch-based-volume-rendering.md)
A memory-saving rendering strategy for high-resolution volumes that splits the spatial dimensions into overlapping patches, renders them independently, and merges the results to scale to arbitrary image resolutions. Unlike full-volume rendering which requires memory proportional to $H \times W$, the patch-based approach trades redundant computation in overlapping regions for drastically reduced memory footprint, enabling scaling to arbitrary resolutions. Builds on [Block-based Volume Rendering Strategy](13140-block-based-volume-rendering-strategy.md) by partitioning the spatial domain into smaller, independently processed regions.

### Blocking Raycaster with Block Sorting
A rendering system that divides the volume into blocks, sorts them by distance from the camera, and renders only non-empty blocks using raycasting shaders [S2, unofficial]. Blocks are sorted by distance from the camera eye in descending order (far to near) [S2, unofficial]. For each non-empty block with GPU resources, the world matrix is set, the block texture is bound, uniforms are configured, and a cube is drawn [S2, unofficial]. Uniforms include `top = b.worldDims()`, `bot = -top`, `mv = getViewMatrix() * getWorldMatrix()`, and `normalMatrix = transpose(inverse(mv))` [S2, unofficial]. Sampler state is configured with `GL_TEXTURE_MIN_FILTER = GL_LINEAR`, `GL_TEXTURE_MAG_FILTER = GL_LINEAR`, and `GL_TEXTURE_WRAP_S/T/R = GL_CLAMP_TO_BORDER` [S2, unofficial]. The rendered geometry uses 8 vertices at the corners of a unit cube centered at origin: `{-0.5,-0.5,0.5}, {0.5,-0.5,0.5}, {0.5,0.5,0.5}, {-0.5,0.5,0.5}, {-0.5,-0.5,-0.5}, {0.5,-0.5,-0.5}, {0.5,0.5,-0.5}, {-0.5,0.5,-0.5}` [S2, unofficial]. These are indexed by 36 indices forming 12 triangles (2 per face) for the front, right, back, left, top, and bottom faces [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of blocks $M$ for the sorting and accumulation steps, plus the cost of rendering each block [S1].
*   **Memory Complexity:** Proportional to the number of active blocks loaded into shaders, constrained by the texture unit limit (typically 16) [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Supports dynamic loading of resources via Level-of-Detail (LOD) generation to handle large scenes on consumer devices [S1].
*   **Strengths:** Enables rendering of large-scale scenes on web platforms with limited texture unit resources [S1]. Ensures correct occlusion and 3D consistency [S1].
*   **Limitations:** Requires sequential rendering, which may be slower than parallel approaches if not optimized [S1]. Limited by the number of texture units available per shader [S1].
*   **When to use:** When rendering large scenes on web platforms with strict texture unit limits [S1].
*   **When not:** When the target hardware supports loading all block resources into a single shader, eliminating the need for sequential accumulation [S1].

## Background
Originated in the "City-on-Web" paper [S1]. The method addresses challenges associated with resource-independent block-based rendering on the web, specifically the limitation on the number and resolution of texture units that can be loaded into a shader [S1]. It draws inspiration from traditional graphics techniques for Level-of-Detail (LOD) generation [S1].

## Sources
[S1] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
[S2] jimpelton/subvol, https://github.com/jimpelton/subvol, fetched 2026-10-26 (unofficial)
