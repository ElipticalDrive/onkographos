# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter Settings define the numerical configuration for the optimization process in 3D Gaussian Splatting (3DGS) variants, specifically governing learning rates, optimizer stability, and regularization strengths. This method addresses the instability inherent in inverse rendering and differentiable rasterization, where erroneous Gaussians can cause gradient overflow or occlusion errors [S2, unofficial]. It sets apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the *numerical* stability of the optimizer rather than the *topological* management of the Gaussian population, although it operates in conjunction with the [Densification Strategy](6234-densification-strategy.md).

## How it works
The method configures the Adam optimizer and loss function weights to ensure stable convergence of Gaussian parameters (position, scale, opacity, SH coefficients) and material properties (in physically-based variants).

**Inputs:**
- Target parameters: Position $\mu$, Scale $s$, Opacity $\alpha$, Spherical Harmonics coefficients $c_{SH}$, Material properties (Base Color, Metallic, Roughness, Normal Rotation).
- Optimization hyperparameters: Learning rates $\eta$, optimizer moments $\beta_1, \beta_2, \epsilon$, regularization weights $\lambda$.

**Data Structures:**
- **Optimizer State:** Per-parameter Adam states (first moment $m$, second moment $v$).
- **Regularization Weights:** Scalar values $\lambda_{smooth}$ for material smoothness and $\lambda_o, \lambda_{sc}, \lambda_{sr}$ for other constraints.

**Parameters and Typical Values:**
1.  **Optimizer:** Adam with $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-15}$ [S1][S2, unofficial].
2.  **Learning Rates ($\eta$):**
    - Position: $1.6 \times 10^{-4}$ [S2, unofficial].
    - Color SH: $2.5 \times 10^{-3}$ [S2, unofficial].
    - Scale: $5 \times 10^{-3}$ [S2, unofficial].
    - Opacity: $5 \times 10^{-2}$ [S2, unofficial].
    - Base Color: $1 \times 10^{-2}$ [S2, unofficial].
    - Metallic: $1 \times 10^{-2}$ [S2, unofficial].
    - Roughness: $1 \times 10^{-2}$ [S2, unofficial].
    - Normal Rotation: $1 \times 10^{-3}$ [S2, unofficial].
    - Direct Light SH (deg=0): $2.5 \times 10^{-3}$ [S2, unofficial].
    - Direct Light SH (deg>0): $1 \times 10^{-4}$ [S2, unofficial].
    - Indirect Light SH (deg=0): $2.5 \times 10^{-4}$ [S2, unofficial].
    - Indirect Light SH (deg>0): $1 \times 10^{-3}$ [S2, unofficial].
    - Visibility SH: $0.1$ [S2, unofficial].
    - Reflectivity: $\gamma_r = 0.00016$ [S1].
    - MLP (deformation): Starts at $\gamma_{MLP} = 0.0008$, decays exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations [S1].
3.  **Regularization Weights:**
    - Smoothness ($\lambda_{smooth}$): Base Color $0.06$, Metallic $0.02$, Roughness $0.02$ [S2, unofficial].
    - Other Regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr}$): $1.0$ [S2, unofficial].
    - Loss weights ($\alpha, \beta$): $\alpha = 5$ (or $1$ for synthetic), $\beta = 0.0008$ [S1].

**Steps:**
1.  **Initialization:** Initialize Gaussian parameters. For deformation MLPs, initialize weights from a normal distribution with $\sigma = 10^{-5}$ to ensure small initial deformations $\delta x_k$ [S1]. Initialize reflectivity to $0.1$ [S1].
2.  **Gradient Computation:** Compute gradients of the loss function $L$ with respect to all learnable parameters using differentiable rasterization [S1].
3.  **Update:** Apply Adam update rule:
    $$ \hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} $$
    $$ \theta_{t+1} = \theta_t - \eta \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} $$
    where $\theta$ represents the specific parameter group (e.g., Position, SH) with its specific $\eta$ [S2, unofficial].
4.  **Regularization Application:** Add smoothness terms to the loss for material properties to prevent high-frequency noise in BRDF parameters [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Erroneous Gaussians:** If a single Gaussian behaves abnormally (e.g., driven to the opposite side of the surface, large scale, or extreme opacity), it causes instability [S2, unofficial].
    - *Mechanism:* Erroneous Gaussians interfere with visibility, occluding global illumination. The optimizer compensates by increasing local illumination intensity or decreasing opacity to extreme values.
    - *Result:* Global illumination SH coefficients or opacity values receive large gradients, leading to overflow [S2, unofficial].
    - *Mitigation:* High weights on other regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr} = 1.0$) are used to suppress this behavior [S2, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Determined by the number of Gaussians and the depth of the SH expansion. The exponential decay of the MLP learning rate adds negligible overhead.
- **Memory Complexity:** Standard 3DGS memory footprint plus storage for material properties (Base Color, Metallic, Roughness) and additional SH coefficients for lighting if used [S2, unofficial].
- **Strengths:** High stability for inverse rendering problems; prevents gradient explosion in physically-based variants.
- **Limitations:** Requires careful tuning of learning rates for different parameter groups; sensitive to initial Gaussian placement.
- **When to use:** When training physically-based 3DGS or dynamic scenes requiring deformation MLPs [S1][S2, unofficial].

## Background
Originated from the need to stabilize 3D Gaussian Splatting for inverse rendering and dynamic scenes. [S1] introduced specific settings for Time-of-Flight (ToF) data and dynamic deformation MLPs. [S2, unofficial] (Phys3DGS) formalized the learning rate table and regularization strengths to handle material-lighting ambiguity and prevent instability from erroneous Gaussians in physically-based rendering.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
