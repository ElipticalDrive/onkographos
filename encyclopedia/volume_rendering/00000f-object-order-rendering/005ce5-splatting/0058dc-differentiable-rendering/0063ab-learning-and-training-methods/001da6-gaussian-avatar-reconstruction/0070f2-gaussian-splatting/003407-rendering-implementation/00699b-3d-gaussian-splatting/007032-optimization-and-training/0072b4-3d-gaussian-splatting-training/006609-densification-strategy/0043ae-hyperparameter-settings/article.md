# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter Settings define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting (3DGS) models [S2, unofficial][S1]. This method solves the problem of stable convergence and high-fidelity reconstruction by tuning the trade-off between fitting training data and maintaining geometric/photometric consistency [G]. It differs from sibling methods like [Densification Strategy](6234-densification-strategy.md) (which manages primitive count) and [Pruning and Sparsification](12354-pruning-and-sparsification.md) (which removes primitives) by focusing on the gradient descent dynamics and loss weighting rather than structural topology changes [G].

## How it works
The method configures the optimization loop by setting the optimizer type, learning rates for specific Gaussian attributes, and weights for auxiliary loss terms [S2, unofficial][S1].

**Inputs:**
*   Model parameters: Position $\mu$, Color SH coefficients $\mathbf{c}$, Scale $s$, Opacity $\alpha$, and potentially material parameters (Base Color, Metallic, Roughness) or dynamic parameters (MLP weights) [S2, unofficial][S1].
*   Ground truth images and camera poses [G].

**Outputs:**
*   Optimized Gaussian parameters [G].

**Data Structures:**
*   **Optimizer State:** Adam optimizer state buffers (first and second moment estimates) for each parameter group [G].
*   **Learning Rate Schedule:** A mapping from parameter type to initial learning rate and decay schedule [S2, unofficial][S1].

**Parameters and Typical Values:**

1.  **Optimizer:** Adam [S1][S2, unofficial].
    *   $\beta_1 = 0.9$
    *   $\beta_2 = 0.999$
    *   $\epsilon = 10^{-15}$ [S1][S2, unofficial].

2.  **Learning Rates (LR):**
    *   Standard 3DGS [S2, unofficial]:
        *   Position: $1.6 \times 10^{-4}$
        *   Color SH: $2.5 \times 10^{-3}$
        *   Scale: $5 \times 10^{-3}$
        *   Opacity: $5 \times 10^{-2}$
        *   Normal Rotation: $1 \times 10^{-3}$
    *   Phys3DGS (Inverse Rendering) [S2, unofficial]:
        *   Base Color: $1 \times 10^{-2}$
        *   Metallic: $1 \times 10^{-2}$
        *   Roughness: $1 \times 10^{-2}$
        *   Direct Light SH (deg=0): $2.5 \times 10^{-3}$ [S2, unofficial]
        *   Direct Light SH (deg>0): $1 \times 10^{-4}$ [S2, unofficial]
        *   Indirect Light SH (deg=0): $2.5 \times 10^{-4}$ [S2, unofficial]
        *   Indirect Light SH (deg>0): $1 \times 10^{-3}$ [S2, unofficial]
        *   Visibility SH: $0.1$
    *   ToRF (Dynamic/Depth) [S1]:
        *   MLP weights: Initial $\gamma_{MLP} = 0.0008$, decaying exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations [S1].
        *   Reflectivity: Initial value $0.1$, LR $\gamma_r = 0.00016$.

3.  **Regularization Strengths (Phys3DGS) [S2, unofficial]:**
    *   Smoothness regularizer for Base Color: $\lambda_{smooth} = 0.06$ [S2, unofficial]
    *   Smoothness regularizer for Roughness: $\lambda_{smooth} = 0.02$ [S2, unofficial]
    *   Smoothness regularizer for Metallic: $\lambda_{smooth} = 0.02$ [S2, unofficial]
    *   Other regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr}$): $1.0$ [S2, unofficial]

