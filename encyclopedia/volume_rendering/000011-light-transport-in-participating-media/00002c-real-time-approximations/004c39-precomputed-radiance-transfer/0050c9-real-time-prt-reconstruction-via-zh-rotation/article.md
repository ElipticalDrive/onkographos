# Real-time PRT Reconstruction via ZH Rotation

Also called: Per-pixel PRT reconstruction, ZH to SH projection

## Overview
This method reconstructs lighting in real-time by rotating precomputed Precomputed Radiance Transfer (PRT) values, stored in the Zernike Harmonic (ZH) basis, to match the local surface frame, projecting them back into the Spherical Harmonic (SH) basis, and computing the dot product with the lighting environment projected into SH [S1, unofficial]. It sets itself apart from standard PRT by enabling per-pixel rotation of the transfer function on the GPU, allowing for dynamic surface orientations without the prohibitive cost of per-sample SH rotation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Precomputed PRT coefficients stored in ZH basis textures; lighting environment projected into SH basis; interpolated surface normal per pixel.
*   **Output:** Final lit color (radiance) per pixel.

**Data Structures:**
*   **ZH Coefficients:** Stored in textures. The specific bit-packing or layout is not detailed, but they are accessed as textures on the GPU [S1, unofficial].
*   **Rotation Representation:** Due to memory constraints of storing 4x4 rotation matrices for ZH coefficients, alternative representations are considered: polar coordinates, unit vectors, or quaternions [S1, unofficial].

**Steps:**
1.  **Vertex Shader:** Pass the interpolated surface normal to define the local frame [S1, unofficial].
2.  **Pixel Shader - Rotation:** Rotate the precomputed ZH transfer function to match the local surface frame defined by the normal [S1, unofficial].
3.  **Pixel Shader - Projection:** Project the rotated ZH coefficients back into the SH basis [S1, unofficial].
4.  **Pixel Shader - Lighting:** Project the current lighting environment into the SH basis (precomputed or updated per frame) [S1, unofficial].
5.  **Pixel Shader - Dot Product:** Compute the dot product between the projected transfer function (in SH) and the lighting environment (in SH) to determine the final lighting [S1, unofficial].

**Formulas:**
*   The rotation of ZH functions is required to align the precomputed transfer with the local surface frame. The exact mathematical formula for ZH rotation is not provided in the sources, but it is noted as being much faster than SH rotation [S1, unofficial].
*   The projection from ZH back to SH is referenced as "equation 2.7" in the source, but the formula itself is not included in the provided text [S1, unofficial].
*   The minimization method for projecting into ZH is referenced as "equation 2.10" in the source, but the formula itself is not included in the provided text [S1, unofficial].

**Parameters:**
*   **SH Order:** Not specified in the sources.
*   **ZH Order:** Not specified in the sources.

**Edge Cases and Pitfalls:**
*   **Rotation Representation Cost:** Storing 4x4 rotation matrices for ZH coefficients is too costly in memory [S1, unofficial].
*   **Polar Coordinates:** Using polar coordinates for rotation representation causes visual artifacts near the poles (drop angle 0 or $\pi$) [S1, unofficial].
*   **Precision:** Not specified in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(position, normal):
    # Pass interpolated normal to pixel shader
    return normal

# Pixel Shader
def pixel_shader(normal, zh_coefficients_texture, lighting_sh_coefficients):
    # 1. Define local frame from normal
    local_frame = normal
    
    # 2. Rotate ZH coefficients to local frame
    # Note: Exact ZH rotation formula not provided in sources [S1, unofficial]
    # Alternative representations (unit vectors, quaternions) are preferred over 4x4 matrices due to memory [S1, unofficial]
    rotated_zh = rotate_zh_to_local_frame(zh_coefficients_texture, local_frame)
    
    # 3. Project rotated ZH back to SH basis
    # Note: Exact projection formula (referenced as eq 2.7) not provided in sources [S1, unofficial]
    sh_coefficients = project_zh_to_sh(rotated_zh)
    
    # 4. Compute dot product with lighting environment (already in SH)
    # Note: Lighting environment projection into SH is assumed to be done elsewhere or per-frame [S1, unofficial]
    final_color = dot_product(sh_coefficients, lighting_sh_coefficients)
    
    return final_color
```

## Subtopics

### Local, Deformable Precomputed Radiance Transfer
LDPRT is a variant of PRT that computes the transfer function of the local surface (in tangent space) rather than the entire entity, allowing for real-time deformation while capturing local effects like shadows from surface bumps [S1, unofficial]. It is chosen when global illumination effects are not required, but real-time deformation is [S1, unofficial].
*   **Precomputation:** Local light transport is precomputed per-vertex and projected into the SH basis [S1, unofficial].
*   **Storage:** The SH coefficients are projected into the ZH basis using a minimization method (equation 2.10) and stored with the model [S1, unofficial].
*   **Runtime:** The lighting environment is projected into the SH basis. The stored ZH vectors are projected back into SH (using equation 2.7) and evaluated with the lighting environment [S1, unofficial].
*   **Rotation:** The local transfer function is rotated per-sample into global space before evaluation using fast ZH rotation, avoiding the expense of per-sample SH rotation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Per-pixel rotation and projection are performed in the pixel shader. ZH rotation is noted as being much faster than SH rotation, making it feasible for real-time per-pixel computation [S1, unofficial].
*   **Memory Complexity:** Storing 4x4 rotation matrices for ZH coefficients is too costly in memory, necessitating more efficient rotation representations (unit vectors, quaternions) [S1, unofficial].
*   **Strengths:** Enables real-time deformation; captures local effects like shadows from surface bumps; faster per-sample rotation than SH [S1, unofficial].
*   **Limitations:** Does not capture global illumination effects; restrictive environment where deformation does not greatly affect light transport [S1, unofficial].
*   **When to Use:** When real-time deformation is required and local surface effects (bumps, wrinkles) are important, but global illumination is not [S1, unofficial].
*   **When Not to Use:** When global illumination effects are critical [S1, unofficial].

## Background
This method is described in the context of a volumetric approach to rendering microgeometry using PRT, specifically referencing work by Joan et al. (2005) and the thesis of John Kloetzli [S1, unofficial]. It builds on the concepts of regular PRT but adapts them for local surface transfer and real-time deformation [S1, unofficial].

## Sources
[S1] A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbcm.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
