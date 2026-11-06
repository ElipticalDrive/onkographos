# Reflectance Map Shading

Also called: 6n table shading, Cube Map Dependent Texture Reads for Shading, Cube-map Phong shading

## Overview
Reflectance Map Shading is a local illumination optimization that evaluates the Blinn-Phong shading model per-pixel using cube map dependent texture reads, thereby avoiding the computational cost of vector normalization [S1, book]. It differs from standard Phong shading by replacing per-pixel dot-product calculations and normalization with a direct lookup into a pre-computed reflectance map organized as six $n \times n$ tables [S2].

## How it works
**Inputs and Outputs**
*   **Input:** A sample gradient vector $\mathbf{G}$ (representing the surface normal approximation) and the object's base color.
*   **Output:** The final shaded RGB intensity for the sample.

**Data Structures**
The reflectance map is a lookup table (LUT) organized as six separate $n \times n$ textures (or tables), corresponding to the six faces of a cube centered at the origin [S2].
*   **Layout:** Each face covers one octant of the normal space. The index enumerates the six major directions [S2].
*   **Content:** Each texel stores the pre-computed RGB intensities resulting from the Phong shading model (ambient, diffuse, and specular contributions) for the specific normal direction mapped to that texel [S2].
*   **Precision:** Typical implementations use 5 bits per index, resulting in $n=32$ [S2].

**Steps**
1.  **Compute Gradient:** Calculate the gradient vector $\mathbf{G}$ at the sample position. In volume rendering, this is often done via central differences on the scalar field [S2].
2.  **Normalize by Maximum Component:** Determine the component of $\mathbf{G}$ with the largest absolute magnitude. Divide all components of $\mathbf{G}$ by this maximum value to form the normalized gradient vector $\mathbf{G}'$ [S2].
    $$ \mathbf{G}' = \frac{\mathbf{G}}{\max(|G_x|, |G_y|, |G_z|)} $$
    This maps the vector to the surface of a cube rather than a sphere, allowing it to be indexed directly by the cube face [S2].
3.  **Determine Cube Face:** Identify which of the six faces of the cube $\mathbf{G}'$ intersects. This is determined by the sign and magnitude of the components of $\mathbf{G}'$ [S2].
4.  **Lookup:** Use the components of $\mathbf{G}'$ as texture coordinates $(s, t, r)$ to access the texel on the identified cube face [S1, book]. The texel represents the point corresponding to the intersection of a line through the origin in the direction of $(s, t, r)$ and the cube face [S1, book].
    *   If $\mathbf{G}$ is interpreted as a surface normal, the texel value represents incoming diffuse radiance [S1, book].
    *   If $\mathbf{G}$ is interpreted as a reflection vector, the texel value represents specular radiance [S1, book].
5.  **Modulate:** Multiply the retrieved RGB intensities by the object's base color to form the final shaded sample intensity [S2].

**Parameters**
*   **LUT Index Precision:** 5 bits per index is considered the largest manageable size for a 4-index LUT, resulting in a total size of $32^4 \times 4 \text{ Bytes} = 4 \text{ MBytes}$ [S2].
*   **Table Size ($n$):** Determined by the bit precision. For 5 bits, $n=32$ [S2].

**Edge Cases and Pitfalls**
*   **Zero Gradient:** Gradient-based surface shading is not robust for homogeneous regions where the gradient magnitude is very low or zero, as the measurement is sensitive to noise [S1, book].
*   **Performance:** Per-pixel cube map evaluation is significantly slower (up to three times) than evaluating dot products for a limited number of light sources in the fragment shader stage [S1, book].
*   **Interpolation Artifacts:** Surface-based shading methods can suffer from artifacts caused by denormalization during interpolation if not handled correctly [S1, book].

## Operating characteristics
*   **Memory Complexity:** The method significantly reduces memory requirements compared to a full 4-index 8-bit LUT (which requires 16 GBytes) by normalizing the gradient vector [S2]. A 5-bit, 4-index LUT requires approximately 4 MBytes [S2].
*   **Time Complexity:** The lookup is $O(1)$ per sample, avoiding the square root operation required for normalization [S1, book]. However, the texture fetch itself is more expensive than simple arithmetic.
*   **Strengths:** Delivers good image quality with fast LUT creation and simple lookup logic [S2]. It allows the cube map to encode an arbitrary number of lights or a full environment map [S1, book].
*   **Limitations:** Performance is lower than analytical dot-product shading for scenes with few lights [S1, book]. It is unsuitable for shading volume renderings of multivariate fields [S1, book].

## Background
Reflectance map shading was introduced as an optimization for volume rendering to avoid the high cost of per-pixel Phong calculations. It leverages hardware support for cube map dependent texture reads, which allows an RGB color component to be treated as a vector for texture coordinate generation [S1, book]. The approach was chosen for implementations because it balances image quality with memory constraints and computational simplicity [S2].

## Sources
[S1] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
[S2] High-Quality Volume Rendering Using Texture Mapping ..., https://cfcs.pku.edu.cn/baoquan/docs/20180622110034067331.pdf, fetched 2026-10-27
