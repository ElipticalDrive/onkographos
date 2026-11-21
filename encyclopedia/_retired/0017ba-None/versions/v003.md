# Reflected Light Component Lookup

Also called: Gradient-based shading lookup, Lookup Table Shading, Lookup Table for Shading, Material-Gradient Lookup Table, Shading via lookup tables

## Overview
Reflected Light Component Lookup is a shading method that pre-calculates surface responses to directional lights for combinations of quantized gradient directions and material types, storing the results in a lookup table [S1, S2, S4, unofficial]. It solves the problem of efficiently applying complex lighting (diffuse and specular) to volume-rendered voxels classified as reflective surfaces, departing from methods that use gradient magnitude in isolation by interpreting the gradient in the context of data value and classification parameters [S1, S2, S4, unofficial]. Unlike sibling methods that avoid recalculation after rotation by lacking directional lighting, this method supports directional lights but requires table recalculation when lighting factors change [S2, S4, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Number of materials $m$, number of gradient refinement levels $k$, voxel field data, classification parameters, lighting properties (color, direction), and shading parameters (specularity model).
*   **Outputs:** A lookup table of color intensities and the final rendered voxel colors.

**Data Structures**
1.  **Lookup Table:** An array of size $(m + 1) \times g$ entries [S1, S2, S4, unofficial].
    *   $m$: Number of materials specified by the client.
    *   $g$: Number of quantized gradients, calculated as $g = 30 \cdot 4^k + 2$, where $k$ is the number of refinement levels [S1, S2, S4, unofficial].
    *   Entry layout: Each entry stores pre-calculated color intensities (floating point values in range [0, 1.0]) for a specific material and quantized gradient combination [S1, S2, S4, unofficial].
    *   Material 0 is the null material; its entries store colors for ambient materials which do not require a gradient index [S1, S2, S4, unofficial].
    *   Storage: Floating point values are converted to integers and packed into 32-bit or 64-bit texture-map entries [S1, S2, S4, unofficial].
2.  **Voxel Fields:** Each voxel stores two indices [S3]:
    *   **Normal Index:** Encoded surface normal vector (computed via `vpNormalIndex()`) used to index the color table [S3].
    *   **Weight Index:** Used to index the weight table for fractional occupancy [S3].
3.  **Color Table:** Layout `float color_table[n][num_materials][color_channels]`, where $n$ is the number of possible values for the color field, and colors are in range 0.0-1.0 [S3].
4.  **Weight Table:** Layout `float weight_table[m][num_materials]`, where $m$ is the number of possible values for the weight field, and weights are in range 0.0-1.0 [S3]. If `num_materials` is 1, the weight table is not used [S3].

**Numbered Steps**
1.  **Setup Lighting:** Set light properties using `vpSetLight()`. For each light (VP_LIGHT0-VP_LIGHT5), set `VP_COLOR` (RGB components 0.0-1.0) and `VP_DIRECTION` (x, y, z components). The direction vector is transformed by the current modeling matrix [S3]. Enable the light using `vpEnable()`. Optionally select 'two-sided' lights [S3].
2.  **Initialize Tables:** Initialize the color table using `vpShadeTable()` after setting lighting and shading parameters [S3].
3.  **Declare Shader:** Declare the lookup-table shader using `vpSetLookupShader(vpc, color_channels, num_materials, color_field, color_table, color_table_size, weight_field, weight_table, weight_table_size)` [S3].
4.  **Classify Voxel:** For each voxel, classify its material index $M$ and calculate its field gradient [S1, S2, S4, unofficial].
5.  **Determine Gradient Index:**
    *   If the voxel has reflection properties (based on criteria from the parent method [S1, S2, S4, unofficial]), determine the quantized gradient index $G$ by finding the closest available quantized gradient from the tessellated sphere [S1, S2, S4, unofficial].
    *   If the voxel has no reflection properties, it is treated as ambient [S1, S2, S4, unofficial].
6.  **Calculate Lookup Index:**
    *   If reflecting: Final lookup index $I = M \cdot g + G$ [S1, S2, S4, unofficial].
    *   If not reflecting: Final lookup index $I = M$ [S1, S2, S4, unofficial].
7.  **Retrieve Color:** Use the normal index to look up a color for each material type from the color table. The actual colors depend on the surface normal, allowing directional lights [S3].
8.  **Combine Colors:** Use the weight index to look up fractional occupancy for each material type from the weight table. Combine the colors retrieved from the color table weighted by these fractional occupancies to determine the final voxel color [S3].

**Formulas**
*   **Lookup Index (Reflecting):** $I = M \cdot g + G$ [S1, S2, S4, unofficial].
*   **Lookup Index (Ambient):** $I = M$ [S1, S2, S4, unofficial].
*   **Number of Quantized Gradients:** $g = 30 \cdot 4^k + 2$ [S1, S2, S4, unofficial].
*   **Lookup Table Size:** $(m + 1) \times g$ entries [S1, S2, S4, unofficial].
*   **Gradient Magnitude (Contextual):** $s = \bar{\Delta}_x |\gamma_x| + \bar{\Delta}_y |\gamma_y| + \bar{\Delta}_z |\gamma_z|$ [S4, unofficial]. (Note: This formula appears in the source context for gradient calculation but is not explicitly defined in terms of $\bar{\Delta}$ or $\gamma$ in the provided notes beyond the equation itself).

**Parameters**
*   $m$: Number of materials specified by the client [S2, S4, unofficial].
*   $k$: Number of refinement levels for quantized gradients [S1, S4, unofficial].
*   Specularity Model: Uses the "half-way vector" specularity model [TM67, CT82] and Lambertian diffuse shading [S1, S2, S4, unofficial].

**Pitfalls**
*   The lookup table must be recalculated whenever any factor involving light changes, including a rotation of the volume or a movement of the light source [S1, S2, S4, unofficial].
*   Gradient quantization involves tessellating a sphere; the choice of tessellation affects the distribution of quantized gradients [S1, S2, S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Pre-computation of the lookup table is $O((m+1) \cdot g)$, where $g$ grows exponentially with refinement level $k$. Per-voxel lookup is $O(1)$ for index calculation and table access, plus $O(num\_materials)$ for color blending.
*   **Memory Complexity:** Lookup table size is $(m+1) \cdot g$ entries. Each entry stores color intensities (32-bit or 64-bit).
*   **Strengths:** Supports complex directional lighting (diffuse and specular) efficiently at render time via lookup. Supports multiple-material voxels with fractional occupancy.
*   **Limitations:** Requires recalculation of the lookup table when lighting or volume orientation changes, unlike methods without directional lighting [S2, S4, unofficial]. The table size grows with the number of materials and gradient refinement levels.
*   **When to use:** When realistic shading with directional lights is required and the lighting/volume orientation is static or changes infrequently.
*   **When not to use:** When dynamic lighting or frequent volume rotation is required and pre-computation overhead is prohibitive, or when directional lighting is not needed.

## Background
Originating from work on Direct Volume Rendering with Shading via Three-Dimensional Textures (1997) [S1, S2, S4, unofficial] and implemented in the VolPack library [S3], this method extends earlier volume rendering techniques by incorporating directional lighting effects. It departs from earlier work that interpreted gradient magnitude in isolation [S1, S2, S4, unofficial]. The method uses a half-way vector specularity model and Lambertian diffuse shading [S1, S2, S4, unofficial]. Gradient quantization is achieved by tessellating a unit sphere [S1, S2, S4, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
[S3] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S4] o,,ect Volume Rendering with Shading via Three- ..., https://ntrs.nasa.gov/api/citations/19970021689/downloads/19970021689.pdf?attachment=true, fetched 2026-10-29 (unofficial)
