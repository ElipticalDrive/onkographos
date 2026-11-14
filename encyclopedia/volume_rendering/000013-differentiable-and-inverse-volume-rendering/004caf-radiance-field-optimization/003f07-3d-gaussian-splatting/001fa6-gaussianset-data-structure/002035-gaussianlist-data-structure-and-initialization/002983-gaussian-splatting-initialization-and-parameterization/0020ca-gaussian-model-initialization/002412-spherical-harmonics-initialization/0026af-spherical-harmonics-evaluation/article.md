# Spherical Harmonics Evaluation

Also called: SH Evaluation, SH2RGB, Splat Lighting

## Overview
Spherical Harmonics (SH) Evaluation computes the view-dependent RGB color of a 3D Gaussian splat by evaluating real spherical harmonic basis functions against a viewing direction vector and dotting the result with stored SH coefficients. It solves the problem of compactly representing complex lighting effects (such as specular highlights and ambient occlusion) within the Gaussian representation without requiring explicit light sources or ray tracing [S10][S7, unofficial]. It differs from [Spherical Harmonics Decoding](7901-spherical-harmonics-decoding.md) by focusing on the mathematical evaluation of the basis functions rather than the unpacking of raw coefficient storage formats, and from [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md) which handles the transformation of coefficients to camera space [S1, unofficial].

## How it works
**Inputs:**
- `shs`: Tensor of SH coefficients. Shape is typically $(N, 3, (D+1)^2)$ or flattened as $(N, 3 \times (D+1)^2)$, where $N$ is the number of Gaussians and $D$ is the active SH degree [S1, S8, unofficial].
- `viewdirs`: Tensor of normalized viewing direction vectors $(x, y, z)$ for each Gaussian. Shape $(N, 3)$ [S7, unofficial].
- `active_sh_degree`: Integer specifying the maximum degree of spherical harmonics to evaluate (typically 0, 1, 2, or 3) [S2, unofficial].

**Outputs:**
- `rgb`: Tensor of RGB color values. Shape $(N, 3)$ [S3, unofficial].

**Data Structures:**
The SH coefficients are stored per Gaussian, per color channel (R, G, B), and per SH basis function. For degree $D$, there are $(D+1)^2$ coefficients per channel. The coefficients are real-valued floats.

**Constants:**
The following constants are derived from the normalization of real spherical harmonics (Condon-Shortley phase convention) [S4, S7, unofficial]:
- $C_0 = \frac{1}{\sqrt{4\pi}} \approx 0.28209479177387814$
- $C_1 = \sqrt{\frac{3}{4\pi}} \approx 0.48860251190291987$
- $C_2 = \sqrt{\frac{15}{4\pi}} \approx 1.0925484305920792$
- $C_{2,0} = \sqrt{\frac{5}{16\pi}} \approx 0.94617469575755997$ (coefficient for $Y_0^2$ term scaling)
- $C_{2,2} = \sqrt{\frac{15}{16\pi}} \approx 0.54627421529603959$ (coefficient for $Y_2^2$ term scaling)
- $C_3 = \sqrt{\frac{7}{16\pi}} \approx 0.59004358992664352$
- $C_{3,2} = \sqrt{\frac{105}{32\pi}} \approx 2.8906114426405538$
- $C_{3,1} = \sqrt{\frac{21}{32\pi}} \approx 0.45704579946446572$
- $C_{3,0} = \sqrt{\frac{7}{64\pi}} \approx 0.3731763325901154$

**Basis Functions:**
The real spherical harmonic basis functions $Y_l^m$ are evaluated based on the direction components $x, y, z$. Note that $x = \sin\theta \cos\phi$, $y = \sin\theta \sin\phi$, $z = \cos\theta$ [S4, S9, unofficial].

Degree 0 ($l=0$):
$$ Y_0^0 = C_0 $$

Degree 1 ($l=1$):
$$ Y_{-1}^1 = -C_1 y $$
$$ Y_0^1 = C_1 z $$
$$ Y_1^1 = -C_1 x $$

Degree 2 ($l=2$):
$$ Y_{-2}^2 = C_2 xy $$
$$ Y_{-1}^2 = -C_2 yz $$
$$ Y_0^2 = C_{2,0} (2z^2 - x^2 - y^2) $$
$$ Y_1^2 = -C_2 xz $$
$$ Y_2^2 = C_{2,2} (x^2 - y^2) $$

