# NRFF

Also called: Neural Radiance Feature Field

## Overview
NRFF is a method for encoding view-dependent specular effects in neural radiance fields by replacing standard positional encodings and roughness parameters with Anisotropic Spherical Gaussians (ASG) and a custom Rendering Equation Encoding (REE) [S1]. It distinguishes itself from Ref-NeRF by using multiscale tensor decomposition for spatial representation and predicting ASG features to model complex reflections, rather than relying on isotropic roughness or precomputed environment maps [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$; viewing direction $\mathbf{v}$ (or reflection direction $\hat{\omega}_r$); regularization coefficient $\beta$.
*   **Output:** Color $\mathbf{c}$, density $\sigma$, and feature vector $\mathbf{f}$ (implicitly handled via the ASG/REE pipeline).

**Data Structures:**
1.  **Spatial MLP:** Takes 3D position $\mathbf{x}$. Uses Multiscale Tensor Decomposition (MTD) instead of Positional Encoding (PE) [S1].
2.  **ASG Features:** A vector of 128 values representing $N = 8 \times 16$ lobes of ASGs on a unit sphere [S1].
3.  **ASG MLP:** Takes ASG features. Outputs:
    *   Bottleneck vector $\mathbf{a}$ of length 2.
    *   Scalars $\lambda$ and $\mu$ (must be $> 0$) [S1].
4.  **REE Vector:** An $8 \times 16 \times 2$ vector resulting from the REE function [S1].
5.  **Directional MLP:** Takes the REE vector and viewing direction. Outputs color and density [S1].

**Numbered Steps:**
1.  **Spatial Encoding:** For a query point $\mathbf{x}$, the Spatial MLP uses MTD to encode the scene geometry in a coarse-to-fine scheme [S1].
2.  **ASG Feature Prediction:** The Spatial MLP predicts a vector of 128 values corresponding to $N = 8 \times 16$ ASG lobes [S1].
3.  **Bottleneck Generation:** The ASG features are passed to an auxiliary ASG MLP. This network outputs:
    *   An anisotropic reflections bottleneck vector $\mathbf{a}$ of length 2.
    *   Two scalar coefficients $\lambda$ and $\mu$, which model the shape of the ASG in orthogonal directions [S1].
4.  **REE Computation:** Compute the Rendering Equation Encoding (REE) for the reflection direction $\hat{\omega}_r$ using the predicted parameters:
    $$
    \text{REE}(\hat{\omega}_r, \mathbf{a}, \lambda, \mu) = \sum_{i=0}^{N-1} a_i S(\hat{\omega}_r; \omega_i) \exp\left(-\lambda_i (\hat{\omega}_r \cdot \omega_{\lambda_i})^2 - \mu_i (\hat{\omega}_r \cdot \omega_{\mu_i})^2\right)
    $$
    where:
    *   $N = 8 \times 16 = 128$ is the number of lobes [S1].
    *   $a_i$ is the bottleneck coefficient for the $i$-th ASG [S1].
    *   $S(\hat{\omega}_r; \omega_i) = \max(\hat{\omega}_r \cdot \omega_i, 0)$ is a smooth term [S1].
    *   $\lambda_i, \mu_i > 0$ are bandwidth coefficients related to orthogonal axes $\omega_{\lambda_i}, \omega_{\mu_i}$ [S1].
    *   The set of directions $\omega_i$ covers 360 degrees (the unit sphere) [S1].
5.  **Directional Processing:** The resulting $8 \times 16 \times 2$ REE vector is ingested by the Directional MLP, replacing the roughness and Incident Direction Encoding (IDE) used in Ref-NeRF [S1].
6.  **Rendering:** The Directional MLP outputs the final color and density, which are composited along the ray using standard volume rendering integration [S1].

**Loss Function:**
NRFF uses the auxiliary loss $L_o$ from Ref-NeRF and adds a regularization term $L_f$ to enforce sparse feature representation:
$$
L_f = \frac{\beta}{M} \sum_{i=0}^{M-1} \frac{|F_\tau^i|}{\tau}
$$
where:
*   $M$ is the number of features in the density field representation [S1].
*   $F_\tau$ is the density field feature map [S1].
*   $\beta$ is a regularization coefficient [S1].
*   $\tau$ is a scaling factor (implied by context) [S1].

**Parameters:**
*   $N$: Number of ASG lobes, set to $8 \times 16$ [S1].
*   $\mathbf{a}$: Anisotropic reflections bottleneck vector, length 2 [S1].
*   $\lambda, \mu$: Scalar coefficients modeling ASG shape, constrained to be $> 0$ [S1].

**Edge Cases and Pitfalls:**
*   **Positivity Constraint:** $\lambda$ and $\mu$ must be strictly positive to ensure valid Gaussian bandwidths [S1].
*   **Sparsity:** The regularization term $L_f$ is critical for preventing overfitting and enhancing interpretability by removing redundant features [S1].

## Operating characteristics
Not given in the sources.

## Background
NRFF was proposed by Albert Barreiro, Roger Marı́, Rafael Redondo, Gloria Haro, Carles Bosch, and David Berga from Eurecat and Universitat Pompeu Fabra [S1]. It was submitted on 2024–07–09, accepted on 2025–01–17, and published in *Image Processing On Line* on 2025–03–12 [S1]. The method builds on Ref-NeRF but introduces architectural improvements to better encode view-dependent specular effects [S1].

## Sources
[S1] Specularity in NeRFs: A Comparative Study of Ref-NeRF and NRFF, https://doi.org/10.5201/ipol.2025.562, fetched 2026-10-26
