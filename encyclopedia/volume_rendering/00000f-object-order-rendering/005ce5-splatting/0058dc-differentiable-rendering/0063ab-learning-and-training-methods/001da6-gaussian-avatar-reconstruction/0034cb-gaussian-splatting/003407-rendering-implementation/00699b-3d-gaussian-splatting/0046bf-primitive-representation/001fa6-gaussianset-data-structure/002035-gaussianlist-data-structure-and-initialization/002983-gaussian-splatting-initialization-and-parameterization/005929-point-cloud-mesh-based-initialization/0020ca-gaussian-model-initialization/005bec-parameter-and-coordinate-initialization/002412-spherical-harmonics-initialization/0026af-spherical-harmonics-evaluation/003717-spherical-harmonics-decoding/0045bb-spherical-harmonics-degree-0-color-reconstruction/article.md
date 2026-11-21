# Spherical Harmonics Degree 0 Color Reconstruction

Also called: Fallback Color Calculation, SH0 Color Processing, _COEF_0 Decoding

## Overview
Reconstructs the base diffuse color of a 3D Gaussian splat from its stored zeroth-order spherical harmonic coefficient ($SH_0$). It serves as the fallback color mechanism for the `COLOR_0` attribute in glTF, allowing renderers without 3D Gaussian splatting support to display the point cloud with correct diffuse shading [S1]. It is distinguished from higher-degree SH decoding by evaluating only the constant basis function, ignoring directional variation [S1].

## How it works
**Inputs:**
- `coef_0`: A 3-component vector (RGB) containing the stored SH0 coefficients for the splat [S1].
- `color_space`: A string indicating the color space of the input data (e.g., `"srgb_rec709_display"`, `"lin_rec709_display"`) [S1].

**Outputs:**
- `color`: A 3-component linear RGB vector representing the reconstructed diffuse color [S1].

**Data Structures:**
- `coef_0`: Typically stored as 3 floats (or packed integers in glTF extensions) representing the RGB coefficients [S1].

**Steps:**
1. For each component $C_{in} \in \{R, G, B\}$ of `coef_0`:
   a. Multiply by the SH0 constant $k_0 = 0.282095$.
   b. Add the bias $b = 0.5$.
   c. Clamp the result to the range $[0, 1]$.
2. If `color_space` is `"srgb_rec709_display"`, apply the inverse sRGB transfer function to the clamped linear values to decode them into the final linear color space [S1].
3. Return the resulting 3-component vector [S1].

**Formula:**
$$
C_{out} = \text{clamp}(C_{in} \cdot 0.282095 + 0.5, 0, 1)
$$
Where $C_{in}$ is the stored RGB coefficient and $C_{out}$ is the intermediate clamped value [S1]. If `color_space` is `"srgb_rec709_display"`, the final color $C_{final}$ is obtained by applying the inverse sRGB transfer function to $C_{out}$ [S1].

**Parameters:**
- $k_0 = 0.282095$ (exact value $0.2820947917738781$): The multiplicative scaling factor for the degree-0 spherical harmonic basis evaluation [S1].
- $b = 0.5$: The bias added to shift the coefficient range to the standard color range [S1].

**Edge Cases and Pitfalls:**
- **Color Space Mismatch:** The `COLOR_0` attribute in glTF stores linear values [S1]. If the scene uses `'srgb_rec709_display'`, the clamped $[0, 1]$ values must be decoded from sRGB to linear before rendering; otherwise, colors will appear washed out or incorrect [S1].
- **Alpha Channel:** The alpha channel of the resulting `COLOR_0` SHOULD contain the opacity of the splat, separate from the RGB reconstruction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_sh0_color(coef_0: tuple, color_space: str) -> tuple:
    """
    Reconstructs the base color of a 3D Gaussian splat from SH0 coefficients.
    
    Args:
        coef_0: Tuple of 3 floats (R, G, B) representing the stored SH0 coefficients.
        color_space: String indicating the color space, e.g., 'srgb_rec709_display'.
        
    Returns:
        Tuple of 3 floats (R, G, B) in linear color space.
    """
    # SH0 constant and bias
    k0 = 0.282095
    bias = 0.5
    
    reconstructed = []
    
    for c_in in coef_0:
        # Step 1: Scale, bias, and clamp
        c_out = c_in * k0 + bias
        c_out = max(0.0, min(1.0, c_out))
        
        # Step 2: Color space decoding if necessary
        if color_space == "srgb_rec709_display":
            # Apply inverse sRGB transfer function
            # Note: Standard inverse sRGB formula
            if c_out <= 0.04045:
                c_linear = c_out / 12.92
            else:
                c_linear = ((c_out + 0.055) / 1.055) ** 2.4
            reconstructed.append(c_linear)
        else:
            # Assume linear input if not srgb_rec709_display (e.g., lin_rec709_display)
            reconstructed.append(c_out)
            
    return tuple(reconstructed)
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per splat. The operation involves a constant number of arithmetic operations and a conditional branch.
- **Memory Complexity:** $O(1)$. No additional memory is required beyond the input coefficients and output color vector.
- **Strengths:** Extremely fast; suitable for fallback rendering where full SH evaluation is not supported.
- **Limitations:** Only provides diffuse color; ignores directional lighting effects captured by higher-degree SH coefficients [S1].

## Background
This method is defined in the `KHR_gaussian_splatting` extension for glTF [S1]. It enables interoperability by providing a standardized way to extract the base color from the SH0 coefficient, which is stored in the `KHR_gaussian_splatting:SH_DEGREE_0_COEF_0` attribute [S1]. The constant $0.282095$ is the value of the zeroth-order spherical harmonic basis function $Y_{0,0}$ [S1].

## Sources
[S1] KHR_gaussian_splatting Extension Specification, https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, fetched 2026-10-30