Degree 3 ($l=3$):
$$ Y_{-3}^3 = C_3 y (3x^2 - y^2) $$
$$ Y_{-2}^3 = C_{3,2} xyz $$
$$ Y_{-1}^3 = C_{3,1} y (1 - 5z^2) $$
$$ Y_0^3 = C_{3,0} z (5z^2 - 3) $$
$$ Y_1^3 = C_{3,1} x (1 - 5z^2) $$
$$ Y_2^3 = C_{3,2} z (x^2 - y^2) $$
$$ Y_3^3 = C_3 x (3y^2 - x^2) $$

*Note: Signs of odd-$m$ terms may vary depending on the specific Condon-Shortley implementation. The sources [S4, S7, unofficial] use the signs listed above. Source [S9, unofficial] uses a different indexing/ordering but equivalent polynomial forms.*

**Algorithm Steps:**
1. **Input Validation:** If gradients are required for `viewdirs` or `shs`, proceed with differentiable operations. If `viewdirs` is not provided but `vox_centers` and `cam_pos` are, compute `viewdirs = normalize(vox_centers - cam_pos)` [S2, unofficial].
2. **Initialize Color:** Extract the zeroth-order (DC) coefficients $c_0, c_1, c_2$ (indices 0, 1, 2 for R, G, B respectively) from `shs`. Initialize the output color tensor `rgb` with these DC terms:
   $$ \text{rgb} = C_0 \cdot \text{shs}[:, :, 0] $$
   This represents the average color of the splat [S5, S7, unofficial].
3. **Evaluate Higher Orders:** Depending on `active_sh_degree`, add contributions from higher-order coefficients:
   - **Degree 1:** Extract coefficients $c_3, c_4, c_5$ (indices 3, 4, 5). Update color:
     $$ \text{rgb} \leftarrow \text{rgb} - C_1 y c_3 + C_1 z c_4 - C_1 x c_5 $$
     (Note: The sign of the $x$ term depends on the coefficient indexing. Source [S7, unofficial] uses $-C_1 x c_3$ for the first-order x-term if $c_3$ corresponds to $Y_1^1$. We follow the polynomial expansion directly: $Y_1^1 = -C_1 x$) [S7, unofficial].
   - **Degree 2:** Extract coefficients $c_4 \dots c_8$ (indices 6..10). Compute squared/cross terms: $xx=x^2, yy=y^2, zz=z^2, xy=xy, yz=yz, xz=xz$. Update color:
     $$ \text{rgb} \leftarrow \text{rgb} + C_2 xy c_4 - C_2 yz c_5 + C_{2,0}(2zz - xx - yy) c_6 - C_2 xz c_7 + C_{2,2}(xx - yy) c_8 $$
     (Mapping coefficients to $Y_{-2}^2 \dots Y_2^2$) [S5, S7, unofficial].
   - **Degree 3:** Extract coefficients $c_9 \dots c_{15}$ (indices 11..17). Update color with cubic terms:
     $$ \text{rgb} \leftarrow \text{rgb} + C_3 y(3xx - yy) c_9 + C_{3,2} xyz c_{10} + C_{3,1} y(1 - 5zz) c_{11} + C_{3,0} z(5zz - 3) c_{12} + C_{3,1} x(1 - 5zz) c_{13} + C_{3,2} z(xx - yy) c_{14} + C_3 x(3yy - xx) c_{15} $$
     (Mapping coefficients to $Y_{-3}^3 \dots Y_3^3$) [S5, S7, unofficial].
