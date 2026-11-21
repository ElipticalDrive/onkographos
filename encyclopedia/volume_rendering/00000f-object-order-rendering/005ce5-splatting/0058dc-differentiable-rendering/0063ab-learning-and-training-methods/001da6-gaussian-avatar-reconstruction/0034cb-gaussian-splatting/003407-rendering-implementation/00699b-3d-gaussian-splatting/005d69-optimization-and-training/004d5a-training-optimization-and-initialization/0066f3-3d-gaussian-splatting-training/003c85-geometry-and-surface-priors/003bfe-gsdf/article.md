# GSDF
Also called: Gaussian SDF, Gaussian Signed Distance Function

## Overview
GSDF is a training optimization technique that aligns the discrete 3D Gaussian representation with a continuous Signed Distance Function (SDF) prior to enhance geometry reconstruction quality [S1]. It sets itself apart from sibling methods like [NeuSG](9172-neusg.md) by explicitly transferring surface properties from a neural SDF to the Gaussians, encouraging Gaussian centers to lie on the zero-level set and normals to align with the SDF gradient [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian parameters (centers $\mathbf{c}_i$, covariances $\Sigma_i$, opacities $\alpha_i$, colors $\mathbf{c}_i$), a pre-trained or concurrently trained neural SDF field $S(\mathbf{x})$, and input images with camera poses.
*   **Outputs:** Optimized 3D Gaussians suitable for high-fidelity rendering and mesh extraction.

**Data Structures:**
*   **Neural SDF:** A multi-layer perceptron (MLP) mapping 3D coordinates $\mathbf{x}$ to a scalar distance value $S(\mathbf{x})$ [S2, unofficial].
*   **Gaussian Attributes:** Standard 3DGS attributes. The covariance matrix $\Sigma_i$ defines the orientation and scale, from which the normal direction is derived [S1].

**Method Steps:**
1.  **Render:** Perform standard differentiable volume rendering to generate predicted images and depth maps [S1].
2.  **Evaluate SDF:** For each Gaussian center $\mathbf{c}_i$, query the neural SDF to obtain $S(\mathbf{c}_i)$ [S1].
3.  **Compute Geometric Loss:** Calculate the loss encouraging Gaussians to lie on the surface:
    $$ \mathcal{L}_{geo} = \sum_i |S(\mathbf{c}_i)| $$
    This term minimizes the signed distance of Gaussian centers to the zero-level set [S1].
4.  **Compute Normal Loss:** Estimate the surface normal from the SDF gradient $\nabla S(\mathbf{c}_i)$ and the Gaussian orientation. The Gaussian normal $\mathbf{n}_i$ is derived from the eigenvector of the covariance matrix $\Sigma_i$ corresponding to the smallest eigenvalue (or the rotation quaternion's forward vector) [S1]. The loss minimizes the angular difference:
    $$ \mathcal{L}_{norm} = \sum_i (1 - \mathbf{n}_i \cdot \frac{\nabla S(\mathbf{c}_i)}{\|\nabla S(\mathbf{c}_i)\|}) $$
    This ensures the Gaussian orientation aligns with the SDF surface normal [S1].
5.  **Optimize:** Minimize the total loss:
    $$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{geo} \mathcal{L}_{geo} + \lambda_{norm} \mathcal{L}_{norm} $$
    where $\mathcal{L}_{rgb}$ is the photometric rendering loss [S1].

**Parameters:**
*   $\lambda_{geo}, \lambda_{norm}$: Hyperparameters balancing geometric consistency against photometric fidelity [S1]. Typical values are not specified in the sources.

**Edge Cases:**
*   **Degenerate Gaussians:** If Gaussians become flat or elongated incorrectly, the normal estimation from covariance may become unstable [S1].
*   **SDF Accuracy:** The method relies on the quality of the underlying neural SDF. Poor SDF training leads to incorrect geometric priors [S2, unofficial].

## Operating characteristics
*   **Complexity:** Adds overhead of querying the neural SDF MLP for each Gaussian center during training.
*   **Strengths:** Recovers more detailed mesh structures compared to standard 3DGS [S2, unofficial]. Enhances geometry reconstruction quality [S1].
*   **Limitations:** Requires a reliable neural SDF prior. Performance depends on the accuracy of the SDF field [S2, unofficial].
*   **When to use:** When high-quality surface reconstruction and mesh extraction are required from 3DGS [S1][S2, unofficial].

## Background
GSDF was introduced to leverage neural signed distance functions (SDFs) for surface reconstruction, inspired by similar approaches in NeRF-based methods [S2, unofficial]. It was extended in subsequent works like 2DGS, which replaced 3D Gaussians with 2D Gaussians for more accurate ray-splat intersection [S2, unofficial]. It is closely related to other methods such as 3DGSR and GSPull that also encourage consistency between SDF and 3DGS [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
