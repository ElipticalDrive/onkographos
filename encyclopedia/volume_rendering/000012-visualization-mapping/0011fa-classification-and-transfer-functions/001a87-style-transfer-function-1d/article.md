# Style Transfer Function (1D)
Also called: STF

## Overview
A Style Transfer Function (STF) extends a 1D transfer function by incorporating surface normal direction into the domain, mapping scalar data values and eye-space normals to color and shading style [S1]. Unlike conventional transfer functions that assign direct RGBA tuples, STFs use lit sphere maps to provide flexible, non-photorealistic shading based on data values and normals [S1]. This method avoids the prohibitively high storage requirements of a single 3D/4D lookup table by storing discrete styles separately and using references in the lookup table [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous volumetric scalar field $f(P)$, where $P$ is the sample point [S1].
*   **Output:** Final color $(r, g, b)$ and directional opacity $\alpha_d$ for the sample [S1].

**Data Structures:**
The implementation relies on three textures [S1]:

1.  **Transfer Function Texture (tft):** A 1D texture storing $(i, \alpha_u)$ for each data value [S1].
    *   **Layout:** Width corresponds to the data value range (typically 256 units) [S1][S3, unofficial].
    *   **Fields:** Stores index $i$ and uniform opacity $\alpha_u$ [S1].
    *   **$i$:** Index value ranging from $0$ to $N-1$, where $N$ is the number of style nodes specified in the transfer function [S1]. Fractional values indicate interpolation between styles [S1].
    *   **$\alpha_u$:** Uniform opacity value in range $[0, 1]$, controlling overall opacity independent of normal direction [S1].

2.  **Index Function Texture (ift):** A 1D texture mapping index values $i$ ($0$ to $N-1$) from `tft` to style indices $j$ ($0$ to $M-1$) in `sft` [S1].
    *   **Layout:** Width equals $N$ (number of style nodes) [S1].
    *   **Fields:** Stores integer style indices $j$ [S1].
    *   **Note:** This mapping is discrete; no interpolation is performed [S1]. If no style is used multiple times, these lookups can be skipped [S1].

3.  **Style Function Texture (sft):** Contains the discrete set of $M$ distinct styles [S1].
    *   **Layout:** Each style is a 2D lit sphere map [S1]. Stored as a 2D texture (preferred), 3D texture (with mipmapping disabled), or texture array [S1].
    *   **Fields:** Stores $(r, g, b, \alpha_d)$ tuples indexed by eye-space normal direction [S1].
    *   **$\alpha_d$:** Directional opacity, varying based on normal direction [S1].

**Algorithm Steps:**
At each sample point $P$ in the volume:

1.  **Compute Scalar and Normal:**
    *   Compute scalar value $s = f(P)$ [S1].
    *   Compute gradient $g = \nabla f(P)$ [S1].
    *   Compute normalized eye-space normal $n = g / |g|$, with components $(n_x, n_y, n_z)$ [S1].

2.  **Lookup Uniform Properties:**
    *   Retrieve index $i$ and uniform opacity $\alpha_u$ from `tft` using sample value $s$:
        $$ (i, \alpha_u) = \text{tft}(s) $$

3.  **Compute Interpolation Weights:**
    *   Compute integer indices for the index function texture:
        $$ i_0 = \lfloor i \rfloor $$
        $$ i_1 = i_0 + 1 $$
    *   Compute interpolation weight:
        $$ w = i - i_0 $$

4.  **Retrieve Style Indices:**
    *   Perform two lookups into `ift` to get style indices $j_0$ and $j_1$:
        $$ j_0 = \text{ift}(i_0) $$
        $$ j_1 = \text{ift}(i_1) $$
    *   *Pitfall:* If no style is used multiple times in the transfer function, $j_0 = j_1$ and these lookups can be skipped [S1].

5.  **Interpolate Style and Opacity:**
    *   Perform two lookups into `sft` using the eye-space normal components $(n_x, n_y)$ and the style indices $j_0, j_1$ [S1].
    *   Linearly interpolate between the results:
        $$ (r, g, b, \alpha_d) = \text{sft}(n_x, n_y, j_0) \cdot (1 - w) + \text{sft}(n_x, n_y, j_1) \cdot w $$

6.  **Final Composition:**
    *   The final sample color is $(r, g, b)$ [S1].
    *   The overall opacity is computed as $\alpha = \alpha_u \cdot \alpha_d$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_1d(tex, coord):
    """Simulates 1D texture lookup returning (index, alpha_u) or (value)."""
    # Implementation depends on backend; returns tuple or float.
    pass

def sample_2d_style(tex, nx, ny, style_idx):
    """Simulates 2D/3D style texture lookup returning (r, g, b, alpha_d)."""
    # Implementation depends on backend; returns tuple.
    pass

def style_transfer_function_lookup(s, n_x, n_y, tft, ift, sft):
    """
    s: scalar data value (float)
    n_x, n_y: eye-space normal components (float)
    tft: 1D texture storing (index, alpha_u)
    ift: 1D texture storing style index j
    sft: 2D/3D texture storing (r, g, b, alpha_d) indexed by (nx, ny, style_index)
    """
    # Step 2: Lookup uniform properties
    i, alpha_u = sample_1d(tft, s)
    
    # Step 3: Compute weights
    i0 = int(i // 1)
    i1 = i0 + 1
    w = i - i0
    
    # Step 4: Retrieve style indices
    # Note: ift lookup is discrete, no interpolation
    j0 = sample_1d(ift, i0)
    j1 = sample_1d(ift, i1)
    
    # Step 5: Interpolate style
    # sft lookup uses nx, ny and style index
    style0 = sample_2d_style(sft, n_x, n_y, j0)
    style1 = sample_2d_style(sft, n_x, n_y, j1)
    
    # Linear interpolation of style tuples
    r = style0[0] * (1 - w) + style1[0] * w
    g = style0[1] * (1 - w) + style1[1] * w
    b = style0[2] * (1 - w) + style1[2] * w
    alpha_d = style0[3] * (1 - w) + style1[3] * w
    
    # Step 6: Final opacity
    alpha = alpha_u * alpha_d
    
    return (r, g, b, alpha)
```

**Parameters:**
*   **$T$:** User-defined thickness value for contour control [S1]. Typical value not specified [S1].
*   **Texture Width:** The resulting 1D transfer function texture typically has a fixed width of 256 [S3, unofficial].
*   **$\alpha_u$:** Uniform opacity range $[0, 1]$ [S1].
*   **$N$:** Number of styles specified in the style transfer function. Index values in `tft` range from $0$ to $N-1$ [S1].
*   **$M$:** Number of distinct styles in the style transfer function. Style indices in `sft` range from $0$ to $M-1$ [S1].

**Edge Cases and Pitfalls:**
*   **Memory:** A naive implementation using a single 3D texture for all data values and normal directions is impractical due to high storage requirements [S1].
*   **Interpolation Constraint:** Interpolation between styles must be uniform for all normal directions (transitions only between whole styles) unless intermediate styles are added [S1].
*   **Gradient Magnitude:** When gradient magnitude approaches zero, the normal direction becomes unreliable [S1].

## Subtopics

### [Mip-Mapping for Style Functions](782-mip-mapping-for-style-functions.md)
Mip-mapping for style functions is a technique to avoid aliasing in style lookups while preventing mixing between styles at higher mip-map levels. It builds on [Style Transfer Function (1D)](777-style-transfer-function-1d.md) by adding anti-aliasing capabilities to the discrete style selection process [G]. Unlike standard hardware mip-mapping, it manually computes mip-map levels to avoid undefined derivatives in raycasting loops and prevents style boundary corruption.

### [Multi-dimensional Style Transfer Function (2D)](813-multi-dimensional-style-transfer-function-2d.md)
This method extends the Style Transfer Function (1D) approach to map two input attributes, such as data value and gradient magnitude, to visual styles using a two-dimensional domain. It allows for finer control and more sophisticated visualization by leveraging the joint distribution of attributes, distinguishing it from the single-attribute mapping of its parent. Unlike standard 2D transfer functions that map directly to color/opacity, this method interpolates between discrete styles defined in a style function texture.

## Operating characteristics
*   **Time Complexity:** The lookup requires a maximum of four additional texture fetches compared to a regular 1D transfer function (two for `ift`, two for `sft`) [S1].
*   **Memory Complexity:** Significantly reduced compared to a full 3D/4D lookup table by storing discrete styles separately [S1].
*   **Measured Performance:** On GeForce 8 series hardware, the additional fetches incur overhead, but the cost of evaluating the illumination model is saved [S1]. The index function texture benefits heavily from texture caching due to its small size and lack of filtering requirements [S1].
*   **Strengths:** Allows flexible combination of different shading styles (e.g., unshaded, tone shading, cartoon, metallic) in a single rendering [S1]. Enables inconsistent lighting of different structures to accentuate features [S1]. Lit sphere maps provide an intuitive preview image for the style [S1].
*   **Limitations:** Requires Shader Model 3.0 GPUs for conditional loops and dynamic branching [S1]. Interpolation is limited to whole styles unless intermediate styles are added [S1].
*   **When to use:** When non-photorealistic, stylized volume rendering is desired, and the user wants to switch or blend between distinct artistic styles based on data values [S1].
*   **When not to use:** When photorealistic, physically-based lighting is required, or when memory bandwidth is extremely constrained and the overhead of multiple texture fetches is prohibitive [S1].

## Background
*   **Origin:** Introduced by Stefan Bruckner and Michael E. Gröller in the paper "Style Transfer Functions for Illustrative Volume Rendering" [S1].
*   **History:** The method was developed to address the difficulty of integrating multiple non-photorealistic rendering approaches into a single framework [S1]. It extends the concept of data-dependent lighting by allowing the user to specify styles captured from existing artwork directly in the transfer function [S1].
*   **Usage:** Implemented in C++ and OpenGL/GLSL for GPU-based ray casting [S1]. Used in illustrative volume rendering to generate high-quality volumetric illustrations with controlled contours and transparency [S1].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
[S2] Style Transfer Functions for Illustrative Volume Rendering, https://www.researchgate.net/publication/227615609_Style_Transfer_Functions_for_Illustrative_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering 102: Transfer Functions, http://graphicsrunner.blogspot.com/2009/01/volume-rendering-102-transfer-functions.html, fetched 2026-10-28 (unofficial)