4.  **Loss Weights (ToRF) [S1]:**
    *   $\alpha = 5$ (for real scenes), $\alpha = 1$ (for synthetic scenes) [S1].
    *   $\beta = 0.0008$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_hyperparameters(params, config):
    """
    Configure optimizer state and learning rates for 3DGS training.
    params: dict mapping parameter names to tensors (numpy-like arrays).
    config: dict containing optimizer settings, lr_schedule, and regularization weights.
    """
    # Initialize Adam state buffers (first and second moments) for each parameter
    optimizer_state = {}
    for name, tensor in params.items():
        optimizer_state[name] = {
            'm': zeros_like(tensor),  # First moment
            'v': zeros_like(tensor)   # Second moment
        }
        
    # Define learning rate mapping based on parameter type
    lr_map = {
        'position': 1.6e-4,
        'color_sh': 2.5e-3,
        'scale': 5.0e-3,
        'opacity': 5.0e-2,
        'base_color': 1.0e-2,
        'metallic': 1.0e-2,
        'roughness': 1.0e-2,
        'normal_rotation': 1.0e-3,
        'direct_light_sh_deg0': 2.5e-3,
        'direct_light_sh_deg_gt0': 1.0e-4,
        'indirect_light_sh_deg0': 2.5e-4,
        'indirect_light_sh_deg_gt0': 1.0e-3,
        'visibility_sh': 0.1,
        'mlp_weights': config.get('mlp_lr', 0.0008),
        'reflectivity': 0.00016
    }
    
    # Assign learning rates to parameter groups
    param_groups = []
    for name, tensor in params.items():
        lr = lr_map.get(name, 1.0e-3)  # Default fallback
        param_groups.append({'params': [tensor], 'lr': lr})
        
    return optimizer_state, param_groups

def compute_loss_and_update(pred, gt, params, gradients, optimizer_state, config, iteration):
    """
    Compute loss with regularization and perform a single Adam update step.
    """
    # Photometric loss (L1)
    photometric_loss = mean(abs(pred - gt))
    
    # Regularization terms
    smoothness_loss = config['lambda_smooth'] * mean(abs(gradients))
    other_reg_loss = 1.0 * mean(abs(params))
    
    total_loss = photometric_loss + smoothness_loss + other_reg_loss
    
    # Backward pass would populate `gradients` dict
    # Adam update step for each parameter
    beta1, beta2, eps = 0.9, 0.999, 1e-15
    for name, tensor in params.items():
        grad = gradients[name]
        state = optimizer_state[name]
        lr = lr_map.get(name, 1.0e-3)
        
        # Exponential decay for MLP weights (ToRF)
        if name == 'mlp_weights':
            lr = 0.0008 * (0.999973 ** iteration)  # Decays to 1.6e-6 over 60k iters
        
        # Adam update
        state['m'] = beta1 * state['m'] + (1 - beta1) * grad
        state['v'] = beta2 * state['v'] + (1 - beta2) * grad ** 2
        m_hat = state['m'] / (1 - beta1 ** (iteration + 1))
        v_hat = state['v'] / (1 - beta2 ** (iteration + 1))
        
        tensor = tensor - lr * m_hat / (sqrt(v_hat) + eps)
        
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Instability from Erroneous Gaussians:** If a single Gaussian behaves abnormally (e.g., driven to the opposite side of the surface, extremely large scale, or extreme opacity), it causes training instability [S2, unofficial].
    *   *Mechanism:* Erroneous Gaussians interfere with visibility, occluding global illumination [S2, unofficial]. The optimizer compensates by either increasing local illumination intensity (dimming global) or decreasing opacity to extreme values [S2, unofficial].
    *   *Result:* Global illumination SH coefficients or opacity values receive large gradients, leading to overflow [S2, unofficial].
    *   *Mitigation:* Use higher weights for other regularizers ($\lambda_o, \lambda_{sc}, \lambda_{sr} = 1.0$) to constrain this behavior [S2, unofficial].
*   **View Selection Bias:** Using training views for optimization focuses on known-view fitting and degrades novel-view performance [S4]. Avoid using training views (+train) as they focus on known-view fitting [S4]. Use test views for optimization [S4]. Alternatively, use synthetic views sampled from the estimated camera trajectory (+syn.) [S4].

## Subtopics

### [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md)
Extends 3DGS to dynamic scenes by conditioning Gaussian covariance and opacity on time $t$ and view direction $d$. It differs from static 3DGS by using a conditional covariance matrix to drive densification and splitting, and from other dynamic variants by integrating these cues directly into the standard 3DGS rasterization pipeline with minimal architectural changes.

### [Blended Error Densification and Splitting](13892-blended-error-densification-and-splitting.md)
Uses an auxiliary per-pixel error metric to drive densification decisions, rather than relying on positional gradient magnitudes. It solves the limitation of gradient-based methods (sensitive to error changes but blind to absolute error in high-frequency areas) by redistributing per-pixel errors to primitives proportional to their rendering contribution. Distinct from siblings like [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md) by explicitly utilizing accumulated L1 error norms as probability weights for splitting.

