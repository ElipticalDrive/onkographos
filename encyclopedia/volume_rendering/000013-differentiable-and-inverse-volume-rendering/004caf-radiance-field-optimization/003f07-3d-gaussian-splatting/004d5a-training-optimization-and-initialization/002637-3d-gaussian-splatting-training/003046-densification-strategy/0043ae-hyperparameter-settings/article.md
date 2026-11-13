# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter settings define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting (3DGS) and its variants. These settings determine the convergence speed, stability, and final quality of the radiance field optimization. Unlike the densification strategy which manages the topology of the Gaussian set, hyperparameter settings govern the magnitude and direction of parameter updates [S1][S2, unofficial]. The specific values vary significantly between standard photometric reconstruction, physically-based inverse rendering, and neural-textured surfel methods [S1][S2, unofficial][S3].

## How it works
The training process involves minimizing a loss function $\mathcal{L}$ with respect to Gaussian parameters $\theta = \{\mu, \Sigma, \alpha, \mathbf{c}\}$ using an iterative optimizer. The configuration consists of optimizer hyperparameters, per-parameter learning rates, and regularization weights.

### Inputs and Outputs
*   **Inputs:** Loss gradients $\frac{\partial \mathcal{L}}{\partial \theta}$, current parameter values $\theta_t$, iteration count $t$.
*   **Outputs:** Updated parameters $\theta_{t+1}$.

### Data Structures
The hyperparameters are stored as scalar constants or vectors indexed by parameter type.
*   **Optimizer State:** First moment $m_t$ and second moment $v_t$ estimates for each parameter group.
*   **Learning Rate Schedule:** A mapping from parameter type to initial LR $\gamma_0$ and decay rules.
*   **Regularization Weights:** Scalars $\lambda_{smooth}, \lambda_o, \lambda_{sc}, \lambda_{sr}$ applied to specific loss terms.

### Steps
1.  **Initialization:** Initialize Gaussian parameters (position, scale, rotation, opacity, SH coefficients). Initialize optimizer moments to zero. Set reflectivity to 0.1 if applicable [S1]. Initialize MLP weights from $\mathcal{N}(0, 10^{-5})$ if using deformation networks [S1].
2.  **Gradient Computation:** Compute gradients of the total loss $\mathcal{L}_{total} = \mathcal{L}_{photometric} + \mathcal{L}_{regularization}$ with respect to all parameters [S1][S2, unofficial].
3.  **Optimizer Update:** Apply the Adam optimizer to update parameters.
    $$ \theta_{t+1} = \theta_t - \frac{\gamma_t}{\sqrt{\hat{v}_t} + \epsilon} \hat{m}_t $$
    where $\hat{m}_t$ and $\hat{v}_t$ are bias-corrected first and second moment estimates [S1][S2, unofficial].
4.  **Regularization Application:** Add smoothness penalties to the loss before gradient computation. For physically-based rendering, apply smoothness regularizers to base color, metallic, and roughness [S2, unofficial].
5.  **Learning Rate Decay:** Update learning rates according to schedule (e.g., exponential decay for MLPs) [S1].

### Formulas and Parameters

#### Optimizer Configuration
*   **Optimizer:** Adam [S1][S2, unofficial].
*   **Beta Coefficients:** $\beta_1 = 0.9, \beta_2 = 0.999$ [S1][S2, unofficial].
*   **Epsilon:** $\epsilon = 10^{-15}$ [S1][S2, unofficial].

#### Learning Rates (Phys3DGS)
The following table lists the learning rates for Phys3DGS [S2, unofficial].

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

#### Regularization Weights (Phys3DGS)
*   **Smoothness Regularizers ($\lambda_{smooth}$):**
    *   Base Color: $0.06$ [S2, unofficial]
    *   Roughness: $0.02$ [S2, unofficial]
    *   Metallic: $0.02$ [S2, unofficial]
*   **Other Regularizers:**
    *   $\lambda_o, \lambda_{sc}, \lambda_{sr}$ are all set to $1.0$ [S2, unofficial].

#### Time-of-Flight (ToRF) Specific Settings
*   **Loss Weights:** $\alpha = 5$ (for real-world scenes), $\beta = 0.0008$ [S1].
*   **Reflectivity Learning Rate:** $\gamma_r = 0.00016$ [S1].
*   **MLP Learning Rate:** Starts at $\gamma_{MLP} = 0.0008$, decays exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_parameters(params, gradients, optimizer_state, t, config):
    """
    Update 3D Gaussian parameters using Adam optimizer with specific hyperparameters.
    """
    # Extract config values
    beta1 = config.beta1  # 0.9
    beta2 = config.beta2  # 0.999
    eps = config.epsilon  # 1e-15
    
    # Initialize optimizer state if first step
    if t == 0:
        optimizer_state = {p: {'m': 0, 'v': 0} for p in params}

    for param in params:
        grad = gradients[param]
        state = optimizer_state[param]
        
        # Update biased first moment estimate
        state['m'] = beta1 * state['m'] + (1 - beta1) * grad
        # Update biased second raw moment estimate
        state['v'] = beta2 * state['v'] + (1 - beta2) * grad ** 2
        
        # Compute bias-corrected estimates
        m_hat = state['m'] / (1 - beta1 ** t)
        v_hat = state['v'] / (1 - beta2 ** t)
        
        # Get learning rate for this parameter type
        lr = get_learning_rate(param.type, t, config.lr_schedule)
        
        # Update parameter
        param.value -= lr * m_hat / (sqrt(v_hat) + eps)
        
    return optimizer_state

