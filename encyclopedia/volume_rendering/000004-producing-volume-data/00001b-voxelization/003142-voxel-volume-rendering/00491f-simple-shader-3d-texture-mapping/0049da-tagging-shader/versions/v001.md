# Tagging Shader

Also called: vzTMTagShader

## Overview
The Tagging Shader performs volumetric tagging by using a second volume as a 3D stencil buffer to selectively render data from a primary volume. It solves the problem of masking arbitrary regions within a dataset without modifying the original source data. Unlike methods that alter the volume texture by setting opacity to zero, this approach keeps the original data intact at the cost of storing an additional 3D texture in memory [S1].

## How it works
**Inputs and Outputs:**
*   **Input 1:** `volume` (`vzParameterVolumeTexture`): The primary 3D texture containing the actual volumetric data.
*   **Input 2:** `tag` (`vzParameterVolumeTexture`): The secondary 3D texture acting as the stencil/mask. It must be perfectly overlapping with the `volume` texture.
*   **Input 3:** `lookup_table` (`vzParameterLookupTable`): The transfer function used for color and opacity mapping.
*   **Output:** The rendered image where voxels are either visible or masked out based on the tag volume.

**Data Structures:**
The method relies on two 3D textures of identical dimensions.
*   `volume`: Standard 3D texture format.
*   `tag`: Ideally requires 1 bit per texel, but hardware constraints typically enforce at least 1 byte per texel [S1].

**Algorithm Steps:**
1.  For each fragment corresponding to a voxel position $(x, y, z)$, sample the value $V_{tag}$ from the `tag` texture.
2.  Compare $V_{tag}$ against the threshold $0.5$.
3.  If $V_{tag} > 0.5$:
    *   Sample the value $V_{vol}$ from the `volume` texture.
    *   Apply the `lookup_table` to $V_{vol}$ to determine color and opacity.
    *   Render the fragment.
4.  If $V_{tag} \le 0.5$:
    *   Mask out the fragment (discard or render with zero opacity).

**Implementation Details:**
The implementation is system-dependent [S1].
*   **GPU Support:** On systems supporting `ARB_fragment_program` or `ATI_fragment_shader` OpenGL extensions, the tagging is performed in the fragment shader using the logic above [S1].
*   **Legacy Support:** On systems lacking these extensions, the shader uses the OpenGL stencil buffer to implement the tagging algorithm. The application must ensure the appropriate visual configuration is selected to support stencil operations [S1].

**Pitfalls:**
*   **Memory Overhead:** The `tag` volume consumes additional texture memory. Since most graphics hardware uses at least one byte per texel for the tag volume, the memory cost is significant compared to the theoretical 1-bit requirement [S1].
*   **Stencil Buffer Requirements:** When falling back to stencil buffer implementation, the rendering context must be configured with a stencil buffer. Failure to do so will result in incorrect rendering or errors [S1].

## Subtopics

### [Mask-based Voxel Filtering](13669-mask-based-voxel-filtering.md)
A fragment shader that combines the original volume dataset with a texture mask (SculptMap) to discard voxels marked for removal during visualization.

## Operating characteristics
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, due to the storage of an additional 3D texture for the tag volume [S1].
*   **Performance:** Rendering performance is comparable to standard volume rendering, with the added cost of sampling the tag texture. The stencil buffer fallback may incur additional overhead depending on the hardware and driver implementation [S1].
*   **Strengths:** Preserves the integrity of the original volume data; allows dynamic masking without pre-processing or data modification [S1].
*   **Limitations:** Increased memory usage; dependency on specific OpenGL extensions or stencil buffer support for correct operation on all hardware [S1].
*   **When to use:** When the user needs to visualize subsets of a volume dataset without permanently altering the source data or when dynamic masking is required.
*   **When not to use:** When memory is constrained and the tag volume cannot be afforded, or when the original data can be pre-processed to remove unwanted regions.

## Background
The Tagging Shader (`vzTMTagShader`) is part of the SGI OpenGL Volumizer library, specifically available in versions prior to OpenGL Volumizer 2.1 [S1]. It is designed to work with the `TMRenderAction` and utilizes the standard shader parameter interface for volume textures and lookup tables [S1]. The method addresses the need for flexible visualization tools in scientific and medical imaging where users often need to isolate specific anatomical or structural regions without losing access to the full dataset.

## Sources
[S1] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
