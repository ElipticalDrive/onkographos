# Temporal Upsampling and Dithering

Also called: Dithering, Temporal Upsampling

## Overview
Temporal Upsampling and Dithering is a rendering technique that combines spatial resolution reduction with temporal accumulation to mitigate sampling artifacts in volume rendering, specifically addressing slicing artifacts caused by insufficient sampling steps [S1, unofficial]. It distinguishes itself from standard adaptive sampling methods by using blue noise dithering to break up coherent aliasing patterns and temporal upsampling to reduce the noise introduced by that dithering, rather than relying solely on spatial supersampling or static resolution scaling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Low-resolution color buffer (e.g., 1/2x1/2 resolution), depth buffer, and previous frame's accumulated buffer.
- Output: High-resolution color buffer (full resolution) blended into the Scene Color Render Target.

**Data Structures:**
- **Blue Noise Texture:** A 2D texture containing precomputed blue noise values, used for spatial dithering.
- **History Buffer:** A buffer storing accumulated color values from previous frames, reprojected using pixel velocity.
- **Pixel Velocity Buffer:** A buffer storing the movement vector of each pixel between frames, used for reprojection.

**Steps:**
1. **Resolution Reduction:** Render the volume at a lower resolution (e.g., 1/2x1/2) to reduce computational cost [S1, unofficial].
2. **Blue Noise Dithering:** Apply blue noise dithering to the sampled values to break up slicing artifacts caused by insufficient sampling steps [S1, unofficial]. This introduces high-frequency noise but eliminates coherent aliasing.
3. **Temporal Accumulation:** Use temporal upsampling to blend the current low-resolution frame with the history buffer. The history buffer is reprojected using pixel velocity to align with the current frame's geometry [S1, unofficial].
4. **Upsampling:** Upsample the accumulated low-resolution result to full resolution. This can be done via bilinear filtering or more advanced techniques, leveraging the temporal stability provided by the accumulation.
5. **Blending:** Blend the upsampled result into the final Scene Color Render Target [S1, unofficial].

**Formulas:**
- **Dithered Sample Value:** $C_{dithered} = C_{sample} + D_{blue\_noise} \times \alpha$, where $C_{sample}$ is the raw sample, $D_{blue\_noise}$ is the blue noise value, and $\alpha$ is a scaling factor for dithering intensity.
- **Temporal Blend:** $C_{accumulated} = w \cdot C_{current} + (1 - w) \cdot C_{history\_reprojected}$, where $w$ is the temporal weight (often based on confidence or time delta).

**Parameters:**
- **Resolution Scale:** Typically 1/2x1/2 for high-end hardware; lower resolutions may be used for mid-low end [S1, unofficial].
- **Dithering Intensity ($\alpha$):** Not specified in sources.
- **Temporal Weight ($w$):** Not specified in sources.

**Edge Cases and Pitfalls:**
- **Ghosting:** If pixel velocity is inaccurate, ghosting artifacts may appear. This is mitigated by using robust velocity estimation and potentially anti-smear techniques (see [Temporal Anti-Aliasing with Anti-Smear](3614-temporal-anti-anti-smear.md)).
- **Noise Amplification:** Excessive dithering can introduce visible noise, which temporal upsampling must effectively smooth out.
- **Boundary Artifacts:** At the edges of the screen or volume boundaries, reprojection may fail, requiring clamping or alternative handling.

## Subtopics

### [Temporal Anti-Aliasing with Anti-Smear](3614-temporal-anti-aliasing-with-anti-smear.md)
Adds depth-based rejection to the accumulation process to prevent smearing artifacts, building on this method by enhancing the temporal stability.

## Operating characteristics
- **Time Complexity:** Reduced by rendering at lower resolution (e.g., 1/4th the pixels for 1/2x1/2). Temporal accumulation adds overhead for reprojection and blending.
- **Memory Complexity:** Requires additional buffers for history, velocity, and blue noise texture.
- **Performance:** Effective on high-end hardware where resolution reduction is feasible. On mid-low end, frame-by-frame processing (similar to RDR2) may be used, which is simpler but may have lower temporal stability [S1, unofficial].
- **Strengths:** Reduces slicing artifacts and noise while maintaining performance.
- **Limitations:** Can introduce ghosting if velocity estimation is poor. Dithering noise requires effective temporal smoothing.
- **When to Use:** When volume rendering suffers from slicing artifacts due to limited sampling steps and performance is a concern.
- **When Not to Use:** When high temporal stability is critical and velocity estimation is unreliable, or when hardware cannot support the additional buffer overhead.

## Background
This technique is discussed in the context of real-time volume cloud rendering, particularly in game engines like Unity's HDRP. It is inspired by techniques used in games like Red Dead Redemption 2 for handling atmospheric effects [S1, unofficial]. The combination of blue noise dithering and temporal upsampling is a practical approach to balancing visual quality and performance in real-time volume rendering.

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
