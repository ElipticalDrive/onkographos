# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter settings define the numerical configuration for the optimization process in 3D Gaussian Splatting variants, specifically governing the learning rates, optimizer stability, and regularization strengths. This method solves the problem of training instability and material-lighting ambiguity in inverse rendering and dynamic scenes by providing specific values for Adam optimizer parameters, per-parameter learning rates, and loss weights. It sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the continuous optimization of Gaussian attributes (position, scale, opacity, SH coefficients) and physical material properties rather than the discrete topology management of the Gaussian population.

## How it works
The method configures the gradient descent process used to update Gaussian primitives $\mathcal{G}$ and associated neural fields.

**Inputs and Outputs**
*   **Inputs:** Hyperparameter configuration file or constants; current Gaussian attributes (position $\mu$, scale $s$, rotation $q$, opacity $\alpha$, spherical harmonic coefficients $c$); material parameters (base color, metallic, roughness); illumination parameters (Direct/Indirect Light SH).
*   **Outputs:** Updated Gaussian attributes and scene representation after convergence.

**Data Structures**
The configuration is typically stored as a flat set of constants or a structured object containing:
1.  **Optimizer Config:** `beta1`, `beta2`, `epsilon`.
2.  **Learning Rates (LR):** A mapping from parameter type to scalar LR.
3.  **Regularization Weights:** Scalars for smoothness ($\lambda_{smooth}$) and other constraints ($\lambda_o, \lambda_{sc}, \lambda_{sr}$).
4.  **Loss Weights:** Scalars for color, depth, and physical consistency terms (e.g., $\alpha, \beta$).

**Numbered Steps**
1.  **Initialization:** Initialize Gaussian attributes. For dynamic or physical variants, initialize MLP weights from a normal distribution with $\sigma = 10^{-5}$ to ensure small initial deformations [S1]. Initialize reflectivity to 0.1 [S1].
2.  **Optimizer Setup:** Instantiate the Adam optimizer with the specified hyperparameters [S1][S2, unofficial].
3.  **Learning Rate Scheduling:** Apply per-parameter learning rates. For MLPs in dynamic variants, apply exponential decay from the initial rate to a target rate over a specified iteration count [S1].
4.  **Regularization Application:** During the loss computation, apply smoothness regularization to material properties (base color, metallic, roughness) using the specified $\lambda_{smooth}$ values [S2, unofficial]. Apply other regularizers with weight 1.0 to prevent abnormal Gaussian behavior [S2, unofficial].
5.  **Gradient Update:** Compute gradients via differentiable rasterization and update parameters using the Adam update rule [S1][S2, unofficial].

**Formulas**
The Adam optimizer update for a parameter $\theta_t$ at step $t$ is:
$$
m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t
$$
$$
v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2
$$
$$
\hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t}
$$
$$
\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \hat{m}_t
$$
Where:
*   $g_t$ is the gradient of the loss with respect to $\theta_t$.
*   $\eta$ is the learning rate specific to the parameter type (see Table 1).
*   $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-15}$ [S1][S2, unofficial].

**Parameters and Typical Values**
*   **Optimizer:** Adam [S1][S2, unofficial].
*   **Loss Weights (Dynamic/ToRF):** $\alpha = 5$ (or 1 for synthetic), $\beta = 0.0008$ [S1].
*   **Reflectivity LR:** $\gamma_r = 0.00016$ [S1].
*   **MLP LR:** Starts at $\gamma_{MLP} = 0.0008$, decays exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations [S1].
*   **Regularization Strengths (Phys3DGS):** $\lambda_{smooth}$ for base color = 0.06, metallic = 0.02, roughness = 0.02. Other regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr}$) = 1.0 [S2, unofficial].

**Table 1: Learning Rates per Parameter**
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

*Source: [S2, unofficial]*

