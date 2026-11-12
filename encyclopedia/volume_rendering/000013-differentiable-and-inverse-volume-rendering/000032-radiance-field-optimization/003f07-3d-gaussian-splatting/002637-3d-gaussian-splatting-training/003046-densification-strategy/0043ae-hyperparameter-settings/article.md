# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter settings define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting (3DGS) and its variants. These settings are critical for stabilizing the optimization process, preventing gradient overflow, and ensuring convergence to a physically plausible or high-fidelity reconstruction. They differ from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the *magnitude* and *decay* of updates rather than the *structural* modification of the Gaussian set, though they operate concurrently within the training loop.

## How it works
The method configures the optimization landscape through three primary components: the optimizer algorithm, per-parameter learning rates, and regularization weights.

**Inputs and Outputs**
*   **Inputs:** Model parameters (positions, scales, opacities, SH coefficients, material properties), loss functions (reconstruction, smoothness, regularization), and hyperparameter constants.
*   **Outputs:** Updated model parameters after one or more optimization steps.

**Data Structures**
The hyperparameters are stored as scalar constants or vectors indexed by parameter type.
*   **Optimizer State:** For Adam, this includes exponential moving averages of gradients ($m$) and squared gradients ($v$) for each parameter tensor [S1][S2, unofficial].
*   **Learning Rate Vector:** A mapping from parameter name to scalar learning rate $\eta$ [S2, unofficial].
*   **Regularization Weights:** Scalars $\lambda$ for each regularization term [S2, unofficial].

**Numbered Steps**
1.  **Initialization:**
    *   Initialize model parameters. For deformable models, layer weights are initialized from a normal distribution with $\sigma = 10^{-5}$ to ensure small initial deformations [S1].
    *   Initialize reflectivity to $0.1$ [S1].
2.  **Loss Computation:**
    *   Compute the primary reconstruction loss (e.g., RGB, C-ToF) and auxiliary losses (smoothness, regularization) [S1][S2, unofficial].
3.  **Gradient Calculation:**
    *   Compute gradients $\nabla_\theta L$ for all learnable parameters $\theta$ via backpropagation [S1][S2, unofficial].
4.  **Regularization Application:**
    *   Add regularization terms to the loss or gradients. For Phys3DGS, smoothness regularizers are applied to base color, metallic, and roughness [S2, unofficial].
    *   If a Gaussian behaves abnormally (e.g., extreme scale/opacity), it can cause visibility interference and gradient overflow in SH coefficients or opacity [S2, unofficial]. High regularization weights ($\lambda_o, \lambda_{sc}, \lambda_{sr} = 1.0$) are used to mitigate this instability [S2, unofficial].
5.  **Parameter Update (Adam):**
    *   Update the exponential moving averages:
        $$ m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t $$
        $$ v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2 $$
    *   Apply bias correction and update parameters:
        $$ \hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} $$
        $$ \theta_{t+1} = \theta_t - \eta \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} $$
    *   Where $g_t$ is the gradient, $\eta$ is the learning rate for the specific parameter, and $\beta_1, \beta_2, \epsilon$ are optimizer hyperparameters [S1][S2, unofficial].
6.  **Learning Rate Decay (Optional):**
    *   For MLP parameters in deformable models, decay the learning rate exponentially from $\gamma_{MLP} = 0.0008$ to $1.6 \times 10^{-6}$ over 60,000 iterations [S1].

**Formulas and Parameters**
*   **Optimizer:** Adam with $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-15}$ [S1][S2, unofficial].
*   **Loss Weights:** $\alpha = 5$ (1 for synthetic scenes) for C-ToF loss, $\beta = 0.0008$ for color loss [S1].
*   **Regularization Strengths ($\lambda_{smooth}$):**
    *   Base Color: $0.06$ [S2, unofficial].
    *   Roughness: $0.02$ [S2, unofficial].
    *   Metallic: $0.02$ [S2, unofficial].
    *   Other regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr}$): $1.0$ [S2, unofficial].
