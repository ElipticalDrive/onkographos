# Depth-Aware Upsampling


## Overview
Depth-aware upsampling reduces blue noise and performance costs in volumetric rendering by rendering clouds to a low-resolution RGBA texture and a distance buffer, then upsampling to high resolution using the depth buffer [S1, unofficial]. It sets itself apart from standard bilinear upsampling by weighting the interpolation of low-resolution samples based on the depth difference between the high-resolution fragment and the low-resolution depth values, ensuring that samples from similar depth planes contribute more to the final color [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `clouds`: A low-resolution RGBA texture containing the pre-rendered cloud color and alpha [S1, unofficial].
    *   `dist`: A low-resolution single-channel (R) texture containing the depth/distance values corresponding to the `clouds` texture [S1, unofficial].
    *   `depth`: The high-resolution depth value of the current fragment being shaded [S1, unofficial].
    *   `cloud_subsampling`: Integer factor by which the low-resolution texture is smaller than the high-resolution output [S1, unofficial].
    *   `overlay_width`, `overlay_height`: Dimensions of the low-resolution textures [S1, unofficial].
    *   `depth_sigma`: Float parameter controlling the sensitivity of the depth weighting (standard deviation of the Gaussian) [S1, unofficial].
    *   `min_depth_exponent`: Float lower bound for clamping Gaussian exponents to prevent numerical underflow [S1, unofficial].
*   **Output:**
    *   `vec4`: The high-resolution RGBA color value for the current fragment [S1, unofficial].

**Data Structures:**
*   `clouds`: `sampler2D`, format RGBA [S1, unofficial].
*   `dist`: `sampler2D`, format R (or grayscale) [S1, unofficial].

**Parameters:**
*   `MAX_EXPONENT`: 20.0. Used to clamp the upper bound of Gaussian exponents [S1, unofficial].
*   `depth_sigma`: Typical values depend on scene scale and noise characteristics; not explicitly given in sources [S1, unofficial].
*   `min_depth_exponent`: Not explicitly given in sources, but used to clamp negative exponents [S1, unofficial].

**Steps:**
1.  **Calculate Low-Resolution Coordinates:**
    Convert the high-resolution fragment coordinates (`gl_FragCoord.xy`) to low-resolution space by dividing by `cloud_subsampling` [S1, unofficial].
    $$ point = \frac{gl\_FragCoord.xy}{cloud\_subsampling} $$
2.  **Determine Bilinear Grid:**
    Calculate the integer texel coordinate (`texcoord`) and the fractional offset (`frac`) for bilinear interpolation [S1, unofficial]. The offset is centered by subtracting 0.5 [S1, unofficial].
    $$ texcoord = \lfloor point - 0.5 \rfloor + 0.5 $$
    $$ frac = \text{fract}(point - 0.5) $$
    Where $\lfloor \cdot \rfloor$ is the floor function and $\text{fract}(x) = x - \lfloor x \rfloor$ [G].
3.  **Compute Neighbor Texcoords:**
    Calculate the normalized texture coordinates for the four surrounding low-resolution texels (00, 01, 10, 11) [S1, unofficial]. The coordinates are adjusted by the fractional part and normalized by the overlay dimensions [S1, unofficial].
    $$ texcoord_{00} = \frac{texcoord + (0.0, 0.0)}{(overlay\_width, overlay\_height)} $$
    $$ texcoord_{01} = \frac{texcoord + (1.0, 0.0)}{(overlay\_width, overlay\_height)} $$
    $$ texcoord_{10} = \frac{texcoord + (0.0, 1.0)}{(overlay\_width, overlay\_height)} $$
    $$ texcoord_{11} = \frac{texcoord + (1.0, 1.0)}{(overlay\_width, overlay\_height)} $$
4.  **Sample Low-Resolution Data:**
    Sample the color values from the `clouds` texture and the depth values from the `dist` texture at the four neighbor coordinates [S1, unofficial].
    $$ value_{ij} = \text{texture}(clouds, texcoord_{ij}) $$
    $$ dist_{ij} = \text{texture}(dist, texcoord_{ij}).r $$
    Where $i,j \in \{0,1\}$ [S1, unofficial].
5.  **Calculate Depth-Weighted Exponents:**
    Compute the Gaussian exponent for each neighbor based on the squared difference between the high-resolution depth and the low-resolution depth, normalized by `depth_sigma` [S1, unofficial].
    $$ exponent_{ij} = -\frac{(depth - dist_{ij})^2}{2 \cdot depth\_sigma^2} $$
6.  **Clamp Exponents:**
    Clamp the exponents to the range $[min\_depth\_exponent, MAX\_EXPONENT]$ to guard against numerical underflows and extreme weights [S1, unofficial].
    $$ exponent'_{ij} = \text{clamp}(exponent_{ij}, min\_depth\_exponent, 20.0) $$
7.  **Compute Weights:**
    Calculate the final weight for each neighbor by multiplying the standard bilinear interpolation factor by the exponential depth weight [S1, unofficial].
    $$ weight_{00} = (1.0 - frac.x) \cdot (1.0 - frac.y) \cdot \exp(exponent'_{00}) $$
    $$ weight_{01} = frac.x \cdot (1.0 - frac.y) \cdot \exp(exponent'_{01}) $$
    $$ weight_{10} = (1.0 - frac.x) \cdot frac.y \cdot \exp(exponent'_{10}) $$
    $$ weight_{11} = frac.x \cdot frac.y \cdot \exp(exponent'_{11}) $$
8.  **Return Weighted Average:**
    Compute the final color as the weighted sum of the sampled colors, normalized by the sum of the weights [S1, unofficial].
    $$ result = \frac{value_{00} \cdot weight_{00} + value_{01} \cdot weight_{01} + value_{10} \cdot weight_{10} + value_{11} \cdot weight_{11}}{weight_{00} + weight_{01} + weight_{10} + weight_{11}} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def depth_aware_upsample(gl_frag_coord, clouds_sampler, dist_sampler, 
                         cloud_subsampling, overlay_width, overlay_height, 
                         depth_sigma, min_depth_exponent, current_depth):
    """
    Performs depth-aware upsampling of a low-res cloud texture to high-res.
    
    Args:
        gl_frag_coord: (x, y) High-resolution fragment coordinates.
        clouds_sampler: Function to sample low-res RGBA texture.
        dist_sampler: Function to sample low-res depth texture.
        cloud_subsampling: Integer downsampling factor.
        overlay_width: Width of low-res textures.
        overlay_height: Height of low-res textures.
        depth_sigma: Standard deviation for depth weighting Gaussian.
        min_depth_exponent: Lower bound for clamping exponents.
        current_depth: Depth value of the current high-res fragment.
        
    Returns:
        vec4: High-resolution RGBA color.
    """
    MAX_EXPONENT = 20.0
    
    # 1. Calculate low-res point
    point_x = gl_frag_coord[0] / cloud_subsampling
    point_y = gl_frag_coord[1] / cloud_subsampling
    
    # 2. Determine integer texcoord and fractional part
    # floor(point - 0.5) + 0.5 centers the grid
    texcoord_x = math.floor(point_x - 0.5) + 0.5
    texcoord_y = math.floor(point_y - 0.5) + 0.5
    
    frac_x = (point_x - 0.5) - math.floor(point_x - 0.5)
    frac_y = (point_y - 0.5) - math.floor(point_y - 0.5)
    
    # 3. Compute neighbor texcoords (normalized)
    # Matches GLSL swizzle: vector01=(0,1), .xx=(0,0), .yx=(1,0), .xy=(0,1), .yy=(1,1)
    texcoord00 = ((texcoord_x + 0.0) / overlay_width, (texcoord_y + 0.0) / overlay_height)
    texcoord01 = ((texcoord_x + 1.0) / overlay_width, (texcoord_y + 0.0) / overlay_height)
    texcoord10 = ((texcoord_x + 0.0) / overlay_width, (texcoord_y + 1.0) / overlay_height)
    texcoord11 = ((texcoord_x + 1.0) / overlay_width, (texcoord_y + 1.0) / overlay_height)
    
    # 4. Sample low-res data
    value00 = clouds_sampler(texcoord00) # vec4
    value01 = clouds_sampler(texcoord01)
    value10 = clouds_sampler(texcoord10)
    value11 = clouds_sampler(texcoord11)
    
    dist00 = dist_sampler(texcoord00)[0] # Assuming .r component
    dist01 = dist_sampler(texcoord01)[0]
    dist10 = dist_sampler(texcoord10)[0]
    dist11 = dist_sampler(texcoord11)[0]
    
    # 5. Calculate Gaussian exponents
    denom = 2.0 * depth_sigma * depth_sigma
    
    exponent00 = -((current_depth - dist00) ** 2) / denom
    exponent01 = -((current_depth - dist01) ** 2) / denom
    exponent10 = -((current_depth - dist10) ** 2) / denom
    exponent11 = -((current_depth - dist11) ** 2) / denom
    
    # 6. Clamp exponents
    exponent00 = max(min_depth_exponent, min(exponent00, MAX_EXPONENT))
    exponent01 = max(min_depth_exponent, min(exponent01, MAX_EXPONENT))
    exponent10 = max(min_depth_exponent, min(exponent10, MAX_EXPONENT))
    exponent11 = max(min_depth_exponent, min(exponent11, MAX_EXPONENT))
    
    # 7. Compute weights
    # Bilinear factors * Depth weights
    weight00 = (1.0 - frac_x) * (1.0 - frac_y) * math.exp(exponent00)
    weight01 = frac_x * (1.0 - frac_y) * math.exp(exponent01)
    weight10 = (1.0 - frac_x) * frac_y * math.exp(exponent10)
    weight11 = frac_x * frac_y * math.exp(exponent11)
    
    # 8. Return weighted average
    total_weight = weight00 + weight01 + weight10 + weight11
    
    # Fallback for zero total weight [G]
    if total_weight == 0:
        return (value00[0] + value01[0] + value10[0] + value11[0]) / 4.0, \
               (value00[1] + value01[1] + value10[1] + value11[1]) / 4.0, \
               (value00[2] + value01[2] + value10[2] + value11[2]) / 4.0, \
               (value00[3] + value01[3] + value10[3] + value11[3]) / 4.0
        
    result_r = (value00[0] * weight00 + value01[0] * weight01 + 
                value10[0] * weight10 + value11[0] * weight11) / total_weight
    result_g = (value00[1] * weight00 + value01[1] * weight01 + 
                value10[1] * weight10 + value11[1] * weight11) / total_weight
    result_b = (value00[2] * weight00 + value01[2] * weight01 + 
                value10[2] * weight10 + value11[2] * weight11) / total_weight
    result_a = (value00[3] * weight00 + value01[3] * weight01 + 
                value10[3] * weight10 + value11[3] * weight11) / total_weight
                
    return (result_r, result_g, result_b, result_a)
```

**Edge Cases and Pitfalls:**
*   **Numerical Underflow:** If `depth_sigma` is very small or depth differences are large, `exponent` can become very negative, causing `exp()` to underflow to zero [S1, unofficial]. Clamping to `min_depth_exponent` prevents this [S1, unofficial].
*   **Zero Total Weight:** If all weights are zero (e.g., due to aggressive clamping), division by zero occurs [G]. A fallback to simple averaging or zero is required [G].
*   **Boundary Conditions:** The calculation assumes `texcoord` falls within the valid range of the low-res texture [G]. If the high-res fragment is near the edge of the screen, `texcoord` might point outside the low-res texture bounds [G]. Standard texture sampling behavior (clamp, repeat, or border color) will apply, but this may introduce artifacts at the edges of the cloud volume [G].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment [G]. It performs a fixed number of texture lookups (8 total: 4 for color, 4 for depth) and arithmetic operations [G].
*   **Memory Complexity:** $O(1)$ additional memory per fragment, plus the storage for the low-resolution `clouds` and `dist` textures [G].
*   **Measured Performance:** Reduces performance cost compared to rendering clouds at full resolution with dense sampling, while reducing blue noise compared to simple low-resolution rendering [S1, unofficial].
*   **Strengths:**
    *   Significantly reduces blue noise artifacts compared to naive low-resolution rendering [S1, unofficial].
    *   Avoids the performance penalty of increasing the sampling interval (number of raymarching steps) at high resolution [S1, unofficial].
    *   Simple to implement as a post-process or overlay shader [S1, unofficial].
*   **Limitations:**
    *   Relies on the accuracy of the low-resolution depth buffer [G]. If the low-res depth is inaccurate or lacks detail, the upsampling may blur or misplace cloud features [G].
    *   Requires rendering an additional distance buffer [S1, unofficial].
    *   Quality depends heavily on the choice of `depth_sigma` [G]. Too small a sigma makes the upsampling sensitive to depth noise; too large a sigma causes excessive blurring [G].
*   **When to use:** When rendering volumetric clouds (or similar translucent volumes) where blue noise is visible at high resolutions, but full-resolution raymarching is too expensive [S1, unofficial].
*   **When not to use:** When depth information is unavailable or unreliable, or when the volume has sharp depth discontinuities that cannot be captured by the low-res depth buffer [G].

## Background
The technique is documented in the context of procedural volumetric cloud rendering [S1, unofficial]. It addresses the trade-off between visual quality (blue noise) and performance in real-time cloud rendering [S1, unofficial]. The method builds on standard bilinear downsampling and upsampling techniques, adding a depth-aware weighting mechanism to preserve sharpness and reduce noise [S1, unofficial].

## Sources
[S1] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
