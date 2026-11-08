# 3D Gaussian Splatting with Reflection and Relighting

Also called: Gaussian-Shader, 3DGS-DR, IRGS

## Overview
This method extends 3D Gaussian Splatting (3DGS) by incorporating classical reflection and shading concepts to address limitations with highly reflective surfaces and enable dynamic relighting [S1]. Unlike standard 3DGS, which bakes lighting into spherical harmonics via an emission-absorption model, this approach integrates secondary rays (reflections and refractions) spawned upon primary ray interactions to model interreflections [S1]. It leverages the Gaussian kernel's shortest axis for natural normal estimation, offering a balance between physically based rendering (PBR) accuracy and training efficiency [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\{G_i\}$, each defined by position $\mathbf{p}_i$, covariance $\Sigma_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), and base appearance parameters. Additional parameters include BRDF properties (albedo, roughness), environment maps for direct lighting, and spherical harmonics coefficients for indirect lighting [S1].
*   **Outputs:** Rendered pixel colors $\hat{C}(\mathbf{r})$ for given camera poses and lighting conditions.

**Data Structures:**
Each Gaussian $G_i$ stores:
1.  **Position:** $\mathbf{p}_i \in \mathbb{R}^3$.
2.  **Covariance:** $\Sigma_i \in \mathbb{R}^{3 \times 3}$, decomposed into rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$. The normal direction is approximated by the shortest axis of the Gaussian kernel [S2].
3.  **Appearance:**
    *   Base color/albedo $\mathbf{c}_i$.
    *   Roughness $\alpha_i$.
    *   Spherical Harmonics (SH) coefficients $\mathbf{c}_{i, \text{SH}}$ for indirect lighting [S1].
    *   Optional: Environment map indices or lighting parameters for direct illumination [S1].

**Numbered Steps:**
1.  **Ray Generation:** For each pixel, generate a primary ray $\mathbf{r}$ from the camera origin through the pixel center.
2.  **Gaussian Sorting:** Sort Gaussians along the ray $\mathbf{r}$ based on their depth or distance from the camera.
3.  **Primary Ray Rendering:**
    *   For each Gaussian $G_i$ in sorted order, compute the projected 2D covariance and opacity $\alpha_i$.
    *   Evaluate the base color contribution. If using a PBR model, compute the direct lighting contribution using the environment map and the estimated normal $\mathbf{n}_i$ (derived from the shortest axis of $\Sigma_i$) [S1, S2].
    *   Accumulate color and opacity using alpha compositing:
        $$ C_{\text{direct}} = \sum_i \alpha_i \mathbf{c}_i \prod_{j < i} (1 - \alpha_j) $$
4.  **Secondary Ray Spawning (Reflection/Refraction):**
    *   Upon interaction with a Gaussian (or if the Gaussian represents a reflective surface), spawn secondary rays (reflections or refractions) [S1].
    *   Trace these secondary rays to sample the environment map or other scene elements to compute indirect lighting or reflection contributions [S1].
    *   Combine direct and indirect lighting: $\mathbf{c}_i = \mathbf{c}_{\text{direct}} + \mathbf{c}_{\text{indirect}}$ [S1].
5.  **Final Compositing:** Apply the accumulated color and opacity to produce the final pixel color $\hat{C}(\mathbf{r})$.

**Formulas:**
*   **Normal Estimation:** The normal $\mathbf{n}_i$ is approximated by the eigenvector corresponding to the smallest eigenvalue of the covariance matrix $\Sigma_i$ [S2].
*   **Optimization Update:** The parameters $\lambda$ (including position, scale, rotation, BRDF parameters, and SH coefficients) are updated using gradient descent:
    $$ \Delta \lambda = M^{-1} \nabla_{\lambda} \mathcal{L} $$
    where $M$ is a preconditioner (e.g., Adam optimizer state) and $\mathcal{L}$ is the photometric loss [S2].

**Parameters:**
*   **Roughness:** Typically optimized per Gaussian or shared [S1].
*   **Environment Map:** High-resolution cubemap or spherical harmonics for direct lighting [S1].
*   **SH Degree:** Usually degree 3 or 4 for indirect lighting [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Ray tracing for secondary rays can be computationally expensive [S1].
*   **Boundaries:** Handling the boundaries of the environment map for reflection rays is critical [S1].
*   **Degenerate Input:** If a Gaussian is highly anisotropic, the normal estimation from the shortest axis may be unstable [S2].

## Operating characteristics
*   **Time Complexity:** Rendering is faster than NeRF-based PBR methods due to the explicit Gaussian representation, but slower than standard 3DGS due to secondary ray tracing [S1, S2].
*   **Memory Complexity:** Higher than standard 3DGS due to additional BRDF parameters and environment map storage [S1].
*   **Strengths:** High-quality rendering of reflective surfaces; enables dynamic relighting; efficient normal estimation [S1, S2].
*   **Limitations:** Slower than standard 3DGS; complex optimization of BRDF parameters [S1, S2].
*   **When to Use:** Scenes with highly reflective surfaces or when relighting is required [S1].
*   **When Not to Use:** Real-time applications requiring maximum speed without reflection effects [S1].

## Background
Standard 3DGS uses an emission-absorption model, baking lighting into spherical harmonics, which limits performance on reflective surfaces [S1]. Methods like Gaussian-Shader, 3DGS-DR, and IRGS were developed to incorporate classical reflection and shading concepts [S1]. 3DGS naturally offers good normal estimation via the shortest axis of the Gaussian kernel, facilitating efficient PBR [S2].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
