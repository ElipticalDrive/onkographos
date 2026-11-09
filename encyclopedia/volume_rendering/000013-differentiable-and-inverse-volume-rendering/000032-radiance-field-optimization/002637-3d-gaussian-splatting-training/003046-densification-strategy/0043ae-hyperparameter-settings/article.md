# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter Settings define the numerical configuration for the differentiable optimization loop in 3D Gaussian Splatting variants, specifically governing the Adam optimizer, learning rates for geometric and photometric attributes, and regularization strengths. This method ensures stable convergence by preventing gradient overflow and material-lighting ambiguities that arise from erroneous Gaussian behavior, such as extreme opacity or scale updates [S2, unofficial][S1]. It is distinct from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the *values* of the optimization constants rather than the *logic* of primitive management.

## How it works
The method configures the training loop inputs (loss functions, Gaussian attributes) and outputs (optimized Gaussian parameters) by setting specific constants for the optimizer and regularizers.

**Data Structures and Parameters:**
The configuration consists of scalar hyperparameters for the Adam optimizer and a mapping of learning rates to specific Gaussian attribute fields.

1.  **Optimizer Configuration:**
    *   Algorithm: Adam [S1][S2, unofficial].
    *   Momentum coefficients: $\beta_1 = 0.9$, $\beta_2 = 0.999$ [S1][S2, unofficial].
    *   Epsilon: $\epsilon = 10^{-15}$ [S1][S2, unofficial].

2.  **Learning Rate (LR) Mapping:**
    Each learnable parameter $\theta$ has an associated learning rate $\eta_\theta$. The update rule for each parameter follows the standard Adam algorithm using the configured $\beta$ and $\epsilon$ [S1][S2, unofficial].

    | Parameter | Learning Rate ($\eta$) | Source |
    | :--- | :--- | :--- |
    | Position | $1.6 \times 10^{-4}$ | [S2, unofficial] |
    | Color SH (Spherical Harmonics) | $2.5 \times 10^{-3}$ | [S2, unofficial] |
    | Scale | $5 \times 10^{-3}$ | [S2, unofficial] |
    | Opacity | $5 \times 10^{-2}$ | [S2, unofficial] |
    | Base Color | $1 \times 10^{-2}$ | [S2, unofficial] |
    | Metallic | $1 \times 10^{-2}$ | [S2, unofficial] |
    | Roughness | $1 \times 10^{-2}$ | [S2, unofficial] |
    | Normal Rotation | $1 \times 10^{-3}$ | [S2, unofficial] |
    | Direct Light SH (deg=0) | $2.5 \times 10^{-3}$ | [S2, unofficial] |
    | Direct Light SH (deg>0) | $1 \times 10^{-4}$ | [S2, unofficial] |
    | Indirect Light SH (deg=0) | $2.5 \times 10^{-4}$ | [S2, unofficial] |
    | Indirect Light SH (deg>0) | $1 \times 10^{-3}$ | [S2, unofficial] |
    | Visibility SH | $0.1$ | [S2, unofficial] |
    | Reflectivity | $1.6 \times 10^{-4}$ | [S1] |
    | MLP Weights | Decaying (see below) | [S1] |

3.  **Regularization Strengths:**
    *   Smoothness regularizer for Base Color: $\lambda_{smooth} = 0.06$ [S2, unofficial].
    *   Smoothness regularizer for Roughness: $\lambda_{smooth} = 0.02$ [S2, unofficial].
    *   Smoothness regularizer for Metallic: $\lambda_{smooth} = 0.02$ [S2, unofficial].
    *   Other regularizers (Opacity, Scale, etc.): $\lambda_o = \lambda_{sc} = \lambda_{sr} = 1.0$ [S2, unofficial].
    *   Loss weighting (if applicable): $\alpha = 5$ (or 1 for synthetic), $\beta = 0.0008$ [S1].

4.  **Initialization:**
    *   Reflectivity initialized to $0.1$ [S1].
    *   MLP weights initialized from a normal distribution with standard deviation $10^{-5}$ to ensure small initial deformations [S1].

5.  **Learning Rate Schedules:**
    *   **MLP Learning Rate:** Starts at $\gamma_{MLP} = 0.0008$ and decays exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations [S1].
    *   **Reflectivity Learning Rate:** Constant $\gamma_r = 0.00016$ [S1].

