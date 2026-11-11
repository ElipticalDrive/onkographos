# Spherical Harmonics Evaluation

Also called: SH Evaluation, SH2RGB, Splat Lighting

## Overview
Spherical Harmonics (SH) Evaluation converts stored spherical harmonic coefficients into RGB color values for a specific viewing direction. It solves the problem of representing view-dependent appearance (such as specular highlights or iridescence) in 3D Gaussian Splatting by projecting directional frequency data into perceptual color space [S1, S2, S3, S7, unofficial]. Unlike [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md), which sets only the zeroth-order (DC) term from RGB colors, this method computes the full view-dependent color by evaluating higher-order basis functions [S5, S7, unofficial]. It is distinct from [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md), which transforms coefficients for coordinate changes, and [Spherical Harmonics Decoding](7901-spherical-harmonics-decoding.md), which focuses on unpacking raw storage formats [S6, S7, unofficial].

## How it works
**Inputs:**
- `shs`: Tensor of SH coefficients. Shape is typically $(N, 3, (d+1)^2)$ or $(N, 48)$ for degree 3, where $N$ is the number of Gaussians and $d$ is the active SH degree [S1, S5, S7, unofficial]. Each color channel (R, G, B) has separate coefficients.
- `viewdirs`: Tensor of normalized viewing direction vectors $(x, y, z)$ for each Gaussian, pointing from the camera center to the Gaussian mean [S5, S7, unofficial][S10].

**Outputs:**
- `rgb`: Tensor of RGB color values, typically in the range $[0, 1]$ or $[0.5, 1.5]$ before clamping [S3, S5, S8, unofficial].

**Data Structures:**
The SH coefficients are stored per Gaussian. For degree $l$, there are $2l+1$ coefficients per color channel. The total number of coefficients per channel for degree $d$ is $(d+1)^2$.
- Degree 0: 1 coefficient per channel (indices 0, 1, 2 for R, G, B).
- Degree 1: 3 coefficients per channel (indices 3–5, 6–8, 9–11).
- Degree 2: 5 coefficients per channel (indices 12–16, 17–21, 22–26).
- Degree 3: 7 coefficients per channel (indices 27–33, 34–40, 41–47).

**Constants:**
The real spherical harmonics use the Condon–Shortley phase. Key normalization constants include:
- $C_0 = \frac{1}{\sqrt{4\pi}} \approx 0.28209479177387814$ [S6, S9, unofficial].
- $SH\_C_1 = \sqrt{\frac{3}{4\pi}} \approx 0.48860251190291987$ [S4, S7, unofficial].
- $SH\_C_2 = \sqrt{\frac{15}{4\pi}} \approx 1.0925484305920792$ [S4, S7, unofficial].
- $SH\_C_3 = \sqrt{\frac{7}{4\pi}} \approx 0.59004358992664352$ [S4, unofficial].

**Steps:**
1. **Normalize Direction:** Ensure `viewdirs` are normalized unit vectors $(x, y, z)$ [S5, S7, unofficial].
2. **Initialize Color:** Start with the DC term (Degree 0). For each Gaussian, extract coefficients $c_0, c_1, c_2$ (R, G, B) and compute:
   $$ \text{color} = C_0 \cdot [c_0, c_1, c_2] $$
   [S5, S7, S9, unofficial].
3. **Evaluate Higher Orders:** Add contributions from higher degrees based on the active SH degree $d$.
   - **Degree 1 (Linear):** Extract coefficients $c_3..c_5$ (R), $c_6..c_8$ (G), $c_9..c_{11}$ (B).
     $$ \text{color} \leftarrow \text{color} - SH\_C_1 \cdot [y \cdot c_{3:5}, -z \cdot c_{6:8}, x \cdot c_{9:11}] $$
     Note: Signs depend on the specific SH basis definition used. Common implementation uses:
     $$ \text{color}_R \leftarrow \text{color}_R - SH\_C_1 \cdot y \cdot c_3 $$
     $$ \text{color}_G \leftarrow \text{color}_G + SH\_C_1 \cdot z \cdot c_6 $$
     $$ \text{color}_B \leftarrow \text{color}_B - SH\_C_1 \cdot x \cdot c_9 $$
     [S7, unofficial].
   - **Degree 2 (Quadratic):** Extract coefficients $c_{12}..c_{16}$ (R), $c_{17}..c_{21}$ (G), $c_{22}..c_{26}$ (B). Compute squared/cross terms: $xx=x^2, yy=y^2, zz=z^2, xy=xy, yz=yz, xz=xz$.
     $$ \text{color} \leftarrow \text{color} + SH\_C_2 \cdot [xy \cdot c_{12:16}, yz \cdot c_{17:21}, (2zz - xx - yy) \cdot c_{22:26}] $$
     Specifically [S5, S7, unofficial]:
     $$ \text{color}_R \leftarrow \text{color}_R + SH\_C_2 \cdot (xy \cdot c_{12} + yz \cdot c_{13} + (2zz - xx - yy) \cdot c_{14} + xz \cdot c_{15} + (xx - yy) \cdot c_{16}) $$
     (Similar for G and B with respective coefficients).
   - **Degree 3 (Cubic):** If $d \ge 3$, extract coefficients $c_{27}..c_{33}$ (R), etc.
     $$ \text{color} \leftarrow \text{color} + SH\_C_3 \cdot [\text{higher order terms}] $$
     [S5, S7, unofficial].
