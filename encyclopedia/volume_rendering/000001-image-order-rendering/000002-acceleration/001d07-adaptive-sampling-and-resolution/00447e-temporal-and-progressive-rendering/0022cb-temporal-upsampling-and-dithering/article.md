# Temporal Upsampling and Dithering

Also called: Temporal Upsampling

## Overview
Temporal Upsampling and Dithering is a technique that combines spatial dithering to mitigate sampling artifacts with temporal accumulation to reduce noise, primarily used in real-time volume rendering (such as volumetric clouds) where computational budget limits per-frame sampling density [S1, unofficial]. It distinguishes itself from its siblings by addressing the specific trade-off between spatial aliasing (slicing artifacts from coarse steps) and temporal noise (stochastic variation from dithering), whereas Checkerboard Progressive Rendering focuses on spatial resolution adaptation and Multi-Resolution Rendering for Clouds focuses on hierarchical data structures [4836-checkerboard-progressive-rendering.md][7260-multi-resolution-rendering-for-clouds.md] [G].

## How it works
**Inputs and Outputs:**
*   **Input:** Current frame's color buffer (often at reduced resolution or with coarse sampling), previous frame's accumulated color buffer, and depth/velocity buffers for reprojection [G].
*   **Output:** High-resolution, denoised color frame [G].

**Data Structures:**
*   **History Buffer:** A texture storing the accumulated color values from previous frames [G].
*   **Depth/Velocity Buffers:** Used to reproject the history buffer into the current frame's pixel coordinates [G].

**Steps:**
1.  **Blue Noise Dithering:** Apply blue noise dithering to the sampling positions or step sizes during the volume integration pass. This breaks up regular aliasing patterns (slicing artifacts) caused by insufficient sampling steps [S1, unofficial].
2.  **Temporal Accumulation:** Reproject the previous frame's color into the current frame's coordinate space using depth and velocity information. Blend the reprojected color with the current frame's dithered result. This reduces the high-frequency noise introduced by the dithering process [S1, unofficial][G].
3.  **Upsampling:** If the rendering was performed at a lower resolution (e.g., 1/2x1/2), upsample the final accumulated result to the full screen resolution [S1, unofficial].

**Formulas:**
*   **Reprojection:** $C_{reprojected}(x, y) = C_{history}(x - v_x \cdot z, y - v_y \cdot z)$, where $(v_x, v_y)$ is the pixel velocity and $z$ is the depth [G].
*   **Blending:** $C_{final}(x, y) = \alpha \cdot C_{current}(x, y) + (1 - \alpha) \cdot C_{reprojected}(x, y)$, where $\alpha$ is a blending factor (often related to temporal stability or motion speed) [G].

**Parameters:**
*   **Dither Pattern:** Blue noise texture [G].
*   **Blending Factor $\alpha$:** Typically small (e.g., 0.1-0.3) to favor history, but adjusted based on motion to prevent ghosting [G].

