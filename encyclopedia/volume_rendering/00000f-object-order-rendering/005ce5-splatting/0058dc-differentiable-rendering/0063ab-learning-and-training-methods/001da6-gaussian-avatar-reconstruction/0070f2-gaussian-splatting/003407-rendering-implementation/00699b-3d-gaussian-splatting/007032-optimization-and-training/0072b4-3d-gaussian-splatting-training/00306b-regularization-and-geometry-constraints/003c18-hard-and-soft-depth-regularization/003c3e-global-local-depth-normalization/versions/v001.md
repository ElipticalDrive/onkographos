# Global-Local Depth Normalization


## Overview
Global-Local Depth Normalization is a depth regularization technique for 3D Gaussian Splatting that prevents scale-invariant losses from overlooking minor depth variations, thereby improving detailed geometry reshaping in texture-rich regions [S1]. It sets itself apart from standard scale-invariant losses by combining global scale preservation with local scale amplification, addressing the issue where existing methods align depth maps to a fixed global scale and ignore small local errors [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered depth map $\hat{D}$, ground truth depth map $D_{gt}$ (if available) or reference depth, and optimization hyperparameters.
*   **Outputs:** Gradient updates for 3D Gaussian parameters (centers, scales, opacities) guided by a combined global-local depth loss.

**Data Structures:**
*   Depth maps are treated as 2D tensors of floating-point values representing distance from the camera plane.
*   Patches are extracted from these maps for local normalization.

**Method Steps:**
1.  **Compute Global Normalized Depth Loss:** Calculate the depth error using a scale-invariant loss function that aligns the predicted depth $\hat{D}$ with the ground truth $D_{gt}$ (or a reference) to preserve absolute scene scale knowledge [S1]. Let this be $\mathcal{L}_{global}$ [S1].
2.  **Compute Local Normalized Depth Loss:** Extract local depth patches from $\hat{D}$ and $D_{gt}$. Apply local scale normalization to these patches to amplify and refocus the loss on small local depth errors [S1]. Let this be $\mathcal{L}_{local}$ [S1].
3.  **Combine Losses:** Sum the global and local normalized losses to form the total depth regularization loss $\mathcal{L}_{depth}$ [S1]:
    $$ \mathcal{L}_{depth} = \mathcal{L}_{global} + \mathcal{L}_{local} $$
4.  **Optimize:** Use $\mathcal{L}_{depth}$ to guide gradient updates during the optimization of 3D Gaussians, encouraging the learning of small local depth changes in a scale-invariant way [S1].

**Formulas:**
While the specific mathematical formulation of the global and local normalization functions is not explicitly detailed in the provided excerpts, the concept relies on:
*   **Global Scale:** Ensuring the overall magnitude of $\hat{D}$ matches the scene scale [S1].
*   **Local Scale:** Normalizing depth values within local patches to highlight variations [S1].

**Parameters:**
*   Weighting between $\mathcal{L}_{global}$ and $\mathcal{L}_{local}$: Not given in the sources.
*   Patch size for local normalization: Not given in the sources.

**Pitfalls:**
*   Standard scale-invariant depth losses align all depth maps to a fixed global scale, which ignores minor local depth changes [S1].
*   This leads to noisy primitive distributions and reconstruction failures in regions with intricate textures [S1].
*   Gaussian radiance fields are more sensitive to small depth errors compared to NeRF, exacerbating these issues [S1].

## Subtopics

### Normal Derivation and Optimization
A strategy to optimize Gaussian normals by guiding them with pseudo-normals derived from depth gradients, while smoothing them using Total Variation (TV) regularization to address noise and lack of multi-view consistency [S2].

**Definition:**
This variant uses the depth gradient to derive pseudo-normals $\hat{n}_{\hat{D}}$ which guide the optimization of normals $\hat{n}$ within the 3D Gaussians [S2]. It addresses the challenge of generating reasonable normals within the 3DGS-based framework [S2].

**Steps:**
1.  **Calculate Pseudo-Normals:** Compute the depth gradient normal $\hat{n}_{\hat{D}}$ from the generated depth map $\hat{D}$ [S2].
2.  **Compute Normal Penalty Loss:** Calculate the penalty loss $\mathcal{L}_{n-p}$ between the Gaussian normal $\hat{n}$ and the pseudo-normal $\hat{n}_{\hat{D}}$ [S2].
3.  **Compute TV Regularization:** Compute the TV normal regularization term $\text{TV}_{\text{normal}}$ to smooth $\hat{n}$ [S2].
4.  **Optimize:** Optimize the 3D Gaussians $\mathcal{G}$ (including normals $\hat{n}$) by minimizing the combined loss $\mathcal{L}_c + \mathcal{L}_n$ [S2].

**Formulas:**
*   Pseudo-normal calculation:
    $$ \hat{n}_{\hat{D}} = \nabla_{uv} \hat{D} $$
    where $\nabla_{uv}$ is the spatial gradient operator on the depth map [S2].
*   Normal penalty loss:
    $$ \mathcal{L}_{n-p} = \| \hat{n} - \hat{n}_{\hat{D}} \| $$
    This term aligns the Gaussian normal $\hat{n}$ with the depth gradient normal $\hat{n}_{\hat{D}}$ [S2].
*   Total normal loss:
    $$ \mathcal{L}_n = \mathcal{L}_{n-p} + \lambda_{n-TV} \text{TV}_{\text{normal}} $$
    This combines the penalty and TV smoothness regularization [S2].

**Depth Generation Context:**
The depth map $\hat{D}$ is computed via volumetric accumulation:
$$ \hat{D} = \sum_{i=1}^{N} \hat{w}_i d_i $$
where $d_i$ is the distance from the $i$-th 3D Gaussian to the image plane, and $\hat{w}_i$ are weights derived from opacity and transmittance [S2]. To avoid disc aliasing from peak selection, depth is treated as a linear interpolation of distances [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** DNGaussian (using this method) demonstrates competitive quality and superior details compared to state-of-the-art methods in sparse-view settings on LLFF, Blender, and DTU datasets [S1]. It achieves a 25× reduction in training time and over 3000× faster rendering speed compared to some baselines [S1].
*   **Strengths:** Prevents noisy primitive distributions in texture-rich regions; enhances detailed geometry reshaping; maintains absolute scale knowledge [S1].
*   **Limitations:** Standard scale-invariant losses fail to capture small local errors, leading to reconstruction failures in complex textures [S1].
*   **When to Use:** When optimizing 3D Gaussian Splatting for scenes with intricate textures and when standard scale-invariant depth losses result in noisy geometry [S1].
*   **When Not to Use:** When absolute scale preservation is not critical and local detail is less important than global structure (though the method is designed to handle both).

## Background
*   **Origin:** Introduced in DNGaussian (2024) to address the sensitivity of Gaussian radiance fields to small depth errors [S1].
*   **History:** Developed as an improvement over existing scale-invariant depth losses that overlook small losses [S1].
*   **Usage:** Used in sparse-view 3D Gaussian radiance field optimization [S1].

## Sources
[S1] DNGaussian: Optimizing Sparse-View 3D Gaussian Radiance Fields with Global-Local Depth Normalization, https://doi.org/10.1109/cvpr52733.2024.01963, fetched 2026-10-26
[S2] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
