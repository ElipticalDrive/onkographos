# Capture CoC

Also called: fragCaptureCoc

## Overview
Capture CoC calculates the unsigned magnitude of the Circle of Confusion (CoC) for a given pixel depth. It serves as the foundational step in depth-of-field post-processing by determining how much a pixel should blur based on its distance from the focal plane. It differs from [Foreground CoC Blur](21024-foreground-coc-blur.md) by providing the raw magnitude value (stored in alpha) rather than performing the weighted blur specific to foreground elements.

## How it works
**Inputs:**
- Camera depth texture (`_CameraDepthTexture`).
- Shader parameters packed in `_CurveParams`:
  - `_CurveParams.x`: Maximum CoC value.
  - `_CurveParams.y`: Minimum CoC threshold (values below this become 0).
  - `_CurveParams.z`: Scale factor.
  - `_CurveParams.w`: Focal depth plane (linear depth).
- Current UV coordinates.

**Outputs:**
- A color value where the alpha channel contains the clamped, unsigned CoC magnitude.

**Data Structures:**
- The CoC is stored as a single floating-point value in the alpha channel of the output color.

**Steps:**
1. Sample the camera depth texture at the current UV coordinates to obtain the raw depth value.
2. Convert the raw depth to linear 0-1 depth space.
3. Calculate the raw CoC magnitude using the scale factor, focal depth, and linear depth.
4. Subtract the minimum threshold. If the result is negative, clamp to 0.
5. Clamp the final result to the maximum CoC value.

**Formula:**
The raw CoC magnitude $C_{raw}$ is calculated as:

$$ C_{raw} = \frac{\text{_CurveParams.z} \times (\text{_CurveParams.w} - d)}{d + 10^{-5}} $$

Where:
- $d$ is the linear depth in 0-1 range.
- `_CurveParams.z` is the scale factor.
- `_CurveParams.w` is the focal depth.

The final alpha value $\alpha$ is:

$$ \alpha = \text{clamp}(\max(0.0, C_{raw} - \text{_CurveParams.y}), 0.0, \text{_CurveParams.x}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_capture_coc(uv, camera_depth_texture, curve_params):
    # 1. Sample depth
    raw_depth = sample_depth_texture(camera_depth_texture, uv)
    
    # 2. Convert to linear 0-1 depth
    # Implementation detail: Linear01Depth converts non-linear depth to linear 0-1 range.
    # This depends on the specific graphics API and depth buffer format.
    d = linear_01_depth(raw_depth)
    
    # 3. Calculate raw CoC magnitude
    # Note: The source formula uses (focal_depth - d). 
    # This implies positive CoC for objects closer than focal plane if scale is positive,
    # or it represents a specific directional convention.
    scale_factor = curve_params.z
    focal_depth = curve_params.w
    min_threshold = curve_params.y
    max_coc = curve_params.x
    
    epsilon = 1e-5
    numerator = scale_factor * (focal_depth - d)
    denominator = d + epsilon
    
    raw_coc = numerator / denominator
    
    # 4. Clamp to unsigned magnitude
    # Subtract min threshold
    adjusted_coc = raw_coc - min_threshold
    
    # Clamp to max(0, ...) to remove negative values (unsigned)
    clamped_coc = max(0.0, adjusted_coc)
    
    # Clamp to max CoC
    final_coc = min(clamped_coc, max_coc)
    
    return final_coc
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** An epsilon ($10^{-5}$) is added to the denominator to prevent division by zero when depth $d$ is 0.
- **Negative Depth Differences:** The formula $(\text{_CurveParams.w} - d)$ can yield negative values if the object is behind the focal plane (assuming standard depth conventions). The `max(0.0, ...)` step ensures the output is unsigned, discarding the sign information.
- **Precision:** Linear depth conversion precision varies by platform and depth buffer format (e.g., 16-bit vs 24-bit/32-bit). Low precision can cause banding in the CoC gradient.

## Subtopics

### Depth-based Circle of Confusion (CoC) Calculation
This variant explicitly details the depth sampling and linearization steps. It emphasizes that the raw CoC is derived from the difference between the focal depth and the linear depth, scaled by a factor. It is chosen when the implementation requires explicit control over the depth texture sampling and linearization process. It is the core calculation logic for Capture CoC.

### Capture Color and Signed CoC
This variant calculates a **signed** CoC instead of an unsigned magnitude. It stores the sign based on whether the depth $d$ is greater than or less than the focal depth `_CurveParams.w`. The magnitude is calculated as `_CurveParams.z * abs(d - _CurveParams.w) / (d + 1e-5f)`. The final value is clamped and then multiplied by `sign(d - _CurveParams.w)`. This is chosen when the downstream blur pass needs to distinguish between foreground and background blur directionality or intensity, although Capture CoC itself typically outputs unsigned values.

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel. It involves a single texture sample, basic arithmetic operations, and clamping.
- **Memory Complexity:** $O(1)$ auxiliary memory. It requires reading the depth texture and storing the result in the color buffer.
- **Performance:** Extremely fast, suitable for real-time post-processing. The cost is dominated by the depth texture read.
- **Strengths:** Simple to implement, computationally inexpensive, provides accurate CoC values based on camera parameters.
- **Limitations:** Does not account for lens distortion or complex optical effects. The unsigned nature discards depth-relative direction information.

## Background
The method originates from Unity's post-processing stack, specifically within the Depth of Field implementation. It is used to generate the CoC map which is subsequently used by blur passes (such as Foreground CoC Blur) to determine the blur radius for each pixel. The term "Capture CoC" refers to the stage where the CoC is "captured" from the depth buffer.

## Sources
[S1] dario-zubovic/metaballs, https://github.com/dario-zubovic/metaballs, fetched 2026-10-26 (unofficial)