**Edge Cases and Pitfalls:**
*   **Ghosting:** If depth or velocity is inaccurate, the reprojected history will misalign, causing ghosting artifacts. Anti-smear techniques (discarding history pixels with large depth differences) may be required [3614-temporal-anti-aliasing-with-anti-smear.md] [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_upsampling_and_dithering(current_color, history_buffer, depth_buffer, velocity_buffer, blue_noise_texture, alpha):
    H, W = len(velocity_buffer), len(velocity_buffer[0])
    
    # Step 1: Apply blue noise dithering to sampling positions to break slicing artifacts [S1, unofficial]
    dithered_color = [[current_color[y][x] + blue_noise_texture[y][x] * 0.05 for x in range(W)] for y in range(H)]

    # Step 2: Reproject previous frame's color using depth and velocity [G]
    reprojected_color = [[0.0]*3 for _ in range(H)]
    for y in range(H):
        for x in range(W):
            vel_x = velocity_buffer[y][x][0]
            vel_y = velocity_buffer[y][x][1]
            depth = depth_buffer[y][x]
            # Screen space offset: velocity * depth
            offset_x = vel_x * depth
            offset_y = vel_y * depth
            src_x = int(clamp(x - offset_x, 0, W - 1))
            src_y = int(clamp(y - offset_y, 0, H - 1))
            reprojected_color[y][x] = history_buffer[src_y][src_x]

    # Step 3: Blend current dithered result with reprojected history to reduce noise [S1, unofficial][G]
    blended_color = [[mix(dithered_color[y][x], reprojected_color[y][x], alpha) for x in range(W)] for y in range(H)]

    # Step 4: Upsample if rendered at lower resolution (e.g., 1/2x1/2) [S1, unofficial]
    final_color = upscale_bilinear(blended_color)

    # Step 5: Update history buffer for next frame
    history_buffer = final_color

    return final_color

def mix(a, b, t):
    return [a[i] * (1.0 - t) + b[i] * t for i in range(len(a))]

def clamp(v, mn, mx):
    return max(mn, min(mx, v))

def upscale_bilinear(tex):
    h, w = len(tex), len(tex[0])
    out_h, out_w = h * 2, w * 2
    out = [[0.0]*3 for _ in range(out_h)]
    for y in range(out_h):
        for x in range(out_w):
            src_y = y / 2.0
            src_x = x / 2.0
            y0, x0 = int(src_y), int(src_x)
            y1, x1 = min(y0 + 1, h - 1), min(x0 + 1, w - 1)
            dy, dx = src_y - y0, src_x - x0
            p00 = tex[y0][x0]
            p10 = tex[y1][x0]
            p01 = tex[y0][x1]
            p11 = tex[y1][x1]
            out[y][x] = [(1-dy)*(1-dx)*p00[i] + dy*(1-dx)*p10[i] + (1-dy)*dx*p01[i] + dy*dx*p11[i] for i in range(3)]
    return out
```

## Subtopics

### [Temporal Anti-Aliasing with Anti-Smear](3614-temporal-anti-aliasing-with-anti-smear.md)
Temporal Anti-Aliasing with Anti-Smear is a post-processing technique that blends the current frame's color with an accumulated history buffer using pixel velocity for reprojection, with an optional anti-smear feature that discards accumulated pixels with large depth differences to prevent ghosting. It distinguishes itself from standard TAA by explicitly checking depth discontinuities during reprojection to avoid smearing artifacts.

### Jittered Dithering Distributions
An algorithm for quickly generating a pseudo-uniform distribution of points based on a regular 2D grid, originally proposed by Mitchell [S2, unofficial]. Create a regular 2D grid with sixteen times as many grid cells as the desired number of points [S2, unofficial]. Visit each cell in turn in a left-to-right and top-to-bottom fashion [S2, unofficial]. Calculate potential $D_{i,j}$ using the function: $D_{i,j} = (4D_{i-1,j} + D_{i-1,j-1} + 2D_{i,j-1} + D_{i+1,j-1}) / 8 + R$, where $R = \text{random}()$ [S2, unofficial]. If $D_{i,j}$ is greater than the threshold of one-half, select a jittered point at cell $(i,j)$ and reduce the $D_{i,j}$ potential by one [S2, unofficial]. Thus, about 1/16th of the data points are selected [S2, unofficial]. Modifying the threshold or random increment to generate weighted point densities produces regular patterns in the splats and causes the number of dots deposited to vary widely from the expected number [S2, unofficial].

### Spatial Dithering for Color Perception
A method using spatial dithering to create the perception of more than 3-bit color from single-bit-depth SLM frames [S3]. The system uses a 3-SLM projection engine where each SLM has 1-bit depth [S3]. Proprietary rasterization and half-toning/dithering methods process the OpenGL pipeline data before projection [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, dominated by the reprojection and blending steps [G].
*   **Memory Complexity:** $O(N)$ for storing the history buffer, depth buffer, and velocity buffer [G].
*   **Performance:** Significantly reduces the need for high per-frame sampling rates, allowing for real-time rendering of complex volumes like clouds [S1, unofficial].
*   **Strengths:** Effective at reducing noise and aliasing artifacts with minimal additional computational cost per frame [G].
*   **Limitations:** Can introduce ghosting or smearing artifacts if motion estimation is inaccurate or if there are rapid scene changes [G]. Requires careful tuning of the blending factor and anti-smear thresholds [G].

## Background
The technique is commonly used in real-time graphics engines for rendering volumetric effects such as clouds and fog, where high sampling rates are computationally prohibitive [S1, unofficial]. It builds on the principles of Temporal Anti-Aliasing (TAA) and blue noise dithering, adapting them to the specific challenges of volume rendering [G].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] Multivariate volume rendering, https://www.osti.gov/biblio/495676, fetched 2026-10-26 (unofficial)
[S3] 100-million-voxel volumetric display, https://doi.org/10.1117/12.480930, fetched 2026-11-01
