# Style Transfer Function (1D)
Also called: STF

## Overview
Style Transfer Function (1D) is a data-driven illumination technique that extends the traditional 1D transfer function by incorporating surface normal direction into the domain, allowing users to specify non-photorealistic shading styles captured from existing artwork [S1][S2, unofficial]. Unlike conventional transfer functions that map scalar data values directly to a single RGBA color, this method maps data values to references of discrete shading styles (represented as lit sphere maps), where the final color is determined by the eye-space normal direction sampling the style [S1][S2, unofficial]. It builds on [Classification and transfer functions](38-classification-and-transfer-functions.md) by replacing direct color assignment with style-based illumination, enabling flexible combinations of shading styles (e.g., cartoon, metallic, painterly) within a single volume rendering [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A continuous volumetric scalar field $f(\mathbf{P})$, where $\mathbf{P}$ is the sample position.
*   **Output:** Final sample color $(r, g, b)$ and opacity $\alpha$.

**Data Structures:**
The method decomposes the high-dimensional lookup into three separate textures to avoid prohibitive memory requirements [S1][S2, unofficial]:

1.  **Transfer Function Texture (tft):** A 1D texture mapping data values $s \in [0, 255]$ (or normalized range) to a pair $(i, \alpha_u)$.
    *   $i$: A floating-point index value ranging from $0$ to $N-1$, where $N$ is the number of style entries defined in the transfer function. Fractional values indicate interpolation between two adjacent style entries.
    *   $\alpha_u$: Uniform opacity value in $[0, 1]$, controlling overall opacity independent of normal direction [S2, unofficial].
2.  **Index Function Texture (ift):** A 1D texture mapping the continuous index $i$ from `tft` to discrete style indices $j$.
    *   Input: Index $i \in [0, N-1]$.
    *   Output: Style index $j \in [0, M-1]$, where $M$ is the number of distinct styles.
    *   Note: This mapping is discrete; no interpolation is performed during lookup. If each style is used only once, this texture can be skipped [S2, unofficial].
3.  **Style Function Texture (sft):** Stores $M$ distinct styles. Each style is a 2D image (lit sphere map) representing color and directional opacity $(r, g, b, \alpha_d)$ as a function of eye-space normal direction.
    *   Storage: Implemented as a 2D texture (preferred), 3D texture (with mipmapping disabled), or texture array [S2, unofficial].
    *   Domain: Eye-space normal components $(n_x, n_y)$ projected onto the sphere.

**Numbered Steps:**
1.  **Reconstruct Data:** At each sample point $\mathbf{P}$, compute the scalar value $s = f(\mathbf{P})$ and the gradient $\mathbf{g} = \nabla f(\mathbf{P})$ [S1][S2, unofficial].
2.  **Compute Normal:** Compute the normalized eye-space normal $\mathbf{n} = \mathbf{g} / \|\mathbf{g}\|$ [S2, unofficial].
3.  **Lookup Uniform Properties:** Sample `tft` at $s$ to retrieve the style index $i$ and uniform opacity $\alpha_u$:
    $$ (i, \alpha_u) = \text{tft}(s) $$
4.  **Determine Interpolation Weights:** Compute integer indices and weight for style interpolation:
    $$ i_0 = \lfloor i \rfloor, \quad i_1 = i_0 + 1, \quad w = i - i_0 $$
5.  **Map to Distinct Styles:** Retrieve the discrete style indices $j_0$ and $j_1$ corresponding to $i_0$ and $i_1$ using `ift`:
    $$ j_0 = \text{ift}(i_0), \quad j_1 = \text{ift}(i_1) $$
    *   If no style is used multiple times in the transfer function, $j_0 = i_0$ and $j_1 = i_1$, and this step can be skipped [S2, unofficial].
6.  **Interpolate Style:** Perform two lookups into `sft` using the normal components $(n_x, n_y)$ and the style indices $j_0, j_1$. Linearly interpolate between the results:
    $$ (r, g, b, \alpha_d) = \text{sft}(n_x, n_y, j_0) \cdot (1 - w) + \text{sft}(n_x, n_y, j_1) \cdot w $$
    *   The interpolation is performed uniformly for all normal directions, ensuring transitions occur only between whole styles [S1][S2, unofficial].
7.  **Compute Final Opacity:** Combine uniform and directional opacity. The sources define $\alpha_u$ as controlling overall opacity and $\alpha_d$ as the directional component from the style. The final opacity $\alpha$ is derived from these, typically as a product or weighted combination to maintain semantic opacity control (e.g., if $\alpha_u=1$, the sample should not be fully transparent due to $\alpha_d$ alone) [S2, unofficial].
8.  **Final Color:** The color $(r, g, b)$ from step 6 is the final color contribution.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def style_transfer_lookup(s, n_x, n_y, tft, ift, sft):
    """
    s: scalar data value (float)
    n_x, n_y: eye-space normal components (float)
    tft: 1D texture storing (index, alpha_u)
    ift: 1D texture storing style index j
    sft: 2D/3D texture storing style colors/opacities
    """
    # Step 3: Lookup uniform properties
    index_float, alpha_u = tft.sample(s)
    
    # Step 4: Compute interpolation weights
    i0 = floor(index_float)
    i1 = i0 + 1
    w = index_float - i0
    
    # Step 5: Map to distinct styles
    # Check if ift is needed (if styles are unique, j0=i0, j1=i1)
    j0 = ift.sample(i0)
    j1 = ift.sample(i1)
    
    # Step 6: Interpolate style
    # sft lookup returns (r, g, b, alpha_d)
    style0 = sft.sample(n_x, n_y, j0)
    style1 = sft.sample(n_x, n_y, j1)
    
    r = style0.r * (1 - w) + style1.r * w
    g = style0.g * (1 - w) + style1.g * w
    b = style0.b * (1 - w) + style1.b * w
    alpha_d = style0.alpha * (1 - w) + style1.alpha * w
    
    # Step 7: Combine opacities
    # Constraint: alpha_u=1 should not result in full transparency
    # Implementation detail: typically alpha = alpha_u * alpha_d or similar blend
    # Source [S2, unofficial] implies alpha_d is weighted by alpha_u or combined to maintain semantics.
    # A common safe implementation for illustrative rendering:
    alpha = alpha_u * alpha_d
    
    return (r, g, b, alpha)
```

**Parameters:**
*   $T$: User-defined thickness value for contour control (used in curvature calculations, not part of the core lookup but part of the illustrative framework) [S2, unofficial].
*   $N$: Number of style entries in `tft`.
*   $M$: Number of distinct styles in `sft`.

**Edge Cases and Pitfalls:**
*   **Gradient Magnitude:** When gradient magnitude approaches zero, the normal direction becomes unreliable. Biasing the mip-map level or handling degenerate normals is required [S2, unofficial].
*   **Memory:** A naive 3D/4D lookup table is impractical due to high storage requirements. The decomposition into `tft`, `ift`, and `sft` is necessary [S1][S2, unofficial].
*   **Interpolation Restriction:** Basic implementation requires uniform interpolation across all normal directions. Non-uniform transitions require adding intermediate styles [S1][S2, unofficial].

## Subtopics

### [Mip-Mapping for Style Functions](782-mip-mapping-for-style-functions.md)
Adds anti-aliasing to style lookups by manually computing mip-map levels to avoid undefined derivatives and style boundary corruption.

### [Multi-dimensional Style Transfer Function (2D)](813-multi-dimensional-style-transfer-function-2d.md)
Extends the domain to two input attributes (e.g., value and gradient), interpolating between discrete styles defined in a style function texture.

### [GPU Implementation of Style Transfer Function Lookup](5389-gpu-implementation-of-style-transfer-function-lookup.md)
Optimizes the lookup by decomposing the mapping into separate 1D and 3D textures to solve memory bandwidth inefficiencies.

## Operating characteristics
*   **Time Complexity:** The lookup requires a maximum of 4 additional texture fetches compared to a regular 1D transfer function (2 for `ift`, 2 for `sft`) [S2, unofficial]. The `ift` texture is small and benefits from caching; it can be implemented as a buffer texture for performance gains on specific hardware (e.g., GeForce 8 series) [S2, unofficial].
*   **Memory Complexity:** Significantly reduced compared to a full high-dimensional lookup table. Storage is proportional to $N$ (entries in `tft`) and $M$ (distinct styles in `sft`), rather than the product of data range and normal resolution [S1][S2, unofficial].
*   **Performance:** On GeForce 8 series hardware, the overhead of additional fetches is offset by the savings from avoiding explicit illumination model evaluation (e.g., Phong shading) in the shader [S2, unofficial].
*   **Strengths:** Enables flexible combination of multiple non-photorealistic styles (unshaded, tone, cartoon, metallic) in a single rendering; allows inconsistent lighting to accentuate features; lit sphere maps provide intuitive preview images [S1][S2, unofficial].
*   **Limitations:** Requires Shader Model 3.0 or higher for dynamic branching and texture sampling; interpolation is restricted to whole styles unless intermediate styles are added; relies on accurate normal estimation [S1][S2, unofficial].

## Background
The method was introduced by Stefan Bruckner and Michael E. Gröller in the paper "Style Transfer Functions for Illustrative Volume Rendering" (2007) [S1][S2, unofficial]. It was developed to address the difficulty of integrating multiple non-photorealistic rendering approaches into a single framework. The technique allows users to specify styles captured from existing artwork, enabling the generation of volumetric illustrations in the style of a given work of art [S2, unofficial]. It extends previous work on data-dependent lighting (e.g., Lum et al. [LM04]) by replacing fixed illumination models with image-based lighting via lit sphere maps [S1][S2, unofficial].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
[S2] Style Transfer Functions for Illustrative Volume Rendering, https://www.researchgate.net/publication/227615609_Style_Transfer_Functions_for_Illustrative_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering 102: Transfer Functions, http://graphicsrunner.blogspot.com/2009/01/volume-rendering-102-transfer-functions.html, fetched 2026-10-28 (unofficial)