**Edge Cases and Pitfalls**
*   **Abnormal Gaussian Behavior:** If a single Gaussian is driven to the opposite side of the surface, acquires an excessively large scale, or extreme opacity, it causes training instability [S2, unofficial].
*   **Visibility Interference:** Erroneous Gaussians occlude global illumination for nearby Gaussians [S2, unofficial].
*   **Gradient Overflow:** The optimizer attempts to compensate for occlusion by increasing local illumination intensity or decreasing opacity to extreme values, leading to gradient overflow in SH coefficients or opacity values [S2, unofficial].
*   **Mitigation:** High weights on regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr} = 1.0$) are used to prevent this instability [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_optimizer(gaussians, config):
    """
    Initialize Adam optimizer with specific learning rates for different Gaussian attributes.
    """
    optimizer = Adam()
    
    # Define parameter groups with specific learning rates
    param_groups = [
        {"params": gaussians.positions, "lr": 1.6e-4},
        {"params": gaussians.color_sh, "lr": 2.5e-3},
        {"params": gaussians.scales, "lr": 5e-3},
        {"params": gaussians.opacities, "lr": 5e-2},
        {"params": gaussians.base_color, "lr": 1e-2},
        {"params": gaussians.metallic, "lr": 1e-2},
        {"params": gaussians.roughness, "lr": 1e-2},
        {"params": gaussians.normal_rotation, "lr": 1e-3},
        {"params": gaussians.direct_light_sh_deg0, "lr": 2.5e-3},
        {"params": gaussians.direct_light_sh_deg_gt0, "lr": 1e-4},
        {"params": gaussians.indirect_light_sh_deg0, "lr": 2.5e-4},
        {"params": gaussians.indirect_light_sh_deg_gt0, "lr": 1e-3},
        {"params": gaussians.visibility_sh, "lr": 0.1}
    ]
    
    optimizer.add_param_groups(param_groups)
    optimizer.set_hyperparameters(beta1=0.9, beta2=0.999, eps=1e-15)
    return optimizer

def apply_regularization(loss, gaussians, config):
    """
    Apply smoothness regularization to material properties to prevent instability.
    """
    # Phys3DGS specific regularization strengths
    lambda_smooth_color = 0.06
    lambda_smooth_metallic = 0.02
    lambda_smooth_roughness = 0.02
    
    # Other regularizers (e.g., for preventing abnormal behavior)
    lambda_other = 1.0
    
    smoothness_loss = (
        lambda_smooth_color * compute_smoothness(gaussians.base_color) +
        lambda_smooth_metallic * compute_smoothness(gaussians.metallic) +
        lambda_smooth_roughness * compute_smoothness(gaussians.roughness)
    )
    
    other_regularization_loss = lambda_other * compute_other_constraints(gaussians)
    
    return loss + smoothness_loss + other_regularization_loss

def update_mlp_lr(iteration, initial_lr, target_lr, total_iterations):
    """
    Exponential decay for MLP learning rate in dynamic variants.
    """
    if iteration >= total_iterations:
        return target_lr
    # Exponential decay formula
    decay_factor = (target_lr / initial_lr) ** (1.0 / total_iterations)
    current_lr = initial_lr * (decay_factor ** iteration)
    return current_lr
```

## Subtopics

### [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md)
Adapts hyperparameters for 7D data (3D spatial, 1D temporal, 3D directional) by modifying pruning thresholds and leveraging temporal cues for adaptive density control.

## Operating characteristics
*   **Time Complexity:** The hyperparameter settings themselves have $O(1)$ cost. The optimization step complexity is dominated by the differentiable rasterization and gradient computation, which is $O(N \cdot M)$ where $N$ is the number of Gaussians and $M$ is the number of pixels [S1][S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing Gaussian attributes and optimizer states (momentum buffers) [S1][S2, unofficial].
*   **Strengths:** The specific learning rate hierarchy (e.g., high LR for opacity, low for position) stabilizes training. The high regularization weights prevent the "floater" and instability issues common in inverse rendering [S2, unofficial].
*   **Limitations:** Requires careful tuning of per-parameter learning rates. The exponential decay for MLPs requires careful scheduling to avoid premature convergence or instability [S1].
*   **When to use:** Use these settings when training Phys3DGS or ToRF variants where physical material properties or dynamic deformations are optimized. Do not use for standard static 3DGS without modification, as the physical regularizers may hinder standard photometric convergence.

## Background
The hyperparameter settings described here originate from two key papers: "Time of the Flight of the Gaussians" (ToRF) [S1] and "Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering" [S2, unofficial]. ToRF introduced specific settings for dynamic scenes involving time-of-flight data, including MLP learning rate decay and reflectivity initialization. Phys3DGS introduced settings for physically-based rendering, emphasizing regularization strengths to handle material-lighting ambiguity and prevent training instability caused by erroneous Gaussians. These settings are critical for the stability of inverse rendering pipelines using 3D Gaussian Splatting.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
