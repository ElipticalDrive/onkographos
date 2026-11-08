# Inverse Rendering with Gaussians (GS-IR / GIR)

Also called: GIR, GS-IR, GS-IR / GIR

## Overview
Inverse Rendering with Gaussians (GS-IR / GIR) is a method that extends 3D Gaussian Splatting to recover scene materials and lighting by solving the inverse rendering problem. It addresses the limitation of standard 3DGS, which only models direct view-dependent appearance, by estimating physically based parameters such as albedo, roughness, and environment lighting. It sets itself apart from sibling methods like [BRDF and Relighting Extensions](6227-brdf-and-relighting-extensions.md) by specifically employing a split-sum approximation to handle indirect illumination, allowing for more accurate material estimation in complex lighting conditions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images with known camera poses, initial 3D Gaussian primitives (position, covariance, opacity, base color), and an environment map (or spherical harmonics coefficients) representing the lighting.
*   **Outputs:** Optimized Gaussian attributes including surface normals, albedo, roughness, and the lighting environment map.

**Data Structures:**
The method builds upon the [GaussianSet Data Structure](2074-gaussianset-data-structure.md). Each Gaussian $i$ stores:
*   $\mathbf{p}_i$: 3D position.
*   $\mathbf{R}_i$: Rotation matrix (from quaternion).
*   $\mathbf{s}_i$: Scaling vector.
*   $\alpha_i$: Opacity.
*   $\mathbf{c}_{i, \text{diffuse}}$: Diffuse albedo (RGB).
*   $\rho_i$: Roughness (scalar).
*   $\mathbf{n}_i$: Surface normal (derived from $\mathbf{R}_i$).

**Method Steps:**
1.  **Normal Estimation:** For each Gaussian, the surface normal $\mathbf{n}_i$ is derived from the rotation matrix $\mathbf{R}_i$. Typically, the normal corresponds to the direction of the Gaussian's shortest axis or the z-axis in the local frame, transformed to world space: $\mathbf{n}_i = \mathbf{R}_i \mathbf{z}_{local}$ [S2].
2.  **Split-Sum Approximation:** To handle indirect lighting efficiently, the rendering equation is split into direct and indirect components. The indirect component is approximated using a split-sum technique, where the integration of the BRDF against the environment map is separated [S1, unofficial].
3.  **Differentiable Rendering:** Rays are cast through the Gaussian set. For each hit, the color is computed using a microfacet BRDF model:
    $$
    C = \int_{\Omega} f_r(\omega_i, \omega_o) L_i(\omega_i) (\mathbf{n} \cdot \omega_i) d\omega_i
    $$
    where $f_r$ is the BRDF (dependent on albedo and roughness), $L_i$ is the incident radiance from the environment map, and $\omega_i, \omega_o$ are incident and view directions. The split-sum approximation allows this to be computed as a sum of a pre-filtered environment map term and a direct lighting term [S1, unofficial].
4.  **Loss Computation:** A photometric loss $\mathcal{L}_{rgb}$ is computed between the rendered image and the ground truth. Additional regularization terms may be applied to smooth roughness and albedo fields [S2].
5.  **Optimization:** Gradients are backpropagated through the differentiable renderer to update Gaussian attributes (albedo, roughness, normals) and the environment map parameters using an optimizer like Adam.

**Parameters:**
*   Learning rate for Gaussian attributes: Not given in the sources.
*   Learning rate for environment map: Not given in the sources.
*   Roughness prior: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Noise:** The method can introduce substantial noise in geometry, material, and lighting estimates if not properly regularized [S2].
*   **Normal Ambiguity:** Deriving normals from Gaussian axes can be ambiguous for highly anisotropic or flat Gaussians [S2].

## Subtopics

### Inverse Rendering with 3DGS
Methods adapting inverse rendering techniques to 3DGS to handle specular reflections by simplifying the classical rendering equation [S2]. These approaches rely on accurate normal estimation, which can be derived from computed depth maps [LZF* 24] or using the shortest direction axis of Gaussians [JTL* 24, YHZ24]. The different lighting conditions are then encoded via integrated shading functions [S2]. While these 3DGS-based methods achieve rapid convergence, they still introduce substantial noise in geometry, material, and lighting, which remains a limitation for high-fidelity rendering [S2].

## Operating characteristics
*   **Time Complexity:** Faster than NeRF-based inverse rendering methods due to the efficiency of Gaussian splatting, but slower than standard 3DGS due to the complexity of BRDF evaluation and environment map integration [S2].
*   **Memory Complexity:** Increases with the number of Gaussian attributes (albedo, roughness, normals) and the resolution of the environment map representation.
*   **Strengths:** Rapid convergence compared to volume-based inverse rendering; handles specular reflections explicitly [S2].
*   **Limitations:** Substantial noise in recovered geometry, material, and lighting; sensitive to normal estimation errors [S2].
*   **When to use:** When high-fidelity relighting and material recovery are needed with faster training times than NeRF-based methods.
*   **When not to use:** When extreme geometric precision or noise-free material maps are required without significant post-processing or regularization.

## Background
Inverse rendering with Gaussians emerged as an adaptation of classical inverse rendering techniques to the 3D Gaussian Splatting representation. Early attempts to integrate inverse rendering with NeRFs suffered from extremely time-consuming optimization and slow rendering speeds [S2]. Recent methods [JTL* 24, LZF* 24, YHZ24] leveraged the efficiency of 3DGS to accelerate the process, though they simplified the rendering equation to handle specular reflections [S2]. The specific GS-IR / GIR approach utilizes a split-sum approximation to handle indirect light, addressing the limitations of SH-only representations which are too low-bandwidth for sharp reflections [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