### [Sparse Matching Integration](14995-sparse-matching-integration.md)
Addresses under-constrained geometry in few-shot neural rendering by integrating precise correspondences from a sparse matching network into 3DGS initialization and optimization. Unlike dense matcher-based methods that introduce blurriness in high-frequency areas, this approach yields precise correspondences that, combined with frequency regularization, enable better reconstruction of fine details. It serves as a densification and initialization strategy enhancing base 3DGS in sparse-view scenarios.

### [Adaptive Gaussian Selector](16104-adaptive-gaussian-selector.md)
A learnable module within the FlexGS framework that adaptively selects a subset of 3D Gaussians during training based on attributes and a desired elastic ratio $e$, using a differentiable mask. It sets itself apart from non-differentiable thresholding or hard masking by allowing gradients to flow through the selection decision via the Straight-Through Estimator (STE) applied to Gumbel-Softmax outputs, unlike sibling methods that rely on fixed heuristics or non-differentiable hard thresholds.

### [Chunk-scale Training](18156-chunk-scale-training.md)
Addresses optimization challenges of large, sparse, unbounded scenes by dividing the environment into independent spatial chunks, each trained separately. It distinguishes itself from standard 3DGS by introducing a skybox for environmental consistency, modifying densification policies to handle sparse gradients, and applying monocular depth regularization for unobserved surfaces.

### [Edge-guided Splitting Strategy](18680-edge-guided-splitting-strategy.md)
Refines primitives by splitting large Gaussians located in high-frequency edge regions to capture fine details. It addresses the weakness of enlarged primitives, which lose high-frequency detail, particularly when used in conjunction with Random Dropout Regularization (RDR). Unlike standard splitting based solely on position gradient magnitude, it explicitly guides splitting toward geometric edges, making it distinct from sibling methods like [Blended Error Densification and Splitting](13892-blended-error-densification-and-splitting.md) which rely on rendering error [13892-blended-error-densification-and-splitting.md].

### [Standard 3DGS Training Configuration](18796-standard-3dgs-training-configuration.md)
Specifies the canonical hyperparameter settings for optimizing 3D Gaussian Splatting models, establishing the baseline for adaptive density control and gradient-based optimization. It differs from sibling methods like [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md) or [Blended Error Densification and Splitting](13892-blended-error-densification-and-splitting.md) by adhering strictly to the default parameters of the official implementation, specifically regarding densification frequency, gradient thresholds, and opacity management.

### [BoundedGS Leaf Loss and Gradient Management](20147-boundedgs-leaf-loss-and-gradient-management.md)
A training optimization that reduces memory overhead during high-resolution rendering by organizing Gaussians into spatial groups (leaves) and using compact gradient storage. It solves the problem of excessive memory consumption caused by maintaining full dense gradients for all Gaussians during the backward pass of large scenes. It differs from sibling methods by explicitly decoupling the gradient accumulation scope to leaf boundaries and using a global ID mapping to feed only necessary gradient rows into the Adam optimizer, while maintaining dense gradients for opacity to support global regularization.

### [Consensus-Inheritance Extension](21245-consensus-inheritance-extension.md)
A mechanism for maintaining correspondence between fixed consensus statistics and newly created Gaussian primitives during densification (splitting or cloning) in 3D Gaussian Splatting. It prevents the breaking of the fixed-consensus protocol by ensuring that new Gaussians inherit relevant prior information, thereby allowing consensus statistics to evolve with the Gaussian population while retaining contamination-resistant properties. It is distinct from sibling methods like [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md) and [Blended Error Densification and Splitting](13892-blended-error-densification-and-splitting.md) by explicitly managing the inheritance of consensus weights and periodic re-estimation thresholds rather than altering the densification criteria or loss functions directly.

### GaMeS
A method simplifying mesh generation by utilizing an initial mesh or estimating it in a single-stage training process, introducing a pseudo-mesh of unconnected triangle faces with Gaussian components positioned on these faces to ensure proper alignment [S3]. It realizes deformation by modifying pseudo-mesh positions, with alterations automatically propagating to the Gaussian mesh, enabling real-time animation and scene editing [S3]. However, it exhibits artifacts when dealing with significant deformations in meshes with large faces, leading to inaccuracies [S3].

