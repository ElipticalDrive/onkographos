# Spherical Beta View-Dependent Color

Also called: Spherical Beta lobes

## Overview
Spherical Beta (SB) is a specular model that adds explicit specular lobes on top of the base or Spherical Harmonic (SH) color to model view-dependent appearance in 3D Gaussian Splatting [S1, unofficial]. It is derived from Deformable Beta Splatting (Liu et al., SIGGRAPH 2025) [S1, unofficial]. Unlike SH, which represents the full view-dependent function, SB uses a small number of explicit lobes, offering higher PSNR and significantly smaller file sizes than SH degree 3 [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Gaussian splat parameters including base color or SH coefficients, and optional SB lobe parameters (axis angles, sharpness, color) [S1, unofficial].
*   **Output:** Final view-dependent color for the splat [S1, unofficial].

**Data Structures:**
*   **Lobe Parameters:** Stored as half-floats (16-bit floats) [S1, unofficial].
*   **Storage Layout:** One 16-byte texel is stored per lobe per splat [S1, unofficial].
*   **Lobe Definition:** Each lobe is defined by an axis (converted from spherical angles to a 3D vector at load time), a sharpness scalar ($\beta$), and a 3D color vector ($rgb$) [S1, unofficial].

**Number of Lobes:**
*   Up to 4 lobes are supported [S1, unofficial].
*   Trainers default to 2 lobes [S1, unofficial].
*   Two lobes require 12 floats per splat [S1, unofficial].

**Algorithm Steps:**
1.  **Preprocessing (Load Time):** Convert lobe parameters from PLY properties (`sb_params_*`) [S1, unofficial]. Convert axis from spherical angles to a 3D direction vector [S1, unofficial]. Store parameters as half-floats [S1, unofficial]. If a splat has lobes but no SH, mark it for dynamic re-evaluation as the camera moves [S1, unofficial].
2.  **Rendering (Shader):** For each splat, compute the view direction vector $dir$ from the camera to the splat center [G]. Initialize $color$ to the base or SH color clamped to zero [S1, unofficial]. For each lobe $i$ (from 0 to $N-1$, where $N \le 4$), compute the dot product $d = \max(dot(dir, axis_i), 0.0)$ [S1, unofficial]. Compute the lobe contribution $L_i = pow(d, 4 \cdot \exp(\beta_i)) \cdot softplus(rgb_i)$ [S1, unofficial]. Add $L_i$ to $color$ [S1, unofficial]. Return $color$.

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
*   **Beta Kernel:** This method only implements the color lobes, not the Beta kernel (opacity falloff) [S1, unofficial].

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
    # Initialize color accumulator
    color = vec3(0.0)
    
    # Add base or SH color, clamped to zero
    if splat.sh_coeffs is not None:
        # Evaluate SH color from coefficients and view_dir [G]
        sh_color = vec3(0.0) # Placeholder for standard SH evaluation
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
        
        # Compute lobe power term: pow(max(dot(dir, axis), 0), 4 * exp(beta))
        exponent = 4.0 * exp(beta)
        lobe_pow = pow(dot_val, exponent)
        
        # Compute softplus of color: softplus(x) = ln(1 + exp(x))
        lobe_rgb = vec3(0.0)
        for i in range(3):
            lobe_rgb[i] = log(1.0 + exp(rgb[i]))
            
        # Add contribution
        color += lobe_pow * lobe_rgb
        
    return color
```

## Subtopics

### Progressive View-Dependent Color Baking
A method for baking view-dependent appearance using iterative optimization. See [Progressive View-Dependent Color Baking](2834-progressive-view-dependent-color-baking.md).

## Operating characteristics
*   **Memory:** Two lobes require 12 floats per splat, compared to 45 floats for SH bands 1-3 [S1, unofficial]. This results in approximately 45% of the PLY file size compared to SH degree 3 [S1, unofficial].
*   **Performance:** The shader performs one `pow` operation per lobe [S1, unofficial].
*   **Quality:** On the Mip-NeRF 360 benchmark, SH degree 0 + 2 lobes achieves +0.15 to +0.30 dB higher PSNR than SH degree 3 [S1, unofficial].
*   **Strengths:** Smaller file size, faster rendering (fewer operations than high-degree SH), higher PSNR than SH degree 3 for specular highlights [S1, unofficial].
*   **Limitations:** Only supports uncompressed PLY. Does not include the Beta kernel (falloff). Limited to explicit lobes, may not capture complex view-dependent effects as well as high-degree SH [S1, unofficial].
*   **When to use:** When specular highlights are important and file size/performance is a concern, and uncompressed PLY is acceptable [S1, unofficial].
*   **When not to use:** When complex view-dependent effects beyond simple lobes are needed, or when compressed PLY/SOG/SPZ formats are required [S1, unofficial].

## Background
Spherical Beta view-dependent color is based on the specular model from Deformable Beta Splatting (Liu et al., SIGGRAPH 2025) [S1, unofficial]. It was implemented in the PlayCanvas engine (Pull Request #9729) for WebGL2 and WebGPU [S1, unofficial]. It is also integrated into SuperSplat and gsplat [S1, unofficial].

## Sources
[S1] Add Spherical Beta view-dependent color for PLY gaussian splats, https://github.com/playcanvas/engine/pull/9729, fetched 2026-10-27 (unofficial)
