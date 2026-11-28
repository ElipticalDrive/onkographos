# Spherical Harmonics Degree 0 Color Reconstruction

Also called: Fallback Color Calculation, SH Degree 0 Fallback Color Computation, SH0 Color Processing, _COEF_0 Decoding

## Overview
This method reconstructs the base diffuse color of a 3D Gaussian splat from its stored zeroth-order spherical harmonic coefficient ($l=0$) to support fallback rendering in glTF viewers that do not support full 3D Gaussian Splatting [S1, S2]. It differs from higher-degree SH decoding by evaluating only the constant basis function, effectively treating the color as a uniform diffuse value independent of viewing direction [S1, S2]. This approach provides a simplified, direction-independent color representation suitable for point-cloud visualization or ellipse-kernel fallbacks [S2].

## How it works

### Inputs and Outputs
**Inputs:**
- `coef_0`: A vector of 3 floating-point values representing the stored RGB spherical harmonic coefficients for degree 0 (attribute `KHR_gaussian_splatting:SH_DEGREE_0_COEF_0`) [S1, S2].
- `color_space`: A string indicating the color space of the scene, typically `"srgb_rec709_display"` or `"lin_rec709_display"` [S1, S2].
- `opacity`: A floating-point value representing the alpha channel of the splat [S1, S2].

**Outputs:**
- `color`: A vector of 3 floating-point values in linear RGB space, suitable for the `COLOR_0` attribute semantic [S1, S2].
- `alpha`: The opacity value, copied directly from the input [S1, S2].

### Data Structures
The input coefficient `coef_0` is typically stored as a `VEC3` attribute in the glTF buffer view [S1, S2]. The output `color` is stored as a `VEC4` attribute (`COLOR_0`) where the RGB channels contain the reconstructed linear color and the A channel contains the opacity [S1, S2].

### Steps
1.  **Scale:** Multiply each component of `coef_0` by the constant zeroth-order spherical harmonic value $Y_0 = 0.282095$ [S1, S2].
2.  **Bias:** Add $0.5$ to each scaled component [S1, S2].
3.  **Clamp:** Clamp each component to the range $[0, 1]$ [S1, S2].
4.  **Decode (Conditional):** If `color_space` is `"srgb_rec709_display"`, apply the inverse sRGB transfer function to the clamped RGB values to convert them from sRGB to linear space [S1, S2]. If `color_space` is `"lin_rec709_display"`, no decoding is required as the values are already linear [S2].
5.  **Assign Alpha:** Set the alpha channel of the output color to the input `opacity` [S1, S2].

### Formulas
The core reconstruction formula for each RGB component $C_{in}$ is:

$$
C_{out} = \text{clamp}(C_{in} \times 0.282095 + 0.5, 0, 1)
$$

Where:
- $C_{in}$ is the stored coefficient value for a specific RGB channel [S1, S2].
- $0.282095$ is the approximate value of the constant spherical harmonic basis function for $l=0$ (exact value: $0.2820947917738781$) [S1, S2].
- $0.5$ is the bias term [S1, S2].
- $\text{clamp}(v, min, max)$ restricts $v$ to the interval $[min, max]$ [S1, S2].

If `color_space` is `"srgb_rec709_display"`, the final linear color $C_{linear}$ is obtained by applying the inverse sRGB transfer function $T^{-1}$ to $C_{out}$:

$$
C_{linear} = T^{-1}(C_{out})
$$

The inverse sRGB transfer function is defined as:

$$
T^{-1}(v) = \begin{cases} 
\frac{v}{12.92} & \text{if } v \le 0.04045 \\
\left(\frac{v + 0.055}{1.055}\right)^{2.4} & \text{if } v > 0.04045 
\end{cases}
$$

### Parameters
- **SH0 Constant:** $0.282095$ (approximate) or $0.2820947917738781$ (exact) [S1, S2].
- **Bias:** $0.5$ [S1, S2].
- **Clamp Range:** $[0, 1]$ [S1, S2].