4. **Bias and Clamp:** Add a bias of 0.5 to the result and apply ReLU or clamp to $[0, 1]$ to handle negative values from the basis functions [S3, S5, S8, S7, unofficial].
   $$ \text{rgb} = \text{ReLU}(\text{color} + 0.5) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def eval_sh(degree: int, shs: torch.Tensor, viewdirs: torch.Tensor) -> torch.Tensor:
    """
    Evaluate spherical harmonics to compute RGB colors.
    
    Args:
        degree: Active SH degree (0, 1, 2, or 3).
        shs: Tensor of shape (N, 3, (degree+1)**2) or (N, 48) for degree 3.
             Stored as [R_coeffs, G_coeffs, B_coeffs].
        viewdirs: Tensor of shape (N, 3) with normalized (x, y, z) vectors.
        
    Returns:
        rgb: Tensor of shape (N, 3) with RGB colors in [0, 1].
    """
    # Normalize viewdirs
    viewdirs = viewdirs / viewdirs.norm(dim=1, keepdim=True)
    x, y, z = viewdirs[:, 0], viewdirs[:, 1], viewdirs[:, 2]
    
    # Precompute powers
    xx, yy, zz = x*x, y*y, z*z
    xy, yz, xz = x*y, y*z, x*z
    
    # Constants
    C0 = 0.28209479177387814
    C1 = 0.48860251190291987
    C2 = 1.0925484305920792
    C3 = 0.59004358992664352
    
    # Initialize color with DC term (Degree 0)
    # shs shape: (N, 3, num_coeffs). First 1 coeff per channel is DC.
    dc_r = shs[:, 0, 0]
    dc_g = shs[:, 1, 0]
    dc_b = shs[:, 2, 0]
    
    color_r = C0 * dc_r
    color_g = C0 * dc_g
    color_b = C0 * dc_b
    
    if degree >= 1:
        # Degree 1: 3 coeffs per channel
        # Indices 1, 2, 3 for each channel
        c1_r = shs[:, 0, 1]
        c1_g = shs[:, 1, 1]
        c1_b = shs[:, 2, 1]
        
        c2_r = shs[:, 0, 2]
        c2_g = shs[:, 1, 2]
        c2_b = shs[:, 2, 2]
        
        c3_r = shs[:, 0, 3]
        c3_g = shs[:, 1, 3]
        c3_b = shs[:, 2, 3]
        
        # Basis functions for degree 1 (Condon-Shortley phase)
        # Y_1^-1 ~ -y, Y_1^0 ~ z, Y_1^1 ~ x
        # Note: Signs may vary by implementation. Standard GLTF/3DGS often uses:
        color_r -= C1 * y * c1_r
        color_g += C1 * z * c2_g
        color_b -= C1 * x * c3_b
        
    if degree >= 2:
        # Degree 2: 5 coeffs per channel
        # Indices 4, 5, 6, 7, 8 for each channel
        c4_r = shs[:, 0, 4]
        c5_r = shs[:, 0, 5]
        c6_r = shs[:, 0, 6]
        c7_r = shs[:, 0, 7]
        c8_r = shs[:, 0, 8]
        
        c4_g = shs[:, 1, 4]
        c5_g = shs[:, 1, 5]
        c6_g = shs[:, 1, 6]
        c7_g = shs[:, 1, 7]
        c8_g = shs[:, 1, 8]
        
        c4_b = shs[:, 2, 4]
        c5_b = shs[:, 2, 5]
        c6_b = shs[:, 2, 6]
        c7_b = shs[:, 2, 7]
        c8_b = shs[:, 2, 8]
        
        # Basis functions for degree 2
        # Y_2^-2 ~ xy, Y_2^-1 ~ yz, Y_2^0 ~ 2zz-xx-yy, Y_2^1 ~ xz, Y_2^2 ~ xx-yy
        color_r += C2 * (xy * c4_r + yz * c5_r + (2*zz - xx - yy) * c6_r + xz * c7_r + (xx - yy) * c8_r)
        color_g += C2 * (xy * c4_g + yz * c5_g + (2*zz - xx - yy) * c6_g + xz * c7_g + (xx - yy) * c8_g)
        color_b += C2 * (xy * c4_b + yz * c5_b + (2*zz - xx - yy) * c6_b + xz * c7_b + (xx - yy) * c8_b)
        
    if degree >= 3:
        # Degree 3: 7 coeffs per channel
        # Indices 9..15 for each channel
        c9_r = shs[:, 0, 9]
        c10_r = shs[:, 0, 10]
        c11_r = shs[:, 0, 11]
        c12_r = shs[:, 0, 12]
        c13_r = shs[:, 0, 13]
        c14_r = shs[:, 0, 14]
        c15_r = shs[:, 0, 15]
        
        c9_g = shs[:, 1, 9]
        c10_g = shs[:, 1, 10]
        c11_g = shs[:, 1, 11]
        c12_g = shs[:, 1, 12]
        c13_g = shs[:, 1, 13]
        c14_g = shs[:, 1, 14]
        c15_g = shs[:, 1, 15]
        
        c9_b = shs[:, 2, 9]
        c10_b = shs[:, 2, 10]
        c11_b = shs[:, 2, 11]
        c12_b = shs[:, 2, 12]
        c13_b = shs[:, 2, 13]
        c14_b = shs[:, 2, 14]
        c15_b = shs[:, 2, 15]
        
        # Basis functions for degree 3
        # Y_3^-3 ~ y(3xx-yy), Y_3^-2 ~ xyz, Y_3^-1 ~ y(1-5zz), Y_3^0 ~ z(5zz-3)
        # Y_3^1 ~ x(1-5zz), Y_3^2 ~ z(xx-yy), Y_3^3 ~ x(xx-3yy)
        # Note: Signs and specific polynomial forms vary slightly by source.
        # Using standard real SH definitions with Condon-Shortley phase.
        
        term3_r = (y * (3*xx - yy) * c9_r + 
                   xy * z * c10_r + 
                   y * (1 - 5*zz) * c11_r + 
                   z * (5*zz - 3) * c12_r + 
                   x * (1 - 5*zz) * c13_r + 
                   z * (xx - yy) * c14_r + 
                   x * (xx - 3*yy) * c15_r)
        
        term3_g = (y * (3*xx - yy) * c9_g + 
                   xy * z * c10_g + 
                   y * (1 - 5*zz) * c11_g + 
                   z * (5*zz - 3) * c12_g + 
                   x * (1 - 5*zz) * c13_g + 
                   z * (xx - yy) * c14_g + 
                   x * (xx - 3*yy) * c15_g)
                   
        term3_b = (y * (3*xx - yy) * c9_b + 
                   xy * z * c10_b + 
                   y * (1 - 5*zz) * c11_b + 
                   z * (5*zz - 3) * c12_b + 
                   x * (1 - 5*zz) * c13_b + 
                   z * (xx - yy) * c14_b + 
                   x * (xx - 3*yy) * c15_b)
                   
        color_r += C3 * term3_r
        color_g += C3 * term3_g
        color_b += C3 * term3_b
        
    # Add bias and clamp
    rgb = torch.stack([color_r, color_g, color_b], dim=1)
    rgb = torch.relu(rgb + 0.5)
    
    return rgb
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating point precision errors can accumulate in higher-degree terms. Use single precision (float32) for real-time rendering, double precision (float64) for offline evaluation if necessary [S10].
- **Boundaries:** The output of SH evaluation can be negative. The bias of 0.5 and ReLU/clamping are essential to map to valid color ranges [S3, S8, unofficial].
- **Degenerate Input:** If `viewdirs` are zero vectors, normalization fails. Ensure `viewdirs` are non-zero before normalization [S5, unofficial].
- **Performance:** Implementations may ignore higher-degree coefficients (e.g., degree 3) for performance reasons, as they contribute less to the overall appearance [S10].

