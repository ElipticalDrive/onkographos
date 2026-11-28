# N-Dimensional Gaussian (6DGS/7DGS)
Also called: 6DGS, 7DGS, Gao et al, 2025a, 2025b

## Overview
N-Dimensional Gaussian extends 3D Gaussian Splatting by modeling a joint probability distribution over spatial coordinates and non-spatial dimensions (such as viewing direction or time), then conditioning on the query to derive view-specific or time-specific 3D Gaussians for rendering [S1, unofficial]. Unlike standard 3DGS which treats primitives as static spatial entities, this method explicitly encodes correlations between geometry and appearance/dynamics via cross-covariance terms, allowing for continuous view-dependent effects and temporal coherence without discrete keyframes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of $N$ N-dimensional Gaussian primitives. Each primitive $i$ is defined by parameters $\{\boldsymbol{\mu}_p, \boldsymbol{\mu}_d, \boldsymbol{\Sigma}_p, \boldsymbol{\Sigma}_d, \boldsymbol{\Sigma}_{pd}, \alpha, \lambda\}$.
- **Query:** A specific non-spatial vector $\mathbf{q}$ (e.g., viewing direction $\mathbf{d}$ or time $t$).
- **Output:** A set of conditional 3D Gaussian primitives $\{\boldsymbol{\mu}_{cond}, \boldsymbol{\Sigma}_{cond}, \alpha_{cond}\}$ ready for standard 3DGS splatting.

**Data Structures:**
Each primitive stores the following fields:
- $\boldsymbol{\mu}_p \in \mathbb{R}^3$: Spatial mean (position).
- $\boldsymbol{\mu}_d \in \mathbb{R}^k$: Non-spatial mean (e.g., $\mathbb{R}^3$ for direction, $\mathbb{R}^1$ for time).
- $\boldsymbol{\Sigma}_p \in \mathbb{R}^{3 \times 3}$: Spatial covariance block.
- $\boldsymbol{\Sigma}_d \in \mathbb{R}^{k \times k}$: Non-spatial covariance block.
- $\boldsymbol{\Sigma}_{pd} \in \mathbb{R}^{3 \times k}$: Cross-covariance block between spatial and non-spatial dimensions.
- $\alpha \in \mathbb{R}$: Base opacity.
- $\lambda \in \mathbb{R}$: Opacity modulation scaling factor.

**Numbered Steps:**
1. **Construct Joint Distribution:** Define the joint vector $X = \begin{pmatrix} X_p \\ X_d \end{pmatrix}$ where $X_p$ represents spatial coordinates and $X_d$ represents non-spatial coordinates. The joint distribution is:
   $$
   X \sim \mathcal{N}\left(\begin{pmatrix} \boldsymbol{\mu}_p \\ \boldsymbol{\mu}_d \end{pmatrix}, \begin{pmatrix} \boldsymbol{\Sigma}_p & \boldsymbol{\Sigma}_{pd} \\ \boldsymbol{\Sigma}_{pd}^\top & \boldsymbol{\Sigma}_d \end{pmatrix}\right) \quad (5)
   $$
2. **Condition on Query:** Given a query value $\mathbf{q}$ for the non-spatial dimension $X_d$, compute the conditional mean and covariance for the spatial dimension $X_p$ using standard Gaussian conditioning formulas:
   $$
   \boldsymbol{\mu}_{cond} = \boldsymbol{\mu}_p + \boldsymbol{\Sigma}_{pd} \boldsymbol{\Sigma}_d^{-1} (\mathbf{q} - \boldsymbol{\mu}_d)
   $$
   $$
   \boldsymbol{\Sigma}_{cond} = \boldsymbol{\Sigma}_p - \boldsymbol{\Sigma}_{pd} \boldsymbol{\Sigma}_d^{-1} \boldsymbol{\Sigma}_{pd}^\top
   $$
3. **Modulate Opacity:** Calculate the conditional opacity $\alpha_{cond}$ based on the likelihood of the query $\mathbf{q}$ under the non-spatial distribution:
   $$
   \alpha_{cond} = \alpha \cdot \exp\left(-\lambda (\mathbf{q} - \boldsymbol{\mu}_d)^\top \boldsymbol{\Sigma}_d^{-1} (\mathbf{q} - \boldsymbol{\mu}_d)\right)
   $$
4. **Render:** Use $\boldsymbol{\mu}_{cond}$, $\boldsymbol{\Sigma}_{cond}$, and $\alpha_{cond}$ as the parameters for the primitive during the standard 3DGS splatting pass.

**Parameters:**
- $\lambda$: Controls the sharpness of opacity modulation. Positive values yield sharper peaks, better capturing high-frequency details [S1, unofficial].
- $\boldsymbol{\Sigma}_d^{-1}$: Requires the non-spatial covariance to be invertible. If singular, pseudo-inverse or regularization is needed [G].

**Edge Cases and Pitfalls:**
- **Singularity:** If $\boldsymbol{\Sigma}_d$ is not positive definite or invertible, the conditioning step fails. Regularization (e.g., adding $\epsilon I$) is required [G].
- **Degenerate Input:** If the query $\mathbf{q}$ is far from $\boldsymbol{\mu}_d$, the exponential term in opacity may underflow to zero, effectively culling the primitive.
- **Dimensionality:** For 6DGS, $k=3$ (view direction). For 7DGS, $k=4$ (view direction + time) [S1, unofficial].

## Subtopics

### [3D/4D Gaussian Splatting Mixture Model](17194-3d-4d-gaussian-splatting-mixture-model.md)
Treats the radiance field as a probabilistic mixture of 3D Gaussians, modeling uncertainty and distributional parameters rather than single-point estimates. Choose this when modeling data uncertainty is critical; it differs from N-Dimensional Gaussian by focusing on mixture components rather than conditional distributions over explicit non-spatial dimensions.

## Operating characteristics
- **Time Complexity:** Rendering time increases slightly due to the per-primitive conditioning calculation ($O(k^3)$ for matrix inversion if done naively, but $k$ is small: 3 or 4). The splatting pass itself remains $O(N)$ per pixel.
- **Memory Complexity:** Each primitive stores additional parameters for the non-spatial mean, covariance, and cross-covariance. For 6DGS, this adds $\approx 3 + 6 + 9 = 18$ floats per primitive compared to standard 3DGS. For 7DGS, it adds more for the temporal dimension.
- **Strengths:** Enables continuous view-dependent appearance and smooth temporal transitions without discrete keyframes or explicit deformation fields.
- **Limitations:** Increased memory footprint per primitive. Training requires optimizing cross-covariance terms, which may be more complex than standard 3DGS.
- **When to use:** Scenes with strong view-dependent effects (specularities, iridescence) or smooth temporal dynamics where discrete keyframing is insufficient.

## Background
N-Dimensional Gaussian (6DGS/7DGS) was introduced by Gao et al. in 2025 [S1, unofficial]. It extends the 3D Gaussian Splatting framework by incorporating conditional probability distributions to handle non-spatial dimensions like viewing direction and time. This approach allows for a more compact and continuous representation of dynamic and view-dependent scenes.

## Sources
[S1] Universal Beta Splatting, https://doi.org/10.48550/arXiv.2510.03312, fetched 2026-10-30 (unofficial)