*   **Learning Rates ($\eta$):**
    *   Position: $1.6 \times 10^{-4}$ [S2, unofficial].
    *   Color SH: $2.5 \times 10^{-3}$ [S2, unofficial].
    *   Scale: $5 \times 10^{-3}$ [S2, unofficial].
    *   Opacity: $5 \times 10^{-2}$ [S2, unofficial].
    *   Base Color: $1 \times 10^{-2}$ [S2, unofficial].
    *   Metallic: $1 \times 10^{-2}$ [S2, unofficial].
    *   Roughness: $1 \times 10^{-2}$ [S2, unofficial].
    *   Normal Rotation: $1 \times 10^{-3}$ [S2, unofficial].
    *   Direct Light SH (deg=0): $2.5 \times 10^{-3}$ [S2, unofficial].
    *   Direct Light SH (deg>0): $1 \times 10^{-4}$ [S2, unofficial].
    *   Indirect Light SH (deg=0): $2.5 \times 10^{-4}$ [S2, unofficial].
    *   Indirect Light SH (deg>0): $1 \times 10^{-3}$ [S2, unofficial].
    *   Visibility SH: $0.1$ [S2, unofficial].
    *   Reflectivity: $\gamma_r = 0.00016$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_parameters(params, gradients, optimizer_state, step, hyperparams):
    """
    Updates 3DGS parameters using Adam optimizer with specific learning rates and regularization.
    """
    beta1 = hyperparams['beta1']  # 0.9
    beta2 = hyperparams['beta2']  # 0.999
    epsilon = hyperparams['epsilon']  # 1e-15
    
    for param_name, param in params.items():
        grad = gradients[param_name]
        
        # Get specific learning rate for this parameter
        lr = hyperparams['learning_rates'][param_name]
        
        # Handle MLP learning rate decay if applicable
        if param_name == 'MLP_weights':
            initial_lr = 0.0008
            final_lr = 1.6e-6
            total_steps = 60000
            # Exponential decay formula
            lr = initial_lr * (final_lr / initial_lr) ** (step / total_steps)
            
        # Update Adam state
        m = optimizer_state[param_name]['m']
        v = optimizer_state[param_name]['v']
        
        m = beta1 * m + (1 - beta1) * grad
        v = beta2 * v + (1 - beta2) * grad ** 2
        
        # Bias correction
        m_hat = m / (1 - beta1 ** (step + 1))
        v_hat = v / (1 - beta2 ** (step + 1))
        
        # Parameter update
        param = param - lr * m_hat / (sqrt(v_hat) + epsilon)
        
        # Store updated state
        optimizer_state[param_name]['m'] = m
        optimizer_state[param_name]['v'] = v
        
    return params, optimizer_state
```

**Edge Cases and Pitfalls**
*   **Gradient Overflow:** Abnormal Gaussians (e.g., driven to the opposite side of a surface or with extreme opacity) can occlude global illumination, causing the optimizer to compensate with extreme intensity/opacity changes. This leads to gradient overflow in SH coefficients or opacity values [S2, unofficial].
*   **Instability:** Without sufficient regularization ($\lambda_o, \lambda_{sc}, \lambda_{sr} = 1.0$), a single erroneous Gaussian can destabilize the entire training process [S2, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** The hyperparameter settings themselves add negligible overhead to the optimization step, dominated by the Adam state update and gradient computation.
*   **Memory Complexity:** Adam requires storing two additional tensors ($m$ and $v$) per parameter, doubling the memory footprint for optimizer states [S1][S2, unofficial].
*   **Strengths:** Specific learning rates allow fine-grained control over different aspects of the Gaussian (e.g., high LR for opacity to allow rapid pruning, low LR for position for stability). Regularization prevents divergence from pathological Gaussian configurations [S2, unofficial].
*   **Limitations:** Poorly chosen learning rates can lead to slow convergence or divergence. The exponential decay for MLPs requires careful tuning of the decay rate and total steps [S1].

## Background
Hyperparameter tuning for 3DGS and its variants has evolved alongside the complexity of the representations. The original 3DGS paper established baseline Adam settings. Subsequent works like [Time of the Flight of the Gaussians](https://doi.org/10.1109/cvpr52734.2025.01958) introduced specific learning rates for deformable models and C-ToF losses [S1]. [Phys3DGS](https://doi.org/10.48550/arxiv.2409.10335) introduced detailed learning rates for physically-based parameters (base color, metallic, roughness) and strong regularization to handle inverse rendering ambiguities and instability [S2, unofficial].

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