## Subtopics

### [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md)
Spherical Harmonics Rotation computes the new SH coefficients of a function after rotating the viewing coordinate system by $\pm \pi/2$ around the X-axis. It sets itself apart from general rotation methods by using precomputed analytic tables for specific $\pm 90^\circ$ rotations, avoiding expensive matrix multiplications or numerical integration.

### [Spherical Harmonics Encoder](4390-spherical-harmonics-encoder.md)
The Spherical Harmonics Encoder is a neural network module that encodes a 3D direction vector into spherical harmonics coefficients up to a specified degree. It solves the problem of representing directional information (view direction) in a differentiable, basis-function expansion for view-dependent rendering.

### [Spherical Harmonics Data Extraction](6960-spherical-harmonics-data-extraction.md)
Spherical Harmonics (SH) Data Extraction is the process of parsing SH coefficients from a 3D Gaussian Splatting model file (typically a PLY format) into a structured data layout suitable for GPU rendering. It solves the problem of mapping flat, property-based storage in the source file to the hierarchical texture arrays required by the rendering engine. It is distinct from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md), which computes color from these coefficients, and [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md), which transforms them for view-dependent effects.

### [Spherical Harmonics Decoding](7901-spherical-harmonics-decoding.md)
Spherical Harmonics (SH) Decoding is the process of reconstructing the RGB color of a 3D Gaussian splat from its stored SH coefficients and a specific viewing direction. It serves as the final color computation step in the Gaussian Splatting pipeline, translating directional frequency data into perceptual color. It is distinguished from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md) by its specific focus on unpacking and interpreting the raw coefficient storage format (often packed in GPU buffers) rather than just evaluating the basis functions, and from [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md) which handles the transformation of coefficients to camera space.