def compute_loss_with_regularization(rendered_image, gt_image, gaussians, config):
    """
    Compute total loss including photometric and regularization terms.
    """
    # Photometric loss (e.g., L1 or LPIPS)
    photometric_loss = compute_photometric_loss(rendered_image, gt_image)
    
    total_loss = photometric_loss
    
    # Add Phys3DGS specific regularizers
    if config.use_phys3dgs_regularization:
        # Smoothness for base color
        total_loss += config.lambda_smooth_color * compute_smoothness(gaussians.base_color)
        # Smoothness for roughness
        total_loss += config.lambda_smooth_rough * compute_smoothness(gaussians.roughness)
        # Smoothness for metallic
        total_loss += config.lambda_smooth_metal * compute_smoothness(gaussians.metallic)
        
        # Other regularizers (opacity, scale, etc.)
        total_loss += config.lambda_o * compute_opacity_regularization(gaussians.opacity)
        total_loss += config.lambda_sc * compute_scale_regularization(gaussians.scale)
        total_loss += config.lambda_sr * compute_rotation_regularization(gaussians.rotation)
        
    return total_loss
```

### Edge Cases and Pitfalls
*   **Instability from Erroneous Gaussians:** If a single Gaussian behaves abnormally (e.g., driven to the opposite side of the surface, extremely large scale, or extreme opacity), it causes training instability [S2, unofficial].
    1.  The erroneous Gaussian may intersect with other surface parts or render outside the object region [S2, unofficial].
    2.  It interferes with visibility, mistakenly occluding global illumination for nearby Gaussians [S2, unofficial].
    3.  The optimizer compensates by either increasing local illumination intensity (dimming global) or decreasing opacity to extreme values [S2, unofficial].
    4.  This leads to large gradients in SH coefficients or opacity, potentially causing overflow [S2, unofficial].
    *   **Mitigation:** High weights for regularizers $\lambda_o, \lambda_{sc}, \lambda_{sr}$ (set to 1.0) are used to prevent this behavior [S2, unofficial].
*   **Gradient Overflow:** Extreme opacity or intensity changes can cause numerical overflow in spherical harmonic coefficients [S2, unofficial].
*   **MLP Initialization:** MLP weights for deformation networks should be initialized with small standard deviation ($10^{-5}$) to ensure stable optimization by starting deformations from small values [S1].

## Subtopics

### Blended Error Densification and Splitting
This variant, used in Nexels, replaces the standard gradient-based densification with a blended error heuristic [S3].
*   **Definition:** Blended L1 errors are accumulated per primitive across iterations by addition (not maximum) [S3].
*   **Splitting Mechanism:** Primitives are sampled for splitting with probability proportional to their normalized blended error (e.g., top 5%) [S3].
*   **Split Formula:** For a surfel $i$ with scales $\sigma_{i,1} \ge \sigma_{i,2}$ and principal direction $v_{i,1}$, it is split into two primitives with scales $(\sigma_{i,1}/2, \sigma_{i,2})$ and means $\mu_i \pm (\sigma_{i,1}/2) v_{i,1}$ [S3]. All other properties, including $\gamma$ values, are copied [S3].
*   **Limit:** As $\gamma_i \to \infty$, the split perfectly halves the quad along its long axis [S3].

## Operating characteristics
*   **Stability:** Phys3DGS requires careful tuning of regularization weights to prevent instability from erroneous Gaussians [S2, unofficial]. Standard 3DGS is more robust but may require densification tuning [S3].
*   **Convergence:** Exponential decay of MLP learning rates (as in ToRF) helps stabilize long-duration training [S1].
*   **Memory:** The choice of learning rates and regularization does not directly impact memory, but affects the number of iterations required for convergence, indirectly influencing total compute cost [S1][S2, unofficial].
*   **Hardware:** Adam optimizer is memory-intensive due to storing first and second moments for all parameters [S1][S2, unofficial].

## Background
Hyperparameter settings for 3D Gaussian Splatting have evolved from the original Kerbl et al. [13] implementation. The Phys3DGS method [S2, unofficial] introduced specific learning rates and regularization strengths for physically-based inverse rendering. The ToRF method [S1] adapted settings for time-of-flight data, including specific reflectivity and MLP learning rates. The Nexels method [S3] introduced a blended error-based densification strategy with specific splitting formulas.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
[S3] Nexels: Neurally-Textured Surfels for Real-Time Novel View Synthesis with Sparse Primitives, https://doi.org/10.1007/978-3-032-37531-5_9, fetched 2026-10-28