4. **Bias and Clamping:** Add a bias of 0.5 to the resulting color vector to shift the range from $[-1, 1]$ (or similar normalized range) to $[0.5, 1.5]$ or similar, then apply ReLU or clamp to $[0.0, 1.0]$ to ensure valid RGB values [S3, S7, S8, unofficial].
   $$ \text{rgb} = \text{clamp}(\text{rgb} + 0.5, 0.0, 1.0) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def eval_sh_bases(degree, viewdirs):
    """
    Evaluates real spherical harmonic basis functions for a batch of view directions.
    Returns tensor of shape (N, (degree+1)^2).
    """
    x = viewdirs[:, 0]
    y = viewdirs[:, 1]
    z = viewdirs[:, 2]
    
    bases = []
    
    # Degree 0
    C0 = 0.28209479177387814
    bases.append(C0 * torch.ones_like(x))
    
    if degree >= 1:
        C1 = 0.48860251190291987
        # Y_-1^1, Y_0^1, Y_1^1
        bases.append(-C1 * y)
        bases.append(C1 * z)
        bases.append(-C1 * x)
        
    if degree >= 2:
        C2 = 1.0925484305920792
        C2_0 = 0.94617469575755997
        C2_2 = 0.54627421529603959
        xx, yy, zz = x*x, y*y, z*z
        xy, yz, xz = x*y, y*z, x*z
        # Y_-2^2, Y_-1^2, Y_0^2, Y_1^2, Y_2^2
        bases.append(C2 * xy)
        bases.append(-C2 * yz)
        bases.append(C2_0 * (2*zz - xx - yy))
        bases.append(-C2 * xz)
        bases.append(C2_2 * (xx - yy))
        
    if degree >= 3:
        C3 = 0.59004358992664352
        C3_2 = 2.8906114426405538
        C3_1 = 0.45704579946446572
        C3_0 = 0.3731763325901154
        # Y_-3^3, Y_-2^3, Y_-1^3, Y_0^3, Y_1^3, Y_2^3, Y_3^3
        bases.append(C3 * y * (3*xx - yy))
        bases.append(C3_2 * x * y * z)
        bases.append(C3_1 * y * (1 - 5*zz))
        bases.append(C3_0 * z * (5*zz - 3))
        bases.append(C3_1 * x * (1 - 5*zz))
        bases.append(C3_2 * z * (xx - yy))
        bases.append(C3 * x * (3*yy - xx))
        
    return torch.stack(bases, dim=1)

def spherical_harmonics_evaluation(shs, viewdirs, active_sh_degree):
    """
    Computes RGB color from SH coefficients and view directions.
    
    Args:
        shs: Tensor of shape (N, 3, (active_sh_degree+1)^2) or (N, 3 * (active_sh_degree+1)^2)
        viewdirs: Tensor of shape (N, 3)
        active_sh_degree: Integer, max degree to evaluate
        
    Returns:
        rgb: Tensor of shape (N, 3)
    """
    # Reshape shs to (N, 3, num_bases) if flattened
    if shs.dim() == 2:
        num_bases = (active_sh_degree + 1) ** 2
        shs = shs.view(shs.shape[0], 3, num_bases)
    
    # Evaluate bases
    # sh_mult shape: (N, num_bases)
    sh_mult = eval_sh_bases(active_sh_degree, viewdirs)
    
    # Dot product: sum over basis dimension
    # shs shape: (N, 3, num_bases)
    # sh_mult shape: (N, num_bases) -> expand to (N, 1, num_bases) for broadcasting
    rgb_sh = torch.sum(shs * sh_mult[:, None, :], dim=-1)
    
    # Add bias and clamp
    rgb = torch.relu(rgb_sh + 0.5)
    
    return rgb
