# Edge-Preserving Kawase Blur
Also called: Edge-Aware Kawase Blur, Kawase Blur with Blue Noise Jittering
## Overview
Edge-Preserving Kawase Blur is a post-processing technique designed to remove dither patterns introduced by blue noise jittering in volumetric rendering while preserving cloud edge details [S1, unofficial]. It sets itself apart from its sibling [Median Filter Denoising](8243-median-filter-denoising.md) by blurring high-frequency jitter rather than removing isolated outliers, making it suitable for smoothing dither patterns without the discrete neighborhood constraints of median filtering.
## How it works
**Inputs and Outputs:**
- Input: A rendered frame buffer containing jittered volume samples.
- Input: A depth buffer (or luminance buffer) for edge detection.
- Input: Blue noise texture for jittering sample positions.
- Output: A smoothed frame buffer with preserved edges.

**Data Structures:**
- `FrameBuffer`: 2D array of pixel colors (RGB or RGBA).
- `DepthBuffer`: 2D array of depth values corresponding to the FrameBuffer.
- `BlueNoiseTexture`: 2D texture containing precomputed blue noise values.

**Parameters:**
- `blurRadius`: The radius of the blur kernel (e.g., 1, 2, 3).
- `edgeThreshold`: The luminance/depth difference threshold for edge preservation.
- `extraBlurFactor`: A multiplier for blurring distant surfaces.

**Steps:**
1. **Initialize Output:** Create an output frame buffer of the same dimensions as the input.
2. **Iterate Pixels:** For each pixel $(x, y)$ in the input frame:
   a. Get the center color $C_c$ and depth $D_c$ at $(x, y)$.
   b. Initialize a weighted sum $W_{sum} = 0$ and a weighted color sum $C_{sum} = (0, 0, 0)$.
   c. **Kawase Passes:** For each pass $p$ from 1 to `blurRadius`:
      i. Define the 4 diagonal offsets for the current pass: $(\pm p, \pm p)$.
      ii. For each offset $(dx, dy)$:
          1. Calculate the sample position $(x_s, y_s) = (x + dx, y + dy)$.
          2. Sample the blue noise value $B$ at $(x_s, y_s)$.
          3. Jitter the sample position: $(x_j, y_j) = (x_s + B \cdot \epsilon, y_s + B \cdot \epsilon)$, where $\epsilon$ is a small jitter magnitude [G].
          4. Clamp $(x_j, y_j)$ to the image boundaries.
          5. Get the sample color $C_s$ and depth $D_s$ at $(x_j, y_j)$.
          6. Calculate the edge weight $w$:
             $$ w = \max(0, 1 - \frac{|C_s - C_c| + |D_s - D_c|}{\text{edgeThreshold}}) $$
             [S1, unofficial]
          7. Apply extra blur for distant surfaces:
             $$ w_{final} = w \cdot (1 + \text{extraBlurFactor} \cdot D_c) $$
             [S1, unofficial]
          8. Update sums:
             $$ W_{sum} += w_{final} $$
             $$ C_{sum} += w_{final} \cdot C_s $$
   d. **Normalize:** If $W_{sum} > 0$, set the output color at $(x, y)$ to $C_{sum} / W_{sum}$. Otherwise, keep $C_c$.
3. **Output:** Return the output frame buffer.

**Edge Cases and Pitfalls:**
- **Boundaries:** Clamp sample positions to image boundaries to avoid out-of-bounds access.
- **Precision:** Use floating-point arithmetic for color and depth calculations to avoid precision errors.
- **Degenerate Input:** If the depth buffer is unavailable, use luminance only for edge detection.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_preserving_kawase_blur(input_frame, depth_buffer, blue_noise_texture, blur_radius, edge_threshold, extra_blur_factor):
    height, width = input_frame.shape[:2]
    output_frame = np.zeros_like(input_frame)
    
    for y in range(height):
        for x in range(width):
            center_color = input_frame[y, x]
            center_depth = depth_buffer[y, x]
            
            weighted_sum_color = np.array([0.0, 0.0, 0.0])
            weighted_sum = 0.0
            
            for pass_idx in range(1, blur_radius + 1):
                offsets = [(pass_idx, pass_idx), (pass_idx, -pass_idx), (-pass_idx, pass_idx), (-pass_idx, -pass_idx)]
                
                for dx, dy in offsets:
                    sample_x = x + dx
                    sample_y = y + dy
                    
                    # Jitter using blue noise
                    noise_val = blue_noise_texture[sample_y % height, sample_x % width]
                    jittered_x = sample_x + noise_val * 0.5  # [G]
                    jittered_y = sample_y + noise_val * 0.5  # [G]
                    
                    # Clamp to boundaries
                    jittered_x = max(0, min(width - 1, jittered_x))
                    jittered_y = max(0, min(height - 1, jittered_y))
                    
                    sample_color = input_frame[int(jittered_y), int(jittered_x)]
                    sample_depth = depth_buffer[int(jittered_y), int(jittered_x)]
                    
                    # Edge weight calculation
                    color_diff = np.abs(sample_color - center_color).sum()
                    depth_diff = abs(sample_depth - center_depth)
                    weight = max(0.0, 1.0 - (color_diff + depth_diff) / edge_threshold)
                    
                    # Extra blur for distant surfaces
                    final_weight = weight * (1.0 + extra_blur_factor * center_depth)
                    
                    weighted_sum_color += final_weight * sample_color
                    weighted_sum += final_weight
            
            if weighted_sum > 0:
                output_frame[y, x] = weighted_sum_color / weighted_sum
            else:
                output_frame[y, x] = center_color
                
    return output_frame
```
## Subtopics

### [Median Filter Denoising](8243-median-filter-denoising.md)
A post-processing technique used to reduce 'Salt and Pepper' noise caused by unpredictable precision errors in raycasting that result in wrong or missing pixel colors. It sets apart from its sibling [Edge-Preserving Kawase Blur](7026-edge-preserving-kawase-blur.md) by operating on a fixed 3x3 neighborhood to remove isolated outliers (single-pixel errors) rather than blurring dither patterns, making it suitable for correcting discrete precision failures rather than high-frequency jitter.

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: $O(N \cdot R^2)$, where $N$ is the number of pixels and $R$ is the blur radius. Each pixel requires $4 \cdot R$ samples.
- Memory Complexity: $O(N)$ for the input and output frame buffers, plus $O(N)$ for the depth buffer and blue noise texture.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Performance scales linearly with the number of pixels and the square of the blur radius.

**Strengths and Limitations:**
- Strengths: Cheap and effective blur; preserves cloud edge details; removes dither patterns introduced by blue noise jittering.
- Limitations: Can introduce banding patterns if not jittered; may not be suitable for hard surfaces where temporal accumulation is preferred.

**When to Use:**
- Use when rendering volumetric clouds with blue noise jittering and edge preservation is desired.

**When Not to Use:**
- Do not use when rendering hard surfaces where temporal accumulation (TAA) is more effective.
## Background
**Origin:**
- The technique is described in a forum post by SimonDev on the three.js forum [S1, unofficial].

**History and Development:**
- The technique builds on the Kawase blur method, which is a type of box-blur known for its efficiency but prone to banding patterns. Blue noise jittering is used to mitigate these banding patterns.

**Where it is Used:**
- Used in efficient volumetric cloud rendering to smooth out noise and improve perceived resolution without extra raymarching steps.
## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
