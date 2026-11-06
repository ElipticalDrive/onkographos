# Opacity-Thin-Side (OTS)
## Overview
Opacity-Thin-Side (OTS) is a parameterization scheme for EWA-based splatting that ensures robust gradient-descent optimization by dynamically scaling the learned weight $\theta$ such that the projected amplitude $a'$ equals $\theta$ when viewing the Gaussian along its thinnest side (shortest axis) [S1]. It differs from standard EWA splatting, which preserves the integral of extinction (mass), and 3D Gaussian Splatting (3DGS), which preserves the 2D projected amplitude regardless of view, by preserving the 3D unbounded integral (view-independent mass) while ensuring the 2D projection behaves as an ordinary integral relative to the thinnest view [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of Gaussian primitives, each defined by a 3D covariance matrix $\mathbf{\Sigma}$, a position, and a color [S1]. The method introduces a learned scalar weight $\theta$ per Gaussian [S1].
*   **Output:** The 3D unnormalized amplitude $a$ and the 2D projected amplitude $a'$ used in the splatting accumulation equation [S1].

**Data Structures:**
*   **Gaussian Primitive:**
    *   $\mathbf{\Sigma}$: $3 \times 3$ symmetric positive-definite covariance matrix [S1].
    *   $\theta$: Scalar learned weight [S1].
    *   $\mathbf{W}$: Transformation matrix to camera space [S1].
    *   $\boldsymbol{\mu}$: Position in camera space [S1].
    *   $c_i$: Color/opacity attribute [S1].

**Steps:**
1.  **Learn Weight:** Optimize a scalar weight $\theta$ per Gaussian during training [S1].
2.  **Compute 3D Normalization:** Calculate the 3D normalization factor $\mathcal{I}_3(\mathbf{\Sigma})$ for the unnormalized Gaussian [S1].
3.  **Compute Projected 2D Normalization:** Calculate the 2D normalization factor $\mathcal{I}_2(\mathbf{\Sigma}')$ of the projected covariance $\mathbf{\Sigma}'$ [S1].
4.  **Compute Maximum 2D Normalization:** Calculate $\mathcal{I}_2^*(\mathbf{\Sigma})$ using the largest and second-largest eigenvalues of the 3D covariance $\mathbf{\Sigma}$ [S1].
5.  **Calculate 3D Amplitude:** Compute the 3D amplitude $a$ using the formula:
    $$a = \theta \frac{\mathcal{I}_2^*(\mathbf{\Sigma})}{\mathcal{I}_3(\mathbf{\Sigma})}$$
    [S1]
6.  **Calculate 2D Amplitude:** Compute the 2D amplitude $a'$ using the formula:
    $$a' = \theta \frac{\mathcal{I}_2^*(\mathbf{\Sigma})}{\mathcal{I}_2(\mathbf{\Sigma}')}$$
    [S1]

**Formulas and Definitions:**
*   **Projected Covariance:** The 2D covariance $\mathbf{\Sigma}'$ is computed via the Jacobian $\mathbf{J}$ of the perspective projection approximation:
    $$\mathbf{\Sigma}' = \mathbf{J} \mathbf{W} \mathbf{\Sigma} \mathbf{W}^T \mathbf{J}^T$$
    where $\mathbf{J}$ is the first-order Taylor expansion of the perspective transform:
    $$\mathbf{J} = \begin{pmatrix} 1/\mu_2 & 0 & -\mu_0/\mu_2^2 \\ 0 & 1/\mu_2 & -\mu_1/\mu_2^2 \\ \mu_0/l & \mu_1/l & \mu_2/l \end{pmatrix}$$
    Here, $\boldsymbol{\mu} = (\mu_0, \mu_1, \mu_2)^T$ is the Gaussian position in camera space, $l = \|\boldsymbol{\mu}\|_2$, and $\mathbf{W}$ transforms to camera space [S1].
*   **Maximum 2D Normalization Factor:**
    $$\mathcal{I}_2^*(\mathbf{\Sigma}) = 2\pi\sqrt{\lambda_1\lambda_2}$$
    where $\lambda_1$ is the largest eigenvalue and $\lambda_2$ is the second-largest eigenvalue of $\mathbf{\Sigma}$ [S1].
*   **Normalization Factors:** $\mathcal{I}_3(\mathbf{\Sigma})$ is the 3D normalization factor and $\mathcal{I}_2(\mathbf{\Sigma}')$ is the 2D normalization factor of the projected covariance [S1]. The exact definitions of $\mathcal{I}_3$ and $\mathcal{I}_2$ are not explicitly provided in the source text beyond their usage in the ratio [S1]. [G: In standard Gaussian theory, $\mathcal{I}_3(\mathbf{\Sigma}) = (2\pi)^{3/2}\sqrt{\det(\mathbf{\Sigma})}$ and $\mathcal{I}_2(\mathbf{\Sigma}') = 2\pi\sqrt{\det(\mathbf{\Sigma}')}$.]

**Pitfalls:**
*   Directly learning the weight $w$ (total integral of normalized Gaussians) impedes training speed and robustness [S1]. Changes in Gaussian size (covariance) inversely affect the projected amplitude $a'$ without affecting $w$, causing implicit transparency changes during optimization [S1].
*   For thin, solid objects, an almost-flat Gaussian requires a much higher extinction than a thicker Gaussian to appear opaque, which can lead to instability if not handled by the OTS scaling [S1].

## Operating characteristics
Not given in the sources.

## Background
OTS was introduced in the paper "Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?" to address the deviation of 3DGS from volume rendering theory and to adapt EWA-based splatting for gradient-descent-based optimization [S1]. It aims to provide a principled alternative that maintains view-independent mass while ensuring robust behavior for thin objects [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