```

**Edge Cases and Pitfalls:**
- **Precision:** The polynomial terms for higher degrees (e.g., $z^3$) can suffer from precision loss if $z$ is close to 1 or -1. Using normalized vectors is critical [S7, unofficial].
- **Boundary Conditions:** If `viewdirs` is not normalized, the SH evaluation will produce incorrect colors. Always normalize `viewdirs` before evaluation [S7, unofficial].
- **Coefficient Indexing:** Different implementations may store SH coefficients in different orders (e.g., $m$ vs $l$ ordering). The pseudocode assumes the standard ordering $Y_{-l}^l \dots Y_l^l$ for each $l$ [S4, S7, unofficial].
- **Performance:** Implementations may ignore higher-degree coefficients (e.g., degree 3) for performance reasons, especially on mobile or low-end hardware [S10].

## Subtopics

### [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md)
Spherical Harmonics Rotation computes the new SH coefficients of a function after rotating the viewing coordinate system by $\pm \pi/2$ around the X-axis. It sets itself apart from general rotation methods by using precomputed analytic tables for specific $\pm 90^\circ$ rotations, avoiding expensive matrix multiplications or numerical integration.

### [Spherical Harmonics Encoder](4390-spherical-harmonics-encoder.md)
The Spherical Harmonics Encoder is a neural network module that encodes a 3D direction vector into spherical harmonics coefficients up to a specified degree. It solves the problem of representing directional information (view direction) in a differentiable, basis-function expansion for view-dependent rendering.

### [Spherical Harmonics Data Extraction](6960-spherical-harmonics-data-extraction.md)
Spherical Harmonics (SH) Data Extraction parses SH coefficients from a 3D Gaussian Splatting model file (typically a PLY format) into a structured data layout suitable for GPU rendering. It solves the problem of mapping flat, property-based storage in the source file to the hierarchical texture arrays required by the rendering engine. It is distinct from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md), which computes color from these coefficients, and [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md), which transforms them for view-dependent effects.

### [Spherical Harmonics Decoding](7901-spherical-harmonics-decoding.md)
Spherical Harmonics (SH) Decoding is the process of reconstructing the RGB color of a 3D Gaussian splat from its stored SH coefficients and a specific viewing direction. It serves as the final color computation step in the Gaussian Splatting pipeline, translating directional frequency data into perceptual color. It is distinguished from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md) by its specific focus on unpacking and interpreting the raw coefficient storage format (often packed in GPU buffers) rather than just evaluating the basis functions, and from [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md) which handles the transformation of coefficients to camera space.

### [Spherical Harmonics Lighting Calculation](11516-spherical-harmonics-lighting-calculation.md)
This method computes the RGB color contribution of a 3D Gaussian splat by evaluating real spherical harmonics (SH) against a viewing direction vector. It solves the problem of representing view-dependent lighting effects (such as specular highlights and ambient occlusion) compactly within the Gaussian representation. It differs from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md) by explicitly defining the lighting calculation pipeline, including the specific Condon–Shortley phase conventions, the mandatory 0.5 bias for numeric stability, and the exact polynomial expansions for degrees 0 through 3.

### Spherical Harmonics Evaluation with Tone Mapping
A module that interprets network output as spherical harmonics coefficients to model view-dependent effects and camera-specific parameters, followed by tone mapping [S11]. The SH coefficients are the output of the last convolution of the neural network [S11]. It applies a tone mapper following Rückert et al. [RFS22] which models exposure time, white balance, sensor response, and vignetting [S11]. It uses spherical harmonics degree 2, corresponding to 27 input coefficients (9 coefficients per color channel) [S11].

## Operating characteristics
- **Time Complexity:** $O(N \cdot D^2)$, where $N$ is the number of Gaussians and $D$ is the active SH degree. For fixed degree (e.g., $D=2$), it is $O(N)$. The evaluation involves a constant number of multiplications and additions per Gaussian.
- **Memory Complexity:** $O(N \cdot D^2)$ to store the SH coefficients. The evaluation itself requires $O(N)$ temporary memory for the basis functions and intermediate color values.
- **Measured Performance:** On modern GPUs, evaluating degree 2 SH for millions of Gaussians is highly parallelizable and typically takes less than 1ms for $10^6$ splats [S3, S8, unofficial].
- **Scaling:** Scales linearly with the number of Gaussians. Higher degrees increase the constant factor significantly (degree 3 has 7x more coefficients than degree 1).
- **Strengths:** Compact representation of view-dependent effects; differentiable; efficient on GPU.
- **Limitations:** High-frequency lighting details may require high degrees, increasing memory and compute cost. Aliasing can occur if the Gaussian size is small relative to the frequency of the SH basis [S10].
- **When to use:** When view-dependent appearance is important and memory/compute budget allows for storing and evaluating SH coefficients.
- **When not to use:** For simple, view-independent materials (degree 0 is sufficient), or when extreme high-frequency details are needed and higher degrees are too expensive.

## Background
Spherical Harmonics have been used in computer graphics for decades to represent lighting and reflectance. In the context of 3D Gaussian Splatting, they were adopted to provide a compact, differentiable way to model view-dependent appearance without the need for explicit light sources or complex BRDFs [S10]. The specific implementation details (polynomial expansions, constants) are derived from standard mathematical definitions of real spherical harmonics [S4, S9, unofficial]. The use of SH in Gaussian Splatting was popularized by the original 3D Gaussian Splatting paper and subsequent implementations [S1, S7, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S4] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S5] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S6] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
[S7] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S8] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S9] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
[S10] https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, fetched 2026-10-30
[S11] TRIPS: Trilinear Point Splatting for Real‐Time Radiance Field Rendering, https://doi.org/10.1111/cgf.15012, fetched 2026-10-27
