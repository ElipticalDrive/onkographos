# Style Transfer Function (1D)
Also called: 1D Style Transfer Function, 1D Transfer Function with Cubic Spline Interpolation, Lit Sphere Maps, STF, Style Transfer Function lookup, Style-based Volume Rendering

## Overview
A Style Transfer Function (STF) is a data-driven illumination technique that extends the traditional 1D transfer function by incorporating surface normal direction into the domain, allowing color and shading style to be defined as a function of both scalar data value and eye-space normal direction [S1, unofficial]. Unlike standard transfer functions that map data values directly to RGBA tuples, STFs use a lit sphere map to provide flexible, non-photorealistic shading based on data values and normals [S1, unofficial]. This method avoids the prohibitively high storage requirements of a single 3D/4D texture by storing discrete styles separately and using references in the lookup table [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous volumetric scalar field $f(P)$, where $P$ is the sample point.
*   **Output:** Final color $(r, g, b)$ and directional opacity $\alpha_d$ for the sample.

**Data Structures:**
The implementation relies on three textures [S1, unofficial]:

1.  **Transfer Function Texture (tft):** A 1D texture mapping data values to style indices and uniform opacity.
    *   **Layout:** Width corresponds to the data value range (typically 256 units).
    *   **Fields:** Stores a pair $(i, \alpha_u)$ for each data value.
    *   **$i$:** Index value ranging from $0$ to $N-1$, where $N$ is the number of style nodes specified in the transfer function. Fractional values indicate interpolation between styles.
    *   **$\alpha_u$:** Uniform opacity value in range $[0, 1]$, controlling overall opacity independent of normal direction.

2.  **Index Function Texture (ift):** A 1D texture mapping style indices to distinct style identifiers.
    *   **Layout:** Width equals $N$ (number of style nodes).
    *   **Fields:** Stores integer style indices $j$ ranging from $0$ to $M-1$, where $M$ is the number of distinct styles.
    *   **Note:** This mapping is discrete; no interpolation is performed. If no style is used multiple times, this texture can be skipped [S1, unofficial].

3.  **Style Function Texture (sft):** Contains the discrete set of $M$ distinct styles.
    *   **Layout:** Each style is a 2D image (lit sphere map). Stored as a 2D texture (preferred), 3D texture (with mipmapping disabled), or texture array [S1, unofficial].
    *   **Fields:** Stores $(r, g, b, \alpha_d)$ tuples indexed by eye-space normal direction.
    *   **$\alpha_d$:** Directional opacity, varying based on normal direction.

**Algorithm Steps:**
At each sample point $P$ in the volume:

1.  **Compute Scalar and Normal:**
    *   Compute scalar value $s = f(P)$.
    *   Compute gradient $g = \nabla f(P)$.
    *   Compute normalized eye-space normal $n = g / |g|$. Let $n = (n_x, n_y, n_z)$ [S1, unofficial].

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
    *   *Pitfall:* If no style is used multiple times in the transfer function, $j_0 = j_1$ and these lookups can be skipped [S1, unofficial].

5.  **Interpolate Style and Opacity:**
    *   Perform two lookups into `sft` using the eye-space normal components $(n_x, n_y)$ and the style indices $j_0, j_1$.
    *   Linearly interpolate between the results:
        $$ (r, g, b, \alpha_d) = \text{sft}(n_x, n_y, j_0) \cdot (1 - w) + \text{sft}(n_x, n_y, j_1) \cdot w $$

6.  **Final Composition:**
    *   The final sample color is $(r, g, b)$.
    *   The final opacity is the product of uniform and directional opacity: $\alpha = \alpha_u \cdot \alpha_d$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def style_transfer_function_lookup(s, n_x, n_y, tft, ift, sft):
    """
    s: scalar data value (float)
    n_x, n_y: eye-space normal components (float)
    tft: 1D texture storing (index, alpha_u)
    ift: 1D texture storing style index j
    sft: 3D/2D texture storing (r, g, b, alpha_d) indexed by (nx, ny, style_index)
    """
    # Step 2: Lookup uniform properties
    i, alpha_u = tft.sample_1d(s)
    
    # Step 3: Compute weights
    i0 = floor(i)
    i1 = i0 + 1
    w = i - i0
    
    # Step 4: Retrieve style indices
    # Note: ift lookup is discrete, no interpolation
    j0 = ift.sample_1d(i0)
    j1 = ift.sample_1d(i1)
    
    # Step 5: Interpolate style
    # sft lookup uses nx, ny and style index
    style0 = sft.sample_2d_style(n_x, n_y, j0)
    style1 = sft.sample_2d_style(n_x, n_y, j1)
    
    # Linear interpolation of style tuples
    r = style0.r * (1 - w) + style1.r * w
    g = style0.g * (1 - w) + style1.g * w
    b = style0.b * (1 - w) + style1.b * w
    alpha_d = style0.a * (1 - w) + style1.a * w
    
    # Step 6: Final opacity
    alpha = alpha_u * alpha_d
    
    return (r, g, b, alpha)
```

**Parameters:**
*   **$T$:** User-defined thickness value for contour control. Used in curvature threshold calculations. Typical value not specified [S1, unofficial].
*   **Texture Width:** The resulting 1D transfer function texture typically has a fixed width of 256 [S2, unofficial].
*   **$\alpha_u$:** Uniform opacity range $[0, 1]$ [S1, unofficial].
*   **$N$:** Number of styles specified in the style transfer function. Index values in `tft` range from $0$ to $N-1$ [S1, unofficial].
*   **$M$:** Number of distinct styles in the style transfer function. Style indices in `sft` range from $0$ to $M-1$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory:** A naive implementation using a single 3D texture for all data values and normal directions is impractical due to high storage requirements [S1, unofficial].
*   **Interpolation Constraint:** Interpolation between styles must be uniform for all normal directions (transitions only between whole styles) unless intermediate styles are added [S1, unofficial].
*   **Gradient Magnitude:** When gradient magnitude approaches zero, the normal direction becomes unreliable. This affects the validity of the lit sphere map lookup [S1, unofficial].

## Subtopics

### [Mip-Mapping for Style Functions](782-mip-mapping-for-style-functions.md)
Mip-mapping for style functions is a technique to avoid aliasing in style lookups while preventing mixing between styles at higher mip-map levels. It builds on [Style Transfer Function (1D)](777-style-transfer-function-1d.md) by adding anti-aliasing capabilities to the discrete style selection process. Unlike standard hardware mip-mapping which may blend distinct styles, this method ensures temporal coherence and spatial anti-aliasing without corrupting style boundaries.

### Multi-dimensional Style Transfer Function (2D)
An extension of the style transfer function to two-dimensional domains (e.g., data value and gradient magnitude, or data value and object identifier) using bilinear interpolation across four styles [S1, unofficial].
*   **Definition:** Extends the 1D concept to 2D textures for both the transfer function and index function [S1, unofficial].
*   **Step:**
    1.  Compute indices: $ix_0 = \lfloor ix \rfloor, iy_0 = \lfloor iy \rfloor, ix_1 = ix_0 + 1, iy_1 = iy_0 + 1$.
    2.  Compute weights: $wx = ix - ix_0, wy = iy - iy_0$.
    3.  Perform four lookups into `ift`: $j_{00} = \text{ift}(ix_0, iy_0), j_{10} = \text{ift}(ix_1, iy_0), j_{01} = \text{ift}(ix_0, iy_1), j_{11} = \text{ift}(ix_1, iy_1)$.
    4.  Perform four lookups into `sft` using $(n_x, n_y)$ and $j_{00}, j_{10}, j_{01}, j_{11}$.
    5.  Compute final color via bilinear interpolation:
        $$ (r,g,b,\alpha_d) = (\text{sft}(n_x,n_y,j_{00}) \cdot (1-wx) + \text{sft}(n_x,n_y,j_{10}) \cdot wx) \cdot (1-wy) + (\text{sft}(n_x,n_y,j_{01}) \cdot (1-wx) + \text{sft}(n_x,n_y,j_{11}) \cdot wx) \cdot wy $$
*   **Layout:**
    *   **Transfer function texture:** 2D texture storing two indices $ix$ and $iy$. $ix$ increases along horizontal axis, $iy$ along vertical [S1, unofficial].
    *   **Index function texture:** 2D texture. Width = max horizontal style nodes, Height = max vertical nodes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The lookup requires a maximum of four additional texture fetches compared to a regular 1D transfer function (two for `ift`, two for `sft`) [S1, unofficial].
*   **Memory Complexity:** Significantly reduced compared to a full 3D/4D lookup table by storing discrete styles separately [S1, unofficial].
*   **Measured Performance:** On GeForce 8 series hardware, the additional fetches incur overhead, but the cost of evaluating the illumination model is saved. The index function texture benefits heavily from texture caching [S1, unofficial].
*   **Strengths:** Allows flexible combination of different shading styles (e.g., unshaded, tone shading, cartoon, metallic) in a single rendering. Enables inconsistent lighting of different structures to accentuate features [S1, unofficial]. Lit sphere maps provide an intuitive preview image for the style [S1, unofficial].
*   **Limitations:** Requires Shader Model 3.0 GPUs for conditional loops and dynamic branching. Interpolation is limited to whole styles unless intermediate styles are added [S1, unofficial].
*   **When to use:** When non-photorealistic, stylized volume rendering is desired, and the user wants to switch or blend between distinct artistic styles based on data values.
*   **When not to use:** When photorealistic, physically-based lighting is required, or when memory bandwidth is extremely constrained and the overhead of multiple texture fetches is prohibitive.

## Background
*   **Origin:** Introduced by Stefan Bruckner and Michael E. Gröller in the paper "Style Transfer Functions for Illustrative Volume Rendering" [S1, unofficial].
*   **History:** The method was developed to address the difficulty of integrating multiple non-photorealistic rendering approaches into a single framework. It extends the concept of data-dependent lighting by allowing the user to specify styles captured from existing artwork directly in the transfer function [S1, unofficial].
*   **Usage:** Implemented in C++ and OpenGL/GLSL for GPU-based ray casting. Used in illustrative volume rendering to generate high-quality volumetric illustrations with controlled contours and transparency [S1, unofficial].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://www.researchgate.net/publication/227615609_Style_Transfer_Functions_for_Illustrative_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering 102: Transfer Functions, http://graphicsrunner.blogspot.com/2009/01/volume-rendering-102-transfer-functions.html, fetched 2026-10-28 (unofficial)
