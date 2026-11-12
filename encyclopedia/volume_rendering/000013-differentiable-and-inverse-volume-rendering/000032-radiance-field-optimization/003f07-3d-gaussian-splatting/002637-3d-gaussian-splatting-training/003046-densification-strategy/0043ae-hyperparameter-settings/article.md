# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
This article defines the specific numerical values and configuration for the optimizer, learning rates, and regularization strengths used in physically-based 3D Gaussian Splatting (Phys3DGS) and dynamic radiance field optimizations. It addresses the problem of training instability and material-lighting ambiguity in inverse rendering by providing precise hyperparameters that prevent gradient overflow and ensure stable convergence of surface properties. It differs from general densification strategies by focusing on the optimization dynamics of the Gaussian attributes (position, scale, opacity, SH coefficients) and physical material parameters rather than primitive topology management.

## How it works
**Inputs:**
*   Model parameters: Gaussian positions $\mu$, scales $s$, rotations $q$, opacities $\alpha$, spherical harmonic coefficients $c_{SH}$, base color $C_{base}$, metallic $M$, roughness $R$, normal rotation $q_{norm}$, reflectivity $r$, and MLP weights $W_{MLP}$.
*   Ground truth images (RGB, Depth/ToF, or Relighting targets).

**Outputs:**
*   Updated model parameters after optimization steps.

**Data Structures:**
*   **Optimizer State:** Adam optimizer state buffers (first and second moment estimates) for each parameter group.
*   **Regularization Weights:** Scalar values $\lambda$ for smoothness and other constraints.

**Parameters and Typical Values:**
*   **Optimizer:** Adam [S1][S2, unofficial].
    *   $\beta_1 = 0.9$
    *   $\beta_2 = 0.999$
    *   $\epsilon = 10^{-15}$ [S1][S2, unofficial]
*   **Learning Rates (LR):** Defined per parameter group [S2, unofficial].
    *   Position: $1.6 \times 10^{-4}$
    *   Color SH: $2.5 \times 10^{-3}$
    *   Scale: $5 \times 10^{-3}$
    *   Opacity: $5 \times 10^{-2}$
    *   Base Color: $1 \times 10^{-2}$
    *   Metallic: $1 \times 10^{-2}$
    *   Roughness: $1 \times 10^{-2}$
    *   Normal Rotation: $1 \times 10^{-3}$
    *   Direct Light SH (deg=0): $2.5 \times 10^{-3}$
    *   Direct Light SH (deg>0): $1 \times 10^{-4}$
    *   Indirect Light SH (deg=0): $2.5 \times 10^{-4}$
    *   Indirect Light SH (deg>0): $1 \times 10^{-3}$
    *   Visibility SH: $0.1$
*   **Regularization Strengths:** [S2, unofficial]
    *   $\lambda_{smooth}$ (Base Color): $0.06$
    *   $\lambda_{smooth}$ (Metallic): $0.02$
    *   $\lambda_{smooth}$ (Roughness): $0.02$
    *   $\lambda_o$ (Opacity regularizer): $1.0$
    *   $\lambda_{sc}$ (Scale regularizer): $1.0$
    *   $\lambda_{sr}$ (Rotation regularizer): $1.0$
*   **Dynamic/MLP Specifics:** [S1]
    *   Reflectivity initialization: $0.1$
    *   Reflectivity LR ($\gamma_r$): $0.00016$
    *   MLP initial LR ($\gamma_{MLP}$): $0.0008$
    *   MLP LR decay: Exponential decay to $1.6 \times 10^{-6}$ over 60,000 iterations.
    *   MLP weight initialization: Normal distribution with $\sigma = 10^{-5}$.
    *   Loss weights: $\alpha = 5$ (or $1$ for synthetic scenes), $\beta = 0.0008$.

**Algorithm Steps:**
1.  **Initialization:** Initialize Gaussian attributes. Initialize MLP weights from $\mathcal{N}(0, 10^{-5})$. Initialize reflectivity to $0.1$. Set Adam optimizer with $\beta=(0.9, 0.999), \epsilon=10^{-15}$.
2.  **Forward Pass:** Render the scene using differentiable rasterization. For dynamic scenes, render twice: once for C-ToF view (quad loss) and once for RGB view (color loss) [S1]. For inverse rendering, compute PBR rendering in linear HDR space [S2, unofficial].
3.  **Loss Computation:** Compute reconstruction loss and apply regularization terms.
    *   Apply smoothness regularizers to Base Color, Metallic, and Roughness with weights $0.06, 0.02, 0.02$ respectively [S2, unofficial].
    *   Apply other regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr}$) with weight $1.0$ [S2, unofficial].
4.  **Backward Pass:** Compute gradients $\frac{\partial L}{\partial \theta}$ for all parameters $\theta$.
5.  **Update:** Update parameters using Adam optimizer with the specific learning rates listed above [S1][S2, unofficial].
    *   For MLP parameters, apply exponential LR decay schedule [S1].
    *   For reflectivity, apply specific LR $\gamma_r = 0.00016$ [S1].

**Pitfalls and Edge Cases:**
*   **Gradient Overflow:** If a single Gaussian behaves abnormally (e.g., driven to the opposite side of the surface, large scale, or extreme opacity), it causes instability [S2, unofficial].
    *   Erroneous Gaussians interfere with visibility, occluding global illumination [S2, unofficial].
    *   The optimizer compensates by increasing local illumination intensity or decreasing opacity to extreme values [S2, unofficial].
    *   This leads to large gradients in SH coefficients or opacity, causing overflow [S2, unofficial].
    *   Mitigation: The high weights ($1.0$) on regularizers $\lambda_o, \lambda_{sc}, \lambda_{sr}$ are critical to prevent this [S2, unofficial].
*   **Material-Lighting Ambiguity:** Inverse rendering suffers from ambiguity between base color and lighting. A rescale factor (median of pixel-wise ratios) is applied to base color for synthetic scenes to resolve this [S2, unofficial]. For real captures, this is omitted [S2, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Determined by the rasterization and backward pass of the Gaussian splatting pipeline. The use of Adam optimizer adds constant memory overhead per parameter for moment estimates.
*   **Memory Complexity:** Standard 3DGS memory footprint plus buffers for MLP weights (if used) and additional regularization state.
*   **Strengths:** The specific learning rate hierarchy (e.g., high LR for Opacity, low for Position) and strong regularization weights stabilize the optimization of physically-based attributes (metallic, roughness) which are prone to instability.
*   **Limitations:** The hyperparameters are tuned for specific setups (e.g., $\alpha=5$ for real-world, $\alpha=1$ for synthetic) [S1]. Deviating from these may lead to the instability described in pitfalls.
*   **When to use:** When training Phys3DGS or dynamic 3DGS variants that require stable convergence of material properties and time-dependent deformations.

## Background
The hyperparameter settings originate from Phys3DGS [S2, unofficial] and Time of the Flight of the Gaussians (ToRF) [S1]. Phys3DGS introduces physically-based attributes (metallic, roughness) and inverse rendering capabilities, requiring careful tuning of regularizers to prevent the "erroneous Gaussian" instability. ToRF adapts 3DGS for dynamic scenes using C-ToF data, introducing MLP-based deformations with specific decay schedules. These settings are critical for the success of inverse rendering and dynamic radiance field optimization within the 3D Gaussian Splatting framework.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
