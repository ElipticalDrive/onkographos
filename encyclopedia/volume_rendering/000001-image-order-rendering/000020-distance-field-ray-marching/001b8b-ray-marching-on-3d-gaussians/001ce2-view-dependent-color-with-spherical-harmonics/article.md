# View-dependent color with Spherical Harmonics

Also called: Spherical Harmonics (SH) for View-Dependent Color, Spherical Harmonics Color Evaluation, Spherical Harmonics Rendering, SH Render, Rendering with Spherical Harmonics, Spherical Harmonic Colour Model

## Overview
This method represents the view-dependent color of a 3D Gaussian primitive as a function of the viewing direction using spherical harmonics (SH) coefficients [S1, unofficial]. It solves the problem of storing directional color variation (such as specular highlights and reflections) compactly without requiring per-ray neural network queries [S1, unofficial]. Unlike simple point colors or view-independent diffuse shading, this model captures glossy effects by evaluating a fixed set of orthogonal basis functions on the unit sphere, allowing each Gaussian to exhibit different colors from different angles [S1, unofficial]. It builds on [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) by providing the color value $C_c(\mathbf{d})$ used during the compositing step [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** For each Gaussian $i$, the normalized viewing direction vector $\mathbf{d}_i$ from the camera center to the Gaussian's mean position, and the stored SH coefficients [S1, unofficial].
*   **Output:** An RGB color vector $\mathbf{C}_i(\mathbf{d}_i) = [R_i, G_i, B_i]^T$ evaluated at direction $\mathbf{d}_i$ [S1, unofficial].

**Data Structures**
Each Gaussian stores 48 SH coefficients: 16 per color channel (Red, Green, Blue) [S1, unofficial]. The coefficients are stored contiguously in memory [S3, unofficial]. The layout is: indices 0–2 are the degree-0 (DC) coefficients for R, G, B respectively; indices 3–47 contain the remaining coefficients for degrees 1, 2, and 3 [S3, unofficial]. The coefficients are scalars $k_{\ell,c}^m$ corresponding to basis functions $Y_{\ell}^m$ [S1, unofficial].

**Spherical Harmonic Basis Constants**
The evaluation uses fixed normalization constants [S3, unofficial]:
*   $SH\_C0 = 0.28209479177387814$
*   $SH\_C1 = 0.4886025119029199$
*   Degree 2 and 3 constants are fixed polynomials but exact values are not given in the sources [S3, unofficial]. Standard values are used [G].

**Numbered Steps**
1.  **Compute Viewing Direction:** For each Gaussian, compute the normalized direction vector $\mathbf{d} = [x, y, z]^T$ from the camera center to the Gaussian's mean position [S1, unofficial].
2.  **Initialize Color:** Initialize the output color vector $\mathbf{C}$ with the DC component (degree 0) scaled by $SH\_C0$ [S3, unofficial].
    $$ \mathbf{C} = SH\_C0 \cdot [k_{0,R}^0, k_{0,G}^0, k_{0,B}^0]^T $$
3.  **Evaluate Degree 1 (if active):** If the SH degree is $\ge 1$, add contributions from the first-order coefficients using the direction components $x, y, z$ scaled by $SH\_C1$ [S3, unofficial]. The specific linear combination follows the convention in the reference implementation: `color = color - SH_C1 * y * c1 + SH_C1 * z * c2 - SH_C1 * x * c3` [S3, unofficial].
4.  **Evaluate Degree 2 (if active):** If the SH degree is $\ge 2$, add contributions from second-order coefficients using quadratic direction terms ($xy, yz, 2z^2-x^2-y^2, xz, x^2-y^2$) scaled by degree-2 normalization constants [S3, unofficial].
5.  **Evaluate Degree 3 (if active):** If the SH degree is $\ge 3$, add contributions from third-order coefficients using cubic direction terms [S3, unofficial].
6.  **Apply Bias and Clamping:** Apply a bias of $+0.5$ and a ReLU activation to the resulting RGB values to ensure non-negative color values [S4, unofficial].
    $$ \mathbf{C}_{final} = \text{ReLU}(\mathbf{C} + 0.5) $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Normalization constants from [S3, unofficial]
SH_C0 = 0.28209479177387814
SH_C1 = 0.4886025119029199
# Standard SH constants for degrees 2 and 3 [G]
SH_C2_0 = 1.0925484305920792
SH_C2_1 = -0.3153915652525201
SH_C2_2 = 0.5462742152960396
SH_C2_3 = -0.3153915652525201
SH_C2_4 = 0.10846220610920792
SH_C3_0 = -0.5900435899266435
SH_C3_1 = 2.890611442640554
SH_C3_2 = -0.4570457994644658
SH_C3_3 = 0.3731763325901154
SH_C3_4 = -0.4570457994644658
SH_C3_5 = 1.445305721320277
SH_C3_6 = -0.5900435899266435

def eval_sh(sh_coeffs, x, y, z, max_degree=3):
    """
    Evaluates view-dependent color for a single Gaussian.
    sh_coeffs: list of 48 floats [DC(3), D1(9), D2(15), D3(21)]
    x, y, z: components of the normalized viewing direction vector
    max_degree: integer, maximum SH degree active (0-3)
    Returns: list of 3 floats [R, G, B]
    """
    # Degree 0 (DC)
    r, g, b = sh_coeffs[0], sh_coeffs[1], sh_coeffs[2]
    r *= SH_C0
    g *= SH_C0
    b *= SH_C0
    
    if max_degree >= 1:
        # Degree 1 coefficients for R, G, B
        r1, r2, r3 = sh_coeffs[3], sh_coeffs[4], sh_coeffs[5]
        g1, g2, g3 = sh_coeffs[6], sh_coeffs[7], sh_coeffs[8]
        b1, b2, b3 = sh_coeffs[9], sh_coeffs[10], sh_coeffs[11]
        # Basis functions: -y, z, -x scaled by SH_C1 [S3, unofficial]
        r += SH_C1 * (-y * r1 + z * r2 - x * r3)
        g += SH_C1 * (-y * g1 + z * g2 - x * g3)
        b += SH_C1 * (-y * b1 + z * b2 - x * b3)
        
    if max_degree >= 2:
        # Degree 2 coefficients for R, G, B
        r1, r2, r3, r4, r5 = sh_coeffs[12:17]
        g1, g2, g3, g4, g5 = sh_coeffs[17:22]
        b1, b2, b3, b4, b5 = sh_coeffs[22:27]
        # Standard SH2 basis functions [G]
        xy, yz, z2 = x * y, y * z, z * z
        x2, y2 = x * x, y * y
        r += SH_C2_0 * xy * r1 + SH_C2_1 * yz * r2 + SH_C2_2 * (2 * z2 - x2 - y2) * r3 + SH_C2_3 * xz * r4 + SH_C2_4 * (x2 - y2) * r5
        g += SH_C2_0 * xy * g1 + SH_C2_1 * yz * g2 + SH_C2_2 * (2 * z2 - x2 - y2) * g3 + SH_C2_3 * xz * g4 + SH_C2_4 * (x2 - y2) * g5
        b += SH_C2_0 * xy * b1 + SH_C2_1 * yz * b2 + SH_C2_2 * (2 * z2 - x2 - y2) * b3 + SH_C2_3 * xz * b4 + SH_C2_4 * (x2 - y2) * b5
        
    if max_degree >= 3:
        # Degree 3 coefficients for R, G, B
        r1, r2, r3, r4, r5, r6, r7 = sh_coeffs[27:34]
        g1, g2, g3, g4, g5, g6, g7 = sh_coeffs[34:41]
        b1, b2, b3, b4, b5, b6, b7 = sh_coeffs[41:48]
        # Standard SH3 basis functions [G]
        x2, y2, z2 = x*x, y*y, z*z
        r += SH_C3_0 * y * (3 * x2 - y2) * r1 + SH_C3_1 * x * y * z * r2 + SH_C3_2 * y * (4 * z2 - x2 - y2) * r3 + SH_C3_3 * x * z * (2 * z2 - 3 * x2 - 3 * y2) * r4 + SH_C3_4 * y * (4 * z2 - 3 * x2 - 3 * y2) * r5 + SH_C3_5 * z * (2 * z2 - 3 * x2 - 3 * y2) * r6 + SH_C3_6 * x * (x2 - 3 * y2) * r7
        g += SH_C3_0 * y * (3 * x2 - y2) * g1 + SH_C3_1 * x * y * z * g2 + SH_C3_2 * y * (4 * z2 - x2 - y2) * g3 + SH_C3_3 * x * z * (2 * z2 - 3 * x2 - 3 * y2) * g4 + SH_C3_4 * y * (4 * z2 - 3 * x2 - 3 * y2) * g5 + SH_C3_5 * z * (2 * z2 - 3 * x2 - 3 * y2) * g6 + SH_C3_6 * x * (x2 - 3 * y2) * g7
        b += SH_C3_0 * y * (3 * x2 - y2) * b1 + SH_C3_1 * x * y * z * b2 + SH_C3_2 * y * (4 * z2 - x2 - y2) * b3 + SH_C3_3 * x * z * (2 * z2 - 3 * x2 - 3 * y2) * b4 + SH_C3_4 * y * (4 * z2 - 3 * x2 - 3 * y2) * b5 + SH_C3_5 * z * (2 * z2 - 3 * x2 - 3 * y2) * b6 + SH_C3_6 * x * (x2 - 3 * y2) * b7
        
    # Apply bias and ReLU [S4, unofficial]
    r = max(0.0, r + 0.5)
    g = max(0.0, g + 0.5)
    b = max(0.0, b + 0.5)
    
    return [r, g, b]
```

**Parameters**
*   **Maximum SH Degree ($L$):** Typically 3 [S1, unofficial]. This results in $(L+1)^2 = 16$ basis functions per color channel, totaling 48 coefficients per Gaussian [S1, unofficial].
*   **Training Schedule:** Optimization begins with only the degree-0 band active. One further band is unlocked every 1,000 iterations until all four bands (degrees 0-3) are active [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Basis Functions are Fixed:** The basis functions $Y_{\ell}^m(\mathbf{d})$ are fixed polynomials of the direction components and are never updated during training. Only the coefficients $k$ are learned via gradient descent [S1, unofficial].
*   **Direction Normalization:** The direction vector $\mathbf{d}$ must be normalized before evaluation [S1, unofficial].
*   **Precision:** Standard single-precision arithmetic is used [G].

## Subtopics

### [Progressive View-Dependent Color Baking](2834-progressive-view-dependent-color-baking.md)
An optimization that reduces memory bandwidth pressure by evaluating view-dependent colors over multiple frames rather than every frame, trading immediate visual fidelity for rendering performance.

## Operating characteristics
*   **Time Complexity:** The evaluation of SH coefficients is $O(1)$ per Gaussian per frame, involving a fixed number of multiplications and additions [S1, unofficial]. It is significantly faster than querying a neural network [S1, unofficial].
*   **Memory Complexity:** Each Gaussian stores 48 additional float values (48 $\times$ 4 bytes = 192 bytes) for the SH coefficients [S1, unofficial]. This is a constant overhead per primitive [S1, unofficial].
*   **Measured Performance:** The method contributes to the overall rendering speed of 3DGS, which reaches 134 fps on the Mip-NeRF360 benchmark [S1, unofficial]. The SH evaluation itself is described as costing "almost nothing at display time" [S1, unofficial].
*   **Strengths:** Captures view-dependent effects like highlights and reflections, which simple point colors cannot [S1, unofficial]. It is differentiable and can be optimized via gradient descent [S1, unofficial].
*   **Limitations:** High-degree SH (degree 3) increases memory usage significantly (48 coefficients vs 3 for DC) [S1, unofficial]. It may struggle with very sharp, high-frequency specularities compared to other representations, though degree 3 is generally sufficient for many scenes [S1, unofficial].
*   **When to Use:** When view-dependent appearance (specularities, reflections) is important for photorealism [S1, unofficial].
*   **When Not to Use:** When memory bandwidth is extremely constrained and view-dependent effects are not critical (use DC only) [S1, unofficial].

## Background
*   **Origin:** The method was introduced in the 3D Gaussian Splatting paper (Kerbl et al., 2023) [S1, unofficial]. It repurposes spherical harmonics from irradiance environment maps to represent the color leaving a Gaussian [S1, unofficial].
*   **History:** Spherical harmonics are a well-known orthogonal basis for functions on the unit sphere, analogous to Fourier series [S1, unofficial]. The specific application to 3DGS involves truncating the series at degree 3 and learning the coefficients via gradient descent [S1, unofficial].
*   **Usage:** It is a core component of the 3D Gaussian Splatting rendering pipeline, enabling photorealistic novel view synthesis [S1, unofficial].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S3] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S4] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
