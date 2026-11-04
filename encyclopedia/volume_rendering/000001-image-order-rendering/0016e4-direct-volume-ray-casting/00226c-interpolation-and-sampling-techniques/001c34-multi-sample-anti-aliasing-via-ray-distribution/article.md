# Multi-Sample Anti-Aliasing via Ray Distribution

Also called: Multi-ray AA, Multi-sample AA, Ray Distribution AA, MSAA via Ray Distribution

## Overview
Multi-Sample Anti-Aliasing (MSAA) via Ray Distribution reduces jagged edges in volume rendering by casting multiple rays per pixel, distributed within the pixel area, and averaging the results [S1, S2, S3, unofficial]. Unlike single-ray casting, which suffers from aliasing artifacts at silhouette edges due to coarse per-pixel sampling granularity [S3, unofficial], this method samples the scene at a higher effective resolution. It is distinct from [Supersampling](3493-supersampling.md) in its specific implementation of fixed offset patterns derived from fragment coordinates, and from [Epipolar Sampling](3899-epipolar-sampling.md) which exploits geometric constraints along epipolar lines.

## How it works
**Inputs and Outputs:**
*   **Input:** Pixel coordinates (e.g., `gl_FragCoord`), an integer uniform `AA` selecting the sampling mode (1, 2, 3, or 4), and a rendering function `render(uv)` that performs the volume integration for a given normalized UV offset within the pixel.
*   **Output:** A single averaged color value for the pixel.

**Data Structures:**
No complex data structures are required. The method relies on scalar arithmetic and fixed offset vectors.

**Parameters:**
*   `AA`: Integer uniform selector. Values: 1, 2, 3, 4. Default is 4 if not explicitly set to 1, 2, or 3 [S2, unofficial].

**Steps:**
1.  **Select Mode:** Determine the number of samples $N$ based on the `AA` uniform.
2.  **Generate Offsets:** Calculate 2D UV offsets $(\delta u, \delta v)$ for each of the $N$ rays relative to the pixel center. The offsets are normalized to the range $[-0.5, 0.5]$ or $[0, 1]$ depending on the coordinate system of `getUV`. The provided sources use offsets that are added to a base UV, implying they are relative to the pixel center or corner. The specific formulas below assume the offsets are added to the pixel's base UV coordinate.
3.  **Cast Rays:** For each sample $i \in [0, N-1]$, compute the ray origin/direction using the calculated offset and call `render` to obtain color $C_i$.
4.  **Average:** Sum the colors and divide by $N$.

**Formulas and Offsets:**
Let $bxy = (x + y) \pmod 2$, where $x$ and $y$ are the integer pixel coordinates (e.g., from `gl_FragCoord`). Let $nbxy = 1 - bxy$.

*   **AAx1 ($N=1$):**
    *   Offset: $(0, 0)$.
    *   Formula: $C = \text{render}(\text{getUV}(0, 0))$.

*   **AAx2 ($N=2$):**
    *   Offsets: $(0.33 \cdot nbxy, 0)$ and $(0.33 \cdot bxy, 0.66)$.
    *   Formula: $C = \frac{1}{2} \left( \text{render}(\text{getUV}(0.33 \cdot nbxy, 0)) + \text{render}(\text{getUV}(0.33 \cdot bxy, 0.66)) \right)$.

*   **AAx3 ($N=3$):**
    *   Offsets: $(0.66 \cdot nbxy, 0)$, $(0.66 \cdot bxy, 0.66)$, and $(0.33, 0.33)$.
    *   Formula: $C = \frac{1}{3} \left( \text{render}(\text{getUV}(0.66 \cdot nbxy, 0)) + \text{render}(\text{getUV}(0.66 \cdot bxy, 0.66)) + \text{render}(\text{getUV}(0.33, 0.33)) \right)$.

