# Spherical Beta View-Dependent Color

Also called: Spherical Beta lobes

## Overview
Spherical Beta (SB) is a specular model that adds explicit specular lobes on top of the base or Spherical Harmonic (SH) color to model view-dependent appearance in 3D Gaussian Splatting [S1, unofficial]. It is derived from Deformable Beta Splatting (Liu et al., SIGGRAPH 2025) [S1, unofficial]. Unlike SH, which represents the full view-dependent function, SB uses a small number of explicit lobes, offering higher PSNR and significantly smaller file sizes than SH degree 3 [S1, unofficial]. It builds on [View-dependent color with Spherical Harmonics](1369-view-dependent-color-with-spherical-harmonics.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Gaussian splat parameters including base color or SH coefficients, and optional SB lobe parameters (axis angles, sharpness, color).
*   **Output:** Final view-dependent color for the splat.

**Data Structures:**
*   **Lobe Parameters:** Stored as half-floats (16-bit floats).
*   **Storage Layout:** One 16-byte texel is stored per lobe per splat [S1, unofficial].
*   **Lobe Definition:** Each lobe is defined by:
    *   Axis: Defined by spherical angles (converted to a 3D vector at load time) [S1, unofficial].
    *   Sharpness ($\beta$): A scalar value [S1, unofficial].
    *   Color ($rgb$): A 3D vector [S1, unofficial].

**Number of Lobes:**
*   Up to 4 lobes are supported [S1, unofficial].
*   Trainers default to 2 lobes [S1, unofficial].
*   2 lobes require 12 floats per splat (assuming 3 floats per lobe for color and 1 for sharpness, plus axis storage details implied by 16-byte texel) [S1, unofficial]. Note: The source states "2 lobes are 12 floats per splat" and "one 16-byte texel per lobe". 16 bytes = 2 half-floats. If axis is 2 angles (2 half-floats), sharpness (1 half-float), color (3 half-floats), that is 6 half-floats = 12 bytes. The source says 16-byte texel. This implies padding or specific packing. The source explicitly states "12 floats per splat" for 2 lobes. We will stick to the explicit "12 floats" and "16-byte texel" statements.

**Algorithm Steps:**
1.  **Preprocessing (Load Time):**
    *   Convert lobe parameters from PLY properties (`sb_params_*`) [S1, unofficial].
    *   Convert axis from spherical angles to a 3D direction vector [S1, unofficial].
    *   Store parameters as half-floats [S1, unofficial].
    *   If a splat has lobes but no SH, mark it for dynamic re-evaluation [S1, unofficial].

2.  **Rendering (Shader):**
    *   For each splat:
        *   Compute the view direction vector $dir$ from the camera to the splat center [G].
        *   Initialize $color = 0.0$ [S1, unofficial].
        *   If SH coefficients are present:
            *   Evaluate SH color $C_{SH}(dir)$ [S1, unofficial].
            *   $color = \max(C_{SH}(dir), 0.0)$ [S1, unofficial].
        *   Else if base color is present:
            *   $color = \max(base\_color, 0.0)$ [S1, unofficial].
        *   For each lobe $i$ (from 0 to $N-1$, where $N \le 4$):
            *   Get lobe axis $axis_i$, sharpness $\beta_i$, and color $rgb_i$ [S1, unofficial].
            *   Compute dot product: $d = \max(dot(dir, axis_i), 0.0)$ [S1, unofficial].
            *   Compute lobe contribution:
                $$ L_i = pow(d, 4 \cdot \exp(\beta_i)) \cdot softplus(rgb_i) $$
                Where $softplus(x) = \ln(1 + \exp(x))$ [G, standard definition].
            *   Add to color: $color = color + L_i$ [S1, unofficial].
        *   Return $color$.

**Formulas:**
*   Lobe Contribution:
    $$ L = pow(\max(dot(dir, axis), 0), 4 \cdot \exp(\beta)) \cdot softplus(rgb) $$
    *   $dir$: View direction vector [S1, unofficial].
    *   $axis$: Lobe axis vector [S1, unofficial].
    *   $\beta$: Sharpness parameter [S1, unofficial].
    *   $rgb$: Lobe color vector [S1, unofficial].
    *   $softplus(x) = \ln(1 + \exp(x))$ [G].

**Parameters:**
*   Number of lobes: 2 (default), up to 4 [S1, unofficial].
*   Sharpness ($\beta$): Learned parameter [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Negative Base Color:** The base/SH color is clamped to zero before adding lobes to prevent negative values from absorbing highlights [S1, unofficial].
*   **Dynamic Re-evaluation:** Splats with lobes but no SH must re-evaluate color as the camera moves [S1, unofficial].
*   **Format Support:** Only uncompressed PLY supports lobe data. Compressed PLY, SOG, and SPZ do not support lobe storage [S1, unofficial].
*   **Beta Kernel:** This method only implements the color lobes, not the Beta kernel (opacity falloff). Using the Beta kernel requires additional changes to quad extent, bounds/culling, and per-splat values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_spherical_beta_color(splat, view_dir):
    """
    Evaluates the view-dependent color for a splat using Spherical Beta lobes.
    
    Args:
        splat: Object containing:
            - sh_coeffs: List of SH coefficients (optional)
            - base_color: RGB vector (optional)
            - lobes: List of lobe dictionaries, each with:
                - axis: 3D vector (unit length)
                - beta: float (sharpness)
                - rgb: 3D vector
        view_dir: 3D vector (direction from camera to splat)
        
    Returns:
        RGB color vector
    """
    # Initialize color to zero
    color = vec3(0.0)
    
    # Add base or SH color, clamped to zero
    if splat.sh_coeffs is not None:
        # Evaluate SH color (assuming a helper function exists for SH evaluation)
        sh_color = evaluate_sh(splat.sh_coeffs, view_dir)
        color = max(sh_color, vec3(0.0))
    elif splat.base_color is not None:
        color = max(splat.base_color, vec3(0.0))
    
    # Add lobe contributions
    for lobe in splat.lobes:
        axis = lobe.axis
        beta = lobe.beta
        rgb = lobe.rgb
        
        # Compute dot product, clamped to [0, 1]
        dot_val = max(dot(view_dir, axis), 0.0)
        
        # Compute lobe power term
        # 4 * exp(beta)
        exponent = 4.0 * exp(beta)
        lobe_pow = pow(dot_val, exponent)
        
        # Compute softplus of color
        # softplus(x) = ln(1 + exp(x))
        lobe_rgb = vec3(0.0)
        for i in range(3):
            lobe_rgb[i] = log(1.0 + exp(rgb[i]))
            
        # Add contribution
        color += lobe_pow * lobe_rgb
        
    return color
```

## Subtopics

### Progressive View-Dependent Color Baking
A method for baking view-dependent color, potentially using different approximations. See [Progressive View-Dependent Color Baking](2834-progressive-view-dependent-color-baking.md).

## Operating characteristics
*   **Memory:** 2 lobes require 12 floats per splat, compared to 45 floats for SH bands 1-3 [S1, unofficial]. This results in approximately 45% of the PLY file size compared to SH degree 3 [S1, unofficial].
*   **Performance:** The shader performs one `pow` operation per lobe [S1, unofficial].
*   **Quality:** On the Mip-NeRF 360 benchmark, SH degree 0 + 2 lobes achieves +0.15 to +0.30 dB higher PSNR than SH degree 3 [S1, unofficial].
*   **Strengths:** Smaller file size, faster rendering (fewer operations than high-degree SH), higher PSNR than SH degree 3 for specular highlights.
*   **Limitations:** Only supports uncompressed PLY. Does not include the Beta kernel (falloff). Limited to explicit lobes, may not capture complex view-dependent effects as well as high-degree SH.
*   **When to use:** When specular highlights are important and file size/performance is a concern, and uncompressed PLY is acceptable.
*   **When not to use:** When complex view-dependent effects beyond simple lobes are needed, or when compressed PLY/SOG/SPZ formats are required.

## Background
Spherical Beta view-dependent color is based on the specular model from Deformable Beta Splatting (Liu et al., SIGGRAPH 2025) [S1, unofficial]. It was implemented in the PlayCanvas engine (Pull Request #9729) for WebGL2 and WebGPU [S1, unofficial]. It is also integrated into SuperSplat and gsplat [S1, unofficial].

## Sources
[S1] Add Spherical Beta view-dependent color for PLY gaussian splats, https://github.com/playcanvas/engine/pull/9729, fetched 2026-10-27 (unofficial)