### [Spherical Harmonics Lighting Calculation](11516-spherical-harmonics-lighting-calculation.md)
Lighting is defined via real spherical harmonics with the Condon–Shortley phase. The zeroth-order coefficients are always required; higher order coefficients are optional. Each color channel has a separate coefficient, so for each degree $l$, there are $(2l + 1)$ coefficients.

### Spherical Harmonics Evaluation with Tone Mapping
A module that interprets network output as spherical harmonics coefficients to model view-dependent effects and camera-specific parameters, followed by tone mapping [S11]. The SH coefficients are the output of the last convolution of the neural network [S11]. It applies a tone mapper following Rückert et al. [RFS22] which models exposure time, white balance, sensor response, and vignetting [S11]. It uses spherical harmonics degree 2, corresponding to 27 input coefficients (9 coefficients per color channel) [S11].

## Operating characteristics
- **Time Complexity:** $O(N \cdot d^2)$, where $N$ is the number of Gaussians and $d$ is the SH degree. For fixed degree (typically 2 or 3), this is $O(N)$.
- **Memory Complexity:** $O(N \cdot d^2)$ to store the coefficients.
- **Measured Performance:** Degree 2 (9 coefficients per channel) is the standard for real-time rendering, providing a good balance between view-dependent detail and performance [S3, unofficial][S11]. Degree 3 adds significant computational cost for marginal visual improvement in many scenes [S10].
- **Scaling:** Scales linearly with the number of Gaussians. Parallelizable on GPU via tensor operations [S3, S8, unofficial].
- **Strengths:** Efficiently represents complex view-dependent effects with a small number of coefficients. Differentiable, allowing for gradient-based optimization [S1, S2, unofficial].
- **Limitations:** High-frequency view-dependent details may require higher degrees, increasing memory and compute. Aliasing can occur if the SH degree is too low for the scene's complexity [S10].
- **When to use:** When view-dependent appearance is important and real-time performance is required.
- **When not to use:** For static, view-independent colors (degree 0 is sufficient). For extremely high-frequency effects that cannot be captured by low-degree SH.

## Background
Spherical Harmonics have been used in computer graphics for lighting and rendering for decades. In the context of 3D Gaussian Splatting, they were adopted to represent view-dependent colors efficiently. The method is based on the expansion of functions on the sphere into orthogonal basis functions [S9, unofficial]. The specific implementation in Gaussian Splatting uses real spherical harmonics with the Condon–Shortley phase [S10]. The use of SH in this context was popularized by the original 3D Gaussian Splatting paper and subsequent implementations [S1, S3, S5, S7, unofficial].

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