**Pitfalls and Edge Cases:**
*   **Instability from Erroneous Gaussians:** If a single Gaussian behaves abnormally (e.g., driven to the opposite side of the surface, large scale, or extreme opacity), it causes training instability [S2, unofficial].
    *   *Mechanism:* Erroneous Gaussians interfere with visibility, occluding global illumination. The optimizer compensates by increasing local illumination intensity or decreasing opacity to extreme values [S2, unofficial].
    *   *Result:* Global illumination SH coefficients or opacity values receive large gradients, leading to overflow [S2, unofficial].
    *   *Mitigation:* High regularization strengths ($\lambda_o, \lambda_{sc}, \lambda_{sr} = 1.0$) are used to penalize such deviations [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration constants
OPTIMIZER_BETA1 = 0.9
OPTIMIZER_BETA2 = 0.999
OPTIMIZER_EPS = 1e-15

# Learning rates dictionary
LEARNING_RATES = {
    "position": 1.6e-4,
    "color_sh": 2.5e-3,
    "scale": 5e-3,
    "opacity": 5e-2,
    "base_color": 1e-2,
    "metallic": 1e-2,
    "roughness": 1e-2,
    "normal_rotation": 1e-3,
    "direct_light_sh_deg0": 2.5e-3,
    "direct_light_sh_deg_gt0": 1e-4,
    "indirect_light_sh_deg0": 2.5e-4,
    "indirect_light_sh_deg_gt0": 1e-3,
    "visibility_sh": 0.1,
    "reflectivity": 1.6e-4
}

# Regularization strengths
LAMBDA_SMOOTH_BASE_COLOR = 0.06
LAMBDA_SMOOTH_ROUGHNESS = 0.02
LAMBDA_SMOOTH_METALLIC = 0.02
LAMBDA_OTHER = 1.0

# MLP schedule parameters
MLP_LR_START = 0.0008
MLP_LR_END = 1.6e-6
MLP_DECAY_ITERATIONS = 60000

def get_mlp_lr(current_iteration):
    """
    Exponential decay for MLP learning rate.
    """
    if current_iteration >= MLP_DECAY_ITERATIONS:
        return MLP_LR_END
    # Exponential decay formula: start * (end/start)^(iter/total)
    ratio = MLP_LR_END / MLP_LR_START
    decay_factor = ratio ** (current_iteration / MLP_DECAY_ITERATIONS)
    return MLP_LR_START * decay_factor

def initialize_gaussians(gaussians):
    """
    Initialize specific attributes according to source settings.
    """
    for g in gaussians:
        # Reflectivity initialization
        g.reflectivity = 0.1
        # MLP weights (if applicable) initialized with std 1e-5
        # g.mlp_weights = torch.randn_like(g.mlp_weights) * 1e-5
        pass

def compute_loss_with_regularization(loss, gaussians):
    """
    Apply regularization terms based on hyperparameter settings.
    """
    total_loss = loss
    
    # Smoothness regularization for physical attributes
    # Note: Exact smoothness term definition is not provided in sources [S2, unofficial]
    # Assuming a standard Laplacian or neighbor-difference smoothness term
    smoothness_loss = compute_smoothness_loss(gaussians)
    
    total_loss += LAMBDA_SMOOTH_BASE_COLOR * smoothness_loss.get("base_color")
    total_loss += LAMBDA_SMOOTH_ROUGHNESS * smoothness_loss.get("roughness")
    total_loss += LAMBDA_SMOOTH_METALLIC * smoothness_loss.get("metallic")
    
    # Other regularizers (opacity, scale, etc.)
    # Note: Specific formulas for lambda_o, lambda_sc, lambda_sr are not provided [S2, unofficial]
    other_regularization_loss = compute_other_regularization(gaussians)
    total_loss += LAMBDA_OTHER * other_regularization_loss
    
    return total_loss
```

## Subtopics

### [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md)
Adapts 3DGS training for 7D data (3D spatial, 1D temporal, 3D directional) by modifying pruning thresholds and splitting logic to account for time and view-direction modulation.

## Operating characteristics
*   **Stability:** High regularization strengths ($\lambda=1.0$) are critical to prevent gradient overflow caused by erroneous Gaussians [S2, unofficial].
*   **Convergence:** The MLP learning rate decay over 60,000 iterations ensures stable optimization for dynamic scenes [S1].
*   **Hardware/Resolution:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Prevents training collapse in inverse rendering tasks by managing material-lighting ambiguity and visibility errors [S2, unofficial].
*   **Limitations:** Requires careful tuning of per-parameter learning rates; incorrect values can lead to instability [S2, unofficial].

## Background
*   **Origin:** The settings are derived from implementations of Phys3DGS [S2, unofficial] and Time of the Flight of the Gaussians (ToRF) [S1].
*   **History:** These settings were developed to address instability in differentiable volume rendering and inverse rendering, specifically regarding the optimization of physically based attributes (roughness, metallic) and dynamic deformations [S1][S2, unofficial].
*   **Usage:** Used in 3D Gaussian Splatting variants for inverse rendering and dynamic scene reconstruction [S1][S2, unofficial].

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
