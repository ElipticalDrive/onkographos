# Optimization Configuration
Also called: LE3D Optimization Settings, Optimizer and LR Settings

## Overview
This configuration defines the hyperparameter schedule and optimizer state for training radiance field models, specifically targeting the color MLP and Gaussian attributes. It addresses the challenge of converging in highly non-convex, high-dimensional optimization landscapes by combining the adaptive gradient estimation of Adam with a cosine annealing learning rate schedule [S2, unofficial][S1]. It distinguishes itself from standard fixed-rate Adam by enforcing a strict decay to a final learning rate of $1.0 \times 10^{-5}$, which helps stabilize training in the final stages [S1].

## How it works
**Inputs:**
- Model parameters: Color MLP weights ($\theta$), Gaussian color features, Gaussian biases.
- Hyperparameters: Initial learning rates, final learning rate, decay schedule type.

**Data Structures:**
- Optimizer State: Adam maintains first moment ($m$) and second moment ($v$) estimates for each parameter group.
- Learning Rate Schedule: A cosine function mapping training step $t$ to $\eta_t$.

**Parameters:**
- Initial Learning Rate for Color MLP ($\eta_{MLP}^{init}$): $1.0 \times 10^{-4}$ [S1].
- Initial Learning Rate for Gaussian Color Features ($\eta_{feat}^{init}$): $2.0 \times 10^{-3}$ [S1].
- Initial Learning Rate for Gaussian Biases ($\eta_{bias}^{init}$): $1.0 \times 10^{-4}$ [S1].
- Final Learning Rate ($\eta_{final}$): $1.0 \times 10^{-5}$ for all groups [S1].
- Decay Schedule: Cosine decay [S1].
- Loss Regularization Weights (contextual, not optimizer params): $\lambda_F = 10$, $\lambda_T = 0.01$, $\lambda_{dist} = 0.1$, $\lambda_{nf} = 0.01$ [S1].

**Steps:**
1. Initialize Adam optimizer with the specified initial learning rates for each parameter group (MLP, features, biases) [S1].
2. For each training iteration $t$ from $0$ to $T_{total}$:
   a. Compute the learning rate $\eta_t$ using cosine annealing:
      $$ \eta_t = \eta_{final} + \frac{1}{2}(\eta_{init} - \eta_{final}) \left( 1 + \cos\left( \frac{t}{T_{total}} \pi \right) \right) $$
      where $\eta_{init}$ is the specific initial rate for the parameter group [S1].
   b. Compute gradients of the loss function with respect to the parameters.
   c. Update the Adam state ($m, v$) and parameters using $\eta_t$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def get_cosine_lr(init_lr, final_lr, current_step, total_steps):
    """
    Computes the learning rate for a specific parameter group using cosine decay.
    """
    if total_steps == 0:
        return init_lr
    # Cosine annealing from init_lr to final_lr
    progress = current_step / total_steps
    cosine_term = 0.5 * (1 + math.cos(math.pi * progress))
    lr = final_lr + 0.5 * (init_lr - final_lr) * cosine_term
    return lr

def configure_optimizer(params_mlp, params_features, params_biases, total_steps):
    """
    Sets up the Adam optimizer with selective initial learning rates and cosine decay.
    """
    # Hyperparameters from [S1]
    lr_mlp_init = 1.0e-4
    lr_feat_init = 2.0e-3
    lr_bias_init = 1.0e-4
    lr_final = 1.0e-5
    
    # Initialize Adam optimizer with group-specific initial LR
    # Note: Standard Adam implementation handles the decay schedule externally or via a wrapper
    optimizer = Adam([
        {'params': params_mlp, 'lr': lr_mlp_init},
        {'params': params_features, 'lr': lr_feat_init},
        {'params': params_biases, 'lr': lr_bias_init}
    ])
    
    return optimizer, lr_mlp_init, lr_feat_init, lr_bias_init, lr_final

def update_step(optimizer, params_mlp, params_features, params_biases, gradients, step, total_steps):
    """
    Performs one optimization step with dynamic learning rates.
    """
    # Calculate current LRs for each group
    lr_mlp = get_cosine_lr(1.0e-4, 1.0e-5, step, total_steps)
    lr_feat = get_cosine_lr(2.0e-3, 1.0e-5, step, total_steps)
    lr_bias = get_cosine_lr(1.0e-4, 1.0e-5, step, total_steps)
    
    # Update parameter groups with their specific LR
    # This requires a custom optimizer step or a scheduler that modifies the LR in the param groups
    # Standard PyTorch-style update:
    optimizer.step() # Assumes scheduler has updated optimizer.param_groups['lr']
    
    # If using a manual scheduler approach:
    # optimizer.param_groups[0]['lr'] = lr_mlp
    # optimizer.param_groups[1]['lr'] = lr_feat
    # optimizer.param_groups[2]['lr'] = lr_bias
    # optimizer.step()
```

**Edge Cases:**
- If $T_{total}$ is not known in advance, the cosine decay cannot be strictly applied as defined [S1].
- The "Selective L2" aspect refers to the specific application of regularization weights ($\lambda$) to different loss components (depth distortion, near-far) rather than a standard L2 weight decay on the optimizer itself, as the sources do not specify a global L2 weight decay coefficient for Adam [S1].

## Subtopics

### Optimizer Selection
Adam is selected over SGD or BFGS because the optimization problems are highly non-convex and start far from the global optimum, making simplifying assumptions in quasi-Newton methods (like BFGS) inapplicable [S2, S2, unofficial]. Adam provides a cheap yet powerful alternative for high-dimensional objectives [S2, S2, unofficial].

## Operating characteristics
- **Time Complexity:** Adam adds $O(1)$ overhead per parameter per step for maintaining moments, similar to SGD but with higher memory usage.
- **Memory Complexity:** Requires storing first and second moment estimates for all parameters, doubling the memory footprint for optimizer states compared to SGD [S2, unofficial].
- **Performance:** Effective for non-convex radiance field optimization, avoiding bad local optima associated with simpler solvers [S2, unofficial].
- **Limitations:** The non-convex nature of the problem means convergence is not guaranteed to the global optimum [S2, unofficial].

## Background
This configuration is derived from practices in modern radiance field reconstruction, specifically noted in the context of "Lighting Every Darkness with 3DGS" [S1] and "ERF: Explicit Radiance Field Reconstruction From Scratch" [S2, unofficial]. The use of Adam with recommended settings is a standard baseline in deep learning for inverse rendering [S2, unofficial]. The specific cosine decay schedule to $1.0 \times 10^{-5}$ is a refinement observed in recent 3D Gaussian Splatting variants [S1].

## Sources
[S1] Lighting Every Darkness with 3DGS: Fast Training and Real-Time Rendering for HDR View Synthesis, https://doi.org/10.52202/079017-2549, fetched 2026-10-26
[S2] ERF: Explicit Radiance Field Reconstruction From Scratch, https://arxiv.org/abs/2203.00051v1, fetched 2026-10-26 (unofficial)