*   **AAx4 ($N=4$):**
    *   Define vector $e = (0.125, -0.125, 0.375, -0.375)$.
    *   Offsets: $(e_x, e_z)$, $(e_y, e_w)$, $(e_w, e_x)$, $(e_z, e_y)$.
    *   Explicit Offsets: $(0.125, 0.375)$, $(-0.125, -0.375)$, $(-0.375, 0.125)$, $(0.375, -0.125)$.
    *   Formula: $C = \frac{1}{4} \left( \text{render}(\text{getUV}(0.125, 0.375)) + \text{render}(\text{getUV}(-0.125, -0.375)) + \text{render}(\text{getUV}(-0.375, 0.125)) + \text{render}(\text{getUV}(0.375, -0.125)) \right)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_aa_aa_mode(AA_mode, pixel_x, pixel_y, get_uv, render):
    """
    Renders a pixel using Multi-Sample Anti-Aliasing via Ray Distribution.
    
    Args:
        AA_mode (int): 1, 2, 3, or 4.
        pixel_x (int): X coordinate of the fragment/pixel.
        pixel_y (int): Y coordinate of the fragment/pixel.
        get_uv (function): Returns the base UV coordinate for the pixel center.
        render (function): Takes a UV offset (vec2) and returns a color (vec3).
        
    Returns:
        vec3: The averaged color for the pixel.
    """
    if AA_mode == 1:
        # Single ray at center
        uv_offset = vec2(0.0, 0.0)
        return render(get_uv(uv_offset))
    
    elif AA_mode == 2:
        # Two rays
        bxy = (pixel_x + pixel_y) & 1
        nbxy = 1.0 - bxy
        
        uv1 = get_uv(vec2(0.33 * nbxy, 0.0))
        uv2 = get_uv(vec2(0.33 * bxy, 0.66))
        
        col1 = render(uv1)
        col2 = render(uv2)
        
        return (col1 + col2) / 2.0
    
    elif AA_mode == 3:
        # Three rays
        bxy = (pixel_x + pixel_y) & 1
        nbxy = 1.0 - bxy
        
        uv1 = get_uv(vec2(0.66 * nbxy, 0.0))
        uv2 = get_uv(vec2(0.66 * bxy, 0.66))
        uv3 = get_uv(vec2(0.33, 0.33))
        
        col1 = render(uv1)
        col2 = render(uv2)
        col3 = render(uv3)
        
        return (col1 + col2 + col3) / 3.0
    
    elif AA_mode == 4:
        # Four rays
        # e = vec4(0.125, -0.125, 0.375, -0.375)
        # Offsets: (e.x, e.z), (e.y, e.w), (e.w, e.x), (e.z, e.y)
        e_x = 0.125
        e_y = -0.125
        e_z = 0.375
        e_w = -0.375
        
        uv1 = get_uv(vec2(e_x, e_z))
        uv2 = get_uv(vec2(e_y, e_w))
        uv3 = get_uv(vec2(e_w, e_x))
        uv4 = get_uv(vec2(e_z, e_y))
        
        col1 = render(uv1)
        col2 = render(uv2)
        col3 = render(uv3)
        col4 = render(uv4)
        
        return (col1 + col2 + col3 + col4) / 4.0
    
    else:
        # Default to AAx4 if mode is not 1, 2, or 3
        return render_aa_aa_mode(4, pixel_x, pixel_y, get_uv, render)
```

**Edge Cases and Pitfalls:**
*   **Performance:** This technique significantly reduces framerate due to the increased ray count (2x to 4x) [S1, unofficial].
*   **Optimization:** An optimization suggested is to only perform AA if the iteration count (or some metric indicating object presence/edge) is above average, as this likely indicates an object edge where aliasing is most visible [S1, unofficial].
*   **Coordinate System:** The offsets provided in the sources [S2, unofficial] are specific to the implementation's `getUV` function. If `getUV` expects normalized coordinates in $[0,1]$ relative to the pixel, the offsets must be interpreted accordingly. The source code uses `vec2(0.33 * nbxy, 0.)` directly, implying these are additive offsets to a base UV.

## Subtopics

### Supersampling
The general concept of sampling at higher resolution. MSAA via Ray Distribution is a specific implementation of supersampling for ray marching [S3, unofficial]. [Link](3493-supersampling.md)

### Sampling and Jittering
MSAA uses deterministic offsets based on pixel coordinates, whereas jittering uses random offsets. [Link](2516-sampling-and-jittering.md)

### Epipolar Sampling
Uses geometric constraints to reduce sampling, whereas MSAA uses fixed pixel-local offsets. [Link](3899-epipolar-sampling.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples (1, 2, 3, or 4). The rendering cost increases linearly with $N$.
*   **Memory Complexity:** $O(1)$ additional memory per pixel, as only the accumulated color sum is needed.
*   **Performance:** Significantly reduces framerate [S1, unofficial]. The performance hit is proportional to the number of rays.
*   **Strengths:** Effectively removes jagged edges and aliasing artifacts, particularly along silhouette edges where intensity changes abruptly [S1, S3, unofficial].
*   **Limitations:** High computational cost. Fixed patterns may not adapt to scene complexity, leading to wasted samples in empty space or insufficient samples in complex regions.
*   **When to Use:** When visual quality is prioritized over performance, and aliasing is a noticeable issue.
*   **When Not to Use:** When performance is critical, or when the scene is simple enough that single-ray casting is sufficient.

## Background
The technique is described in the context of raymarching distance fields and volume rendering. It is presented as a straightforward method to improve image quality by distributing rays within a pixel [S1, unofficial]. The specific offset patterns for AAx2, AAx3, and AAx4 are derived from shader code examples in a GitHub repository [S2, unofficial]. The theoretical basis for supersampling to combat aliasing artifacts caused by coarse sampling granularity is discussed in academic literature on volume ray casting [S3, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S3] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