### Edge Cases and Pitfalls
- **Color Space Mismatch:** If the scene uses `"srgb_rec709_display"` and the sRGB-to-linear decoding step is skipped, the resulting colors will appear washed out or incorrect because the `COLOR_0` attribute in glTF is defined to contain linear values [S1, S2].
- **Precision:** The constant $0.282095$ is an approximation. Using the exact value $0.2820947917738781$ may yield slightly different results, but the difference is negligible for visual purposes [S1, S2].
- **Degenerate Input:** If `coef_0` contains values outside the expected range, the clamping step ensures the output remains within $[0, 1]$ before any potential sRGB decoding [S1, S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_sh0_color(coef_0: list[float], color_space: str, opacity: float) -> tuple[list[float], float]:
    """
    Reconstructs the base diffuse color from SH degree 0 coefficients.
    
    Args:
        coef_0: List of 3 floats [R, G, B] representing the stored SH0 coefficients.
        color_space: String indicating the color space, e.g., "srgb_rec709_display" or "lin_rec709_display".
        opacity: Float representing the splat opacity.
        
    Returns:
        Tuple of (linear_color: list[float], alpha: float).
    """
    SH0_CONSTANT = 0.282095
    BIAS = 0.5
    
    linear_color = []
    
    for c in coef_0:
        # Step 1: Scale
        scaled = c * SH0_CONSTANT
        # Step 2: Bias
        biased = scaled + BIAS
        # Step 3: Clamp
        clamped = max(0.0, min(1.0, biased))
        
        # Step 4: Decode if necessary
        if color_space == "srgb_rec709_display":
            linear_val = inverse_srgb(clamped)
        else:
            # For "lin_rec709_display" or other linear spaces, no decoding is needed
            linear_val = clamped
            
        linear_color.append(linear_val)
        
    return linear_color, opacity

def inverse_srgb(v: float) -> float:
    """
    Applies the inverse sRGB transfer function to convert sRGB to linear.
    
    Args:
        v: Float value in sRGB space, expected to be in [0, 1].
        
    Returns:
        Float value in linear space.
    """
    if v <= 0.04045:
        return v / 12.92
    else:
        return ((v + 0.055) / 1.055) ** 2.4
```

## Subtopics

### Fallback Color Calculation
- **What it is:** The process of generating a simple, direction-independent color for 3D Gaussians to support renderers that do not support full Gaussian Splatting.
- **What sets it apart:** It uses only the zeroth-order SH coefficient, ignoring directional information provided by higher-order coefficients.
- **When to choose it:** When rendering 3D Gaussian Splatting data in a viewer that lacks support for the `KHR_gaussian_splatting` extension or when a simplified point-cloud visualization is sufficient.
- **Link:** [Fallback Color Calculation](7901-spherical-harmonics-degree-0-color-reconstruction.md)

### SH Degree 0 Fallback Color Computation
- **What it is:** The specific computation of multiplying the SH0 coefficient by the constant basis function and applying bias/clamp.
- **What sets it apart:** It is the core mathematical operation within the fallback color calculation.
- **When to choose it:** When implementing the core logic of the fallback color calculation.
- **Link:** [SH Degree 0 Fallback Color Computation](7901-spherical-harmonics-degree-0-color-reconstruction.md)

### SH0 Color Processing
- **What it is:** The overall process of handling the SH0 coefficient, including scaling, biasing, clamping, and color space decoding.
- **What sets it apart:** It encompasses all steps required to convert the stored coefficient into a usable linear color value.
- **When to choose it:** When implementing the complete SH0 color processing pipeline.
- **Link:** [SH0 Color Processing](7901-spherical-harmonics-degree-0-color-reconstruction.md)

### _COEF_0 Decoding
- **What it is:** The decoding of the `_COEF_0` attribute from the glTF buffer into a linear color value.
- **What sets it apart:** It focuses on the attribute-specific decoding logic.
- **When to choose it:** When reading and decoding the `_COEF_0` attribute from a glTF file.
- **Link:** [_COEF_0 Decoding](7901-spherical-harmonics-degree-0-color-reconstruction.md)

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity:** $O(1)$ per Gaussian splat, as it involves a fixed number of arithmetic operations and a conditional sRGB decoding.
- **Memory Complexity:** $O(1)$ per Gaussian splat, requiring storage for the input coefficient and output color.

### Measured Performance
- Not given in the sources.

### Scaling
- Not given in the sources.

### Strengths and Limitations
- **Strengths:** Simple to implement, computationally inexpensive, provides a reasonable approximation of the diffuse color for fallback rendering [S1, S2].
- **Limitations:** Ignores directional color information, resulting in a less accurate representation compared to higher-degree SH decoding [S1, S2].

### When to Use and When Not
- **Use:** When rendering 3D Gaussian Splatting data in a viewer that does not support the `KHR_gaussian_splatting` extension or when a simplified visualization is sufficient [S1, S2].
- **Do Not Use:** When high-fidelity, direction-dependent color rendering is required [S1, S2].

## Background

### Origin
The method is defined in the `KHR_gaussian_splatting` extension for glTF, which provides a standardized format for storing 3D Gaussian Splatting data [S1, S2].

### History and Development
The `KHR_gaussian_splatting` extension was developed by Khronos Group to support the growing adoption of 3D Gaussian Splatting in web and interactive applications [S1, S2]. The fallback color calculation was included to ensure compatibility with renderers that do not support the full extension [S1, S2].

### Where It Is Used
The method is used in glTF viewers and renderers that support the `KHR_gaussian_splatting` extension to provide a fallback color for 3D Gaussian splats when full rendering is not supported [S1, S2].

## Sources

[S1] KHR_gaussian_splatting, https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, fetched 2026-10-30
[S2] KHR_gaussian_splatting, https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_gaussian_splatting, fetched 2026-10-30
