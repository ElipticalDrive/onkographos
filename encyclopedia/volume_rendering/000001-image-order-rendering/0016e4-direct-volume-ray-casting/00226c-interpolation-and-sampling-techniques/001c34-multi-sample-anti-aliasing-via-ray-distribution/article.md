# Multi-Sample Anti-Aliasing via Ray Distribution

Also called: Multi-ray AA, Multi-sample AA, Ray Distribution AA, MSAA via Ray Distribution

## Overview
Multi-Sample Anti-Aliasing via Ray Distribution reduces jagged edges and aliasing artifacts in volume rendering by casting multiple rays per pixel, distributed within the pixel area, and averaging the results [S1, unofficial][S2, unofficial][S3, unofficial]. It sets itself apart from standard single-ray casting by explicitly addressing the coarse granularity of per-pixel sampling, which causes abrupt intensity changes along object silhouettes [S3, unofficial]. Compared to [Supersampling](3493-supersampling.md), which generally implies a uniform grid increase in resolution, this method often employs specific offset patterns (such as parity-based or fixed vector offsets) to distribute samples efficiently within the pixel footprint [S2, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A uniform integer `AA` (1, 2, 3, or 4) selecting the sampling mode; the fragment coordinates `gl_FragCoord` (or equivalent pixel coordinates); a rendering function `render(uv)` that performs the volume ray casting for a given normalized UV offset `uv` within the pixel.
*   **Output:** A single color value $C_{pixel}$ for the pixel, computed as the average of the colors returned by the distributed rays.

**Data Structures:**
No complex data structures are required. The method relies on scalar arithmetic and fixed offset vectors.

**Parameters:**
*   `AA`: Integer uniform. Selects the number of samples per pixel. Values: 1, 2, 3, or 4. Default is typically 4 if not explicitly set to 1, 2, or 3 [S2, unofficial].

**Steps:**
1.  Select the anti-aliasing function based on the `AA` parameter.
2.  Calculate sample offsets within the unit square $[0, 1] \times [0, 1]$ representing the pixel area.
3.  For each sample $i$, compute the UV coordinate: $\mathbf{uv}_i = \mathbf{uv}_{center} + \mathbf{offset}_i$.
4.  Call the ray casting routine for each UV: $C_i = \text{render}(\mathbf{uv}_i)$.
5.  Average the results: $C_{pixel} = \frac{1}{N} \sum_{i=1}^{N} C_i$, where $N$ is the number of samples (1, 2, 3, or 4).

**Formulas and Offsets:**
The offsets are defined relative to the pixel center (normalized to 0,0 in the provided implementations, or applied as deltas to the pixel's base UV). The following patterns are derived from the source implementations [S2, unofficial]:

*   **AAx1 (1 sample):**
    *   Offset: $(0, 0)$.
    *   Formula: $C_{pixel} = \text{render}(\mathbf{uv}_{center})$.

*   **AAx2 (2 samples):**
    *   Depends on the parity of the fragment coordinates. Let $b_{xy} = \lfloor x + y \rfloor \pmod 2$. Let $n_{bxy} = 1 - b_{xy}$.
    *   Offsets: $(0.33 \cdot n_{bxy}, 0)$ and $(0.33 \cdot b_{xy}, 0.66)$.
    *   Formula: $C_{pixel} = \frac{\text{render}(\mathbf{uv}_{center} + (0.33 \cdot n_{bxy}, 0)) + \text{render}(\mathbf{uv}_{center} + (0.33 \cdot b_{xy}, 0.66))}{2}$.

*   **AAx3 (3 samples):**
    *   Uses the same parity logic as AAx2 for the first two samples, plus a fixed third sample.
    *   Offsets: $(0.66 \cdot n_{bxy}, 0)$, $(0.66 \cdot b_{xy}, 0.66)$, and $(0.33, 0.33)$.
    *   Formula: $C_{pixel} = \frac{\text{render}(\mathbf{uv}_{center} + (0.66 \cdot n_{bxy}, 0)) + \text{render}(\mathbf{uv}_{center} + (0.66 \cdot b_{xy}, 0.66)) + \text{render}(\mathbf{uv}_{center} + (0.33, 0.33))}{3}$.

*   **AAx4 (4 samples):**
    *   Uses a fixed vector $e = (0.125, -0.125, 0.375, -0.375)$.
    *   Offsets are formed by combinations of $e$: $(e_x, e_z)$, $(e_y, e_w)$, $(e_w, e_x)$, $(e_z, e_y)$.
    *   Explicit Offsets: $(0.125, 0.375)$, $(-0.125, -0.375)$, $(-0.375, 0.125)$, $(0.375, -0.125)$.
    *   Formula: $C_{pixel} = \frac{\sum \text{render}(\mathbf{uv}_{center} + \mathbf{offset}_i)}{4}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_aa(uv_center, aa_mode):
    """
    Renders a pixel using Multi-Sample Anti-Aliasing via Ray Distribution.
    
    Args:
        uv_center (vec2): The normalized center UV of the pixel.
        aa_mode (int): Anti-aliasing mode (1, 2, 3, or 4).
        
    Returns:
        vec3: The averaged color of the pixel.
    """
    if aa_mode == 1:
        return render(uv_center)
    
    elif aa_mode == 2:
        # Calculate parity based on integer fragment coordinates
        # Note: In a real shader, this uses gl_FragCoord. 
        # Here we assume access to integer pixel coordinates (px, py).
        bxy = (px + py) & 1
        nbxy = 1 - bxy
        
        offset1 = vec2(0.33 * nbxy, 0.0)
        offset2 = vec2(0.33 * bxy, 0.66)
        
        col1 = render(uv_center + offset1)
        col2 = render(uv_center + offset2)
        
        return (col1 + col2) / 2.0
    
    elif aa_mode == 3:
        bxy = (px + py) & 1
        nbxy = 1 - bxy
        
        offset1 = vec2(0.66 * nbxy, 0.0)
        offset2 = vec2(0.66 * bxy, 0.66)
        offset3 = vec2(0.33, 0.33)
        
        col1 = render(uv_center + offset1)
        col2 = render(uv_center + offset2)
        col3 = render(uv_center + offset3)
        
        return (col1 + col2 + col3) / 3.0
    
    elif aa_mode == 4:
        # Vector e components
        ex = 0.125
        ey = -0.125
        ez = 0.375
        ew = -0.375
        
        # Offsets derived from combinations: xz, yw, wx, zy
        offset1 = vec2(ex, ez)
        offset2 = vec2(ey, ew)
        offset3 = vec2(ew, ex)
        offset4 = vec2(ez, ey)
        
        col1 = render(uv_center + offset1)
        col2 = render(uv_center + offset2)
        col3 = render(uv_center + offset3)
        col4 = render(uv_center + offset4)
        
        return (col1 + col2 + col3 + col4) / 4.0
    
    else:
        # Default to AAx4 if mode is not 1, 2, or 3
        return render_aa(uv_center, 4)
```

**Edge Cases and Pitfalls:**
*   **Performance:** This technique significantly reduces framerate due to the increased ray count (1x to 4x) [S1, unofficial].
*   **Optimization:** An optimization suggested is to only perform AA if the ray marching iteration count is above average, as this likely indicates the presence of an object edge where aliasing is most visible [S1, unofficial].
*   **Boundary Conditions:** The offsets are defined within a specific range. Implementers must ensure that `render()` correctly handles UV coordinates that might fall slightly outside $[0,1]$ if `uv_center` is near the edge of the screen, though the specific offsets provided generally stay within reasonable bounds relative to the center.

## Subtopics

### Supersampling
A broader category of sampling the scene at a higher resolution than displayed. Multi-sample AA via Ray Distribution is a specific implementation of supersampling tailored for ray marching. See [Supersampling](3493-supersampling.md).

### Sampling and Jittering
General techniques for distributing samples. The parity-based approach in AAx2/AAx3 is a form of deterministic jittering, while other distributions like Hammersley or Multi-Jittered are mentioned as alternatives for better uniformity. See [Sampling and Jittering](2516-sampling-and-jittering.md).

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of samples (1-4) and $R$ is the average number of ray marching steps per ray. The cost scales linearly with the AA mode.
*   **Memory Complexity:** $O(1)$ additional memory per pixel, as no intermediate buffers are required beyond the accumulated color.
*   **Performance:** Significantly impacts framerate. A 4x mode quadruples the computational load of the ray casting step [S1, unofficial].
*   **Strengths:** Effectively removes jagged edges and reduces aliasing artifacts caused by abrupt intensity changes at silhouettes [S3, unofficial]. Simple to implement.
*   **Limitations:** High computational cost. The specific offset patterns (especially parity-based) may not provide optimal spectral properties compared to more advanced distributions like Hammersley or Multi-Jittered [S3, unofficial].
*   **When to use:** When visual quality is prioritized over performance, or when edges are prominent.
*   **When not to use:** In real-time applications with strict performance budgets, or in scenes with no high-frequency details (where AAx1 is sufficient).

## Background
This technique originates from practical implementations of ray marching for distance fields and volume rendering. It is documented in early online tutorials and code repositories focusing on real-time ray marching [S1, unofficial][S2, unofficial]. The specific offset patterns (AAx2, AAx3, AAx4) are derived from shader code found in GitHub repositories [S2, unofficial]. The theoretical underpinning relates to supersampling, a well-known anti-aliasing method described in academic literature on volume ray casting [S3, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S3] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
