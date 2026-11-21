# Per-Parameter Learning Rate Scheduling

Also called: Feature LR, Opacity LR, Other Learning Rate Adjustments, Rotation LR, Scaling LR

## Overview
Per-Parameter Learning Rate Scheduling adjusts the learning rates for specific parameter groups (scaling, opacity, rotation, and features) independently during each training iteration [S1, unofficial]. It addresses the need to tune convergence speed and final rendering quality by allowing distinct optimization dynamics for different geometric and radiometric properties, setting it apart from uniform scheduling methods that apply a single rate to all parameters.

## How it works
**Inputs and Outputs:**
- Input: Current training iteration $t$, initial learning rates for each parameter group ($\eta_{scale}, \eta_{opacity}, \eta_{rotation}, \eta_{feature}$), and the base learning rate schedule.
- Output: Updated learning rates for each parameter group for the current iteration.

**Data Structures:**
- A dictionary or object mapping parameter group names to their current learning rate values.
- Fields: `scale_lr`, `opacity_lr`, `rotation_lr`, `feature_lr`.

**Steps:**
1. Initialize learning rates for each parameter group. For scaling, the optimal range is typically between $10^{-3}$ and $10^{-6}$ [S1, unofficial]. Other parameters (opacity, rotation, feature) can be initialized similarly or derived from a base rate.
2. For each training iteration $t$:
   a. Determine the scaling factor or decay function for each parameter group. This may involve exponential decay, step decay, or constant values depending on the specific variant.
   b. Update the learning rate for each group:
      $$ \eta_{group}(t) = \eta_{group, initial} \times \text{decay}(t) $$
      where $\text{decay}(t)$ is specific to the group and iteration.
   c. Apply the updated learning rates to the respective parameter groups in the optimizer (e.g., Adam).

**Parameters:**
- $\eta_{scale, initial}$: Initial learning rate for scaling parameters. Typical optimal range: $[10^{-3}, 10^{-6}]$ [S1, unofficial].
- $\eta_{opacity, initial}$: Initial learning rate for opacity parameters.
- $\eta_{rotation, initial}$: Initial learning rate for rotation parameters.
- $\eta_{feature, initial}$: Initial learning rate for feature parameters.

**Edge Cases and Pitfalls:**
- Setting $\eta_{feature} = 0$ does not significantly change training speed or quality, indicating that feature learning has less impact on improvement compared to other parameters [S1, unofficial].
- Precision issues may arise if learning rates are too small, leading to negligible updates.
- Boundary conditions: Ensure learning rates do not drop below a minimum threshold to prevent stagnation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_per_parameter_lrs(iteration, initial_lrs, decay_functions):
    """
    Update learning rates for each parameter group based on iteration and decay functions.
    
    Args:
    iteration (int): Current training iteration.
    initial_lrs (dict): Dictionary with initial learning rates for 'scale', 'opacity', 'rotation', 'feature'.
    decay_functions (dict): Dictionary with decay functions for each parameter group.
    
    Returns:
    dict: Updated learning rates for each parameter group.
    """
    updated_lrs = {}
    for group in ['scale', 'opacity', 'rotation', 'feature']:
        initial_lr = initial_lrs[group]
        decay_func = decay_functions[group]
        updated_lrs[group] = initial_lr * decay_func(iteration)
    return updated_lrs

# Example decay functions
def constant_decay(iteration):
    return 1.0

def exponential_decay(iteration, gamma=0.99):
    return gamma ** iteration

# Example usage
initial_lrs = {
    'scale': 1e-3,
    'opacity': 1e-2,
    'rotation': 1e-2,
    'feature': 1e-2
}

decay_functions = {
    'scale': exponential_decay,
    'opacity': constant_decay,
    'rotation': constant_decay,
    'feature': constant_decay
}

# Update learning rates for iteration 100
updated_lrs = update_per_parameter_lrs(100, initial_lrs, decay_functions)
```

## Subtopics

### Stepwise Constant Learning Rate Schedule
Modifies the positional learning rate schedule to align with DashGaussian's two-stage (LR/HR) optimization, holding it constant during LR training and decaying it only after a specific iteration threshold [S2]. The positional learning rate is kept constant at its initial value throughout the LR optimization stage and only allowed to decay from the iteration $i = \arg\min_i \lfloor r(i) \rfloor = 1/3$ [S2]. This approach is beneficial when using a resolution scheduler that transitions from low to high resolution, ensuring stable optimization during the initial phase.

## Operating characteristics
- **Time Complexity:** The overhead of updating per-parameter learning rates is minimal, typically $O(1)$ per iteration, as it involves simple arithmetic operations.
- **Memory Complexity:** Negligible, as it only requires storing a few additional scalar values for each parameter group.
- **Measured Performance:** Adjusting the scaling learning rate within the range $[10^{-3}, 10^{-6}]$ can explicitly change the final rendering result and training speed [S1, unofficial]. Other parameters have less impact on quality and speed but are still useful.
- **Strengths:** Allows fine-tuned control over different aspects of the model, potentially leading to better convergence and higher quality results.
- **Limitations:** Requires careful tuning of initial learning rates and decay functions for each parameter group. Setting feature learning rate to zero may not significantly impact performance, suggesting limited benefit from tuning this parameter.
- **When to Use:** When different parameters require different optimization dynamics, such as in complex scenes where geometric and radiometric properties need distinct handling.
- **When Not to Use:** When computational resources are limited, or when the impact of per-parameter tuning is negligible for the specific application.

## Background
Per-Parameter Learning Rate Scheduling originated from the need to optimize 3D Gaussian Splatting (3DGS) and related radiance field representations more effectively. The method was introduced in I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions [S1, unofficial], which explored the impact of adjusting learning rates for scaling, opacity, rotation, and feature parameters. The approach has since been adopted and adapted in various other works, such as DashGaussian [S2], to enhance optimization efficiency and rendering quality.

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S2] DashGaussian: Optimizing 3D Gaussian Splatting in 200 Seconds, https://doi.org/10.1109/cvpr52734.2025.01041, fetched 2026-10-31
