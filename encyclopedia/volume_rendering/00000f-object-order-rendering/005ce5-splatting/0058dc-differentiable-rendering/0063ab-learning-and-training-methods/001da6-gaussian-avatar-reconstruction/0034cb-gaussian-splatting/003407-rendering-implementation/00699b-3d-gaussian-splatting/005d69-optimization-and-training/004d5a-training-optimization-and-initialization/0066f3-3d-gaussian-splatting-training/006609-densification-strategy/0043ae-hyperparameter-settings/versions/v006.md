# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter settings define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting variants. This article focuses on the specific values required to stabilize optimization, particularly in physically-based inverse rendering contexts where material-lighting ambiguities can cause instability. It sets apart from general training guides by providing exact learning rates for PBR parameters (base color, metallic, roughness) and regularization strengths that prevent gradient overflow caused by erroneous Gaussians [S2, unofficial].

## How it works
The training process relies on the Adam optimizer and specific regularization weights to manage the Gaussian population $\mathcal{G}$. The settings are applied to the parameters defined in the parent [Densification Strategy](6234-densification-strategy.md) and the specific radiance field representation.

### Inputs and Outputs
*   **Inputs:** Gaussian parameters (position, scale, rotation, opacity, spherical harmonics coefficients), scene data, and hyperparameter configuration.
*   **Outputs:** Updated Gaussian parameters after gradient descent steps.

### Data Structures
The hyperparameters are stored as scalar values or tuples associated with specific parameter groups.

**Optimizer Configuration:**
*   `optimizer_type`: String, "Adam".
*   `beta`: Tuple of floats $(\beta_1, \beta_2)$.
*   `epsilon`: Float, numerical stability constant.

**Learning Rate Map:**
A dictionary mapping parameter names to their initial learning rates ($\eta$).

**Regularization Weights:**
*   $\lambda_{smooth}$: Dictionary mapping material properties to smoothness weights.
*   $\lambda_{other}$: Scalar or dictionary for other regularizers (opacity, scale, rotation).

### Parameters and Values
The following values are derived from Phys3DGS [S2, unofficial] and ToRF [S1].

**Optimizer:**
*   $\beta_1 = 0.9$
*   $\beta_2 = 0.999$
*   $\epsilon = 10^{-15}$ [S1][S2, unofficial]

**Learning Rates ($\eta$):**
| Parameter | Learning Rate |
| :--- | :--- |
| Position | $1.6 \times 10^{-4}$ |
| Color SH | $2.5 \times 10^{-3}$ |
| Scale | $5 \times 10^{-3}$ |
| Opacity | $5 \times 10^{-2}$ |
| Base Color | $1 \times 10^{-2}$ |
| Metallic | $1 \times 10^{-2}$ |
| Roughness | $1 \times 10^{-2}$ |
| Normal Rotation | $1 \times 10^{-3}$ |
| Direct Light SH (deg=0) | $2.5 \times 10^{-3}$ |
| Direct Light SH (deg>0) | $1 \times 10^{-4}$ |
| Indirect Light SH (deg=0) | $2.5 \times 10^{-4}$ |
| Indirect Light SH (deg>0) | $1 \times 10^{-3}$ |
| Visibility SH | $0.1$ |

*Note: ToRF uses a specific MLP learning rate decay: $\gamma_{MLP}$ starts at $0.0008$ and decays exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations [S1].*

**Regularization Strengths:**
*   $\lambda_{smooth}$ (Base Color) = $0.06$
*   $\lambda_{smooth}$ (Metallic) = $0.02$
*   $\lambda_{smooth}$ (Roughness) = $0.02$
*   $\lambda_o$ (Opacity) = $1.0$
*   $\lambda_{sc}$ (Scale) = $1.0$
*   $\lambda_{sr}$ (Rotation) = $1.0$ [S2, unofficial]

**Other Initialization:**
*   Reflectivity initialization: $0.1$ [S1]
*   Reflectivity learning rate: $\gamma_r = 0.00016$ [S1]
*   Layer weights (e.g., for deformations): Normal distribution with $\sigma = 10^{-5}$ [S1]

### Numbered Steps
1.  **Initialize Optimizer:** Create an Adam optimizer instance with $\beta=(0.9, 0.999)$ and $\epsilon=10^{-15}$ [S1][S2, unofficial].
2.  **Assign Learning Rates:** Map the learning rates from the table above to the corresponding parameter tensors in the Gaussian representation.
3.  **Configure Regularization:** Set the smoothness weights for PBR parameters ($\lambda_{smooth}$) and the general regularizer weights ($\lambda_o, \lambda_{sc}, \lambda_{sr}$) to 1.0 [S2, unofficial].
4.  **Compute Loss:** Calculate the reconstruction loss and apply regularization terms weighted by the configured $\lambda$ values.
5.  **Backpropagate and Step:** Compute gradients and perform the optimizer step.

### Formulas
The loss function $L$ includes regularization terms $R$:
$$ L = L_{recon} + \lambda_{smooth}^{color} R_{smooth}^{color} + \lambda_{smooth}^{metal} R_{smooth}^{metal} + \lambda_{smooth}^{rough} R_{smooth}^{rough} + \lambda_o R_o + \lambda_{sc} R_{sc} + \lambda_{sr} R_{sr} $$
where $R_{smooth}$ typically penalizes differences in SH coefficients between neighboring Gaussians or views [S2, unofficial].

### Edge Cases and Pitfalls
*   **Gradient Overflow:** If a single Gaussian behaves abnormally (e.g., driven to the opposite side of the surface, extremely large scale, or extreme opacity), it interferes with visibility and global illumination [S2, unofficial].
*   **Instability Mechanism:** Erroneous Gaussians occlude global illumination. The optimizer compensates by either increasing local illumination intensity or decreasing opacity to extreme values. This leads to large gradients in SH coefficients or opacity, causing overflow [S2, unofficial].
*   **Mitigation:** The high regularization weights ($\lambda_o, \lambda_{sc}, \lambda_{sr} = 1.0$) are explicitly set to prevent this instability [S2, unofficial].

## Subtopics

## Operating characteristics
*   **Stability:** The specific regularization weights ($\lambda=1.0$ for non-smooth terms) are critical for stability in inverse rendering. Without them, training becomes unstable due to the material-lighting ambiguity [S2, unofficial].
*   **Convergence:** The learning rates are tuned to balance the convergence of geometric parameters (position, scale) with appearance parameters (SH, PBR colors). Position has a lower learning rate ($1.6e-4$) compared to opacity ($5e-2$) to ensure stable geometry updates [S2, unofficial].
*   **Hardware:** Standard GPU acceleration is assumed for the Adam optimizer and rasterization.

## Background
These settings originate from Phys3DGS [S2, unofficial] and ToRF [S1], which extend standard 3D Gaussian Splatting to physically-based inverse rendering and dynamic scenes, respectively. The settings address the specific numerical challenges of optimizing PBR parameters and dynamic deformations.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