### View Selection for Optimization
Selects specific views for adaptation optimization to prevent overfitting to known training views and maintain novel-view performance [S4]. Steps: 1. Avoid using training views (+train) as they focus on known-view fitting [S4]. 2. Use test views for optimization [S4]. 3. Alternatively, use synthetic views sampled from the estimated camera trajectory (+syn.) [S4]. Typical number of synthetic views used: 50 [S4]. On the bicycle scene, PSNR with test views was Zoom-out 30.84, Zoom-in 24.23; with synthetic views: Zoom-out 29.72, Zoom-in 30.05 [S4].

### Peripheral Model Initialization
Creates a set of initial images from various views around the target volume and trains a peripheral model for 700 iterations using a MiniSplatting2-based approach [S5]. Renders training views at 512x512 pixels covering a 20° field of view [S5]. Number of initial views: 8, 12, 16, or 32 [S5]. Samples per pixel (SPP): 4, 8, or 16. Normal quality preset: 12 views at 8 SPP [S5]. High quality preset: 16 views at 16 SPP [S5]. Using only 8 images or 4 SPP often misses important details [S5]. Using more than 16 views significantly delays the availability of the initial model with diminishing returns [S5]. Initial model generation takes under 300 ms; full reconstruction completes in about a second [S5]. Produces a higher-quality model faster than standard 3DGS with half the number of Gaussians, as opacity reset struggles to lower Gaussian counts [S5].

### PUP (Pruning using Uncertainty/Second-order sensitivity)
A pruning method that reduces the number of Gaussians ($N$) using a second-order sensitivity criterion based on the local reconstruction-loss change [S6]. Calculates the local Hessian block $H_j$ for each Gaussian's spatial parameters [S6]. Computes the sensitivity $\Delta L_j$. Removes Gaussians with low sensitivity (low $\Delta L_j$) before refinement [S6]. Formula: $\Delta L_j \approx \frac{1}{2} \delta\theta_j^T H_j \delta\theta_j$, where $\Delta L_j$ is the local reconstruction-loss change for Gaussian $j$, $\delta\theta_j$ is the perturbation of spatial Gaussian parameters $\theta_j$, and $H_j$ is the local Hessian block associated with the spatial parameters of Gaussian $j$ [S6]. On the shared baseline in Table 9, PUP removes approximately 90% of Gaussians and 90% of serialized bytes, changing PSNR from 25.01 to 25.42 dB [S6]. PUP primarily reduces $N$ (primitive count), whereas Compact3D primarily reduces $bg$ (storage per Gaussian) [S6].

## Operating characteristics
*   **Strengths:** Carefully tuned learning rates (e.g., higher LR for opacity and scale) allow for rapid adaptation of primitive properties [S2, unofficial]. Regularization weights prevent instability from outlier Gaussians [S2, unofficial].
*   **Limitations:** Improper regularization weights can lead to gradient overflow in SH coefficients or opacity [S2, unofficial]. Using training views for optimization degrades novel view synthesis quality [S4].
*   **When to use:** Standard 3DGS training requires the base learning rates [S2, unofficial]. Inverse rendering (Phys3DGS) requires specific material and lighting LR settings [S2, unofficial]. Dynamic scenes (ToRF) require MLP decay schedules [S1]. Sparse view scenarios may benefit from view selection strategies [S4].

## Background
The hyperparameter settings for 3D Gaussian Splatting were introduced alongside the original 3DGS paper, establishing the baseline Adam optimizer configuration [S1][S2, unofficial]. Subsequent works have adapted these settings for specific domains: Phys3DGS introduced settings for physically-based inverse rendering [S2, unofficial], ToRF adapted them for time-of-flight and dynamic scenes [S1], and Mip-Splatting variants explored view selection strategies to prevent overfitting to known training views [S4]. The importance of regularization weights to prevent instability from erroneous Gaussians was highlighted in Phys3DGS [S2, unofficial].

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S4] Mipmap-GS: Let Gaussians Deform with Scale-Specific Mipmap for Anti-Aliasing Rendering, https://doi.org/10.1109/3dv66043.2025.00091, fetched 2026-10-27
[S5] Hybrid Foveated Path Tracing with Peripheral Gaussians for Immersive Anatomy, https://doi.org/10.1109/vr67842.2026.00078, fetched 2026-10-26
[S6] Gaussian Splatting for real-time scene reconstruction: Methods, systems, and trade-offs, https://doi.org/10.1016/j.patcog.2026.115120, fetched 2026-10-26
