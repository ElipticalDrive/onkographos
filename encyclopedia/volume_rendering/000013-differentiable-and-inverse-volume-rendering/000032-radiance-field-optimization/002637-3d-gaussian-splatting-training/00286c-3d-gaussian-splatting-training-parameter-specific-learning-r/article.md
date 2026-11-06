# 3D Gaussian Splatting Training > Parameter-Specific Learning Rates

Also called: Adam parameter learning rates, Gaussian Optimizer Configuration, Per-parameter group learning rates, Training Optimizer Setup, X-Gaussian Hyperparameters, X-Gaussian Implementation Details

## Overview
Configures an Adam optimizer with distinct learning rates for each Gaussian parameter group to stabilize training and prevent rapid divergence [S2, unofficial][S1]. It sets up the optimizer with a base learning rate of 0.0, relying entirely on per-group overrides defined in the parameter groups [S2, unofficial]. This approach addresses the varying sensitivity of Gaussian attributes (position, scale, color, opacity) to gradient updates [S2, unofficial].

## How it works
**Inputs:**
- A list of Gaussian parameters to optimize, typically including means (3D positions), scales, rotations, opacities, and colors (SH coefficients).
- Configuration flags indicating optimization mode (e.g., isotropic vs. anisotropic).

**Outputs:**
- An initialized `torch.optim.Adam` optimizer instance configured with specific learning rates for each parameter group.

**Data Structures:**
- **Parameter Groups:** A list of dictionaries passed to the optimizer. Each dictionary contains:
  - `params`: A list of tensors (the specific Gaussian attributes).
  - `lr`: The learning rate for this group.
- **Gaussian State:**
  - `means`: 3D position values.
  - `scales`: Pre-activation scale values.
  - `opacities`: Pre-activation opacity values.
  - `colours`: RGB color values (or SH coefficients).
  - `quaternions`: Rotation representations (excluded in isotropic mode).

**Steps:**
1. **Verify Trainability:** Ensure all required parameters are marked as trainable. If `isotropic=True`, quaternions must not be marked as trainable; otherwise, raise a `RuntimeError` [S2, unofficial].
2. **Define Parameter Groups:** Create groups for each attribute with specific learning rates.
3. **Initialize Optimizer:** Instantiate `torch.optim.Adam` with `lr=0.0` and `eps=1e-15` [S2, unofficial]. The actual learning rates are applied via the parameter groups.

**Formulas and Values:**
The Adam optimizer uses the following hyperparameters [S1]:
$$ \beta_1 = 0.9, \quad \beta_2 = 0.999, \quad \epsilon = 1 \times 10^{-15} $$

Learning rates vary by source and parameter:

| Parameter | Learning Rate | Source | Notes |
|-----------|---------------|--------|-------|
| Opacities | $8 \times 10^{-3}$ | [S1] | |
| Opacities | 0.00085 | [S2, unofficial] | |
| Opacities | 0.05 | [S2, unofficial] | For `pre_act_opacities` |
| Scales | $5 \times 10^{-3}$ | [S1] | |
| Scales | 0.001 | [S2, unofficial] | |
| Scales | 0.05 | [S2, unofficial] | For `pre_act_scales` |
| Colours | $2 \times 10^{-3}$ | [S1] | For point features |
| Colours | 0.02 | [S2, unofficial] | |
| Colours | 0.05 | [S2, unofficial] | For RGB colors |
| Means | $1.9 \times 10^{-4}$ | [S1] | Initial value |
| Means | 0.00015 | [S2, unofficial] | |
| Means | 0.05 | [S2, unofficial] | For 3D positions |
| Rotation | $1 \times 10^{-3}$ | [S1] | |

**Decay Schedule:**
In the X-Gaussian implementation, the learning rate for position decays exponentially from $1.9 \times 10^{-4}$ to $1.9 \times 10^{-6}$ over $2 \times 10^4$ iterations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def setup_optimizer(gaussians, isotropic=True):
    """
    Initialize Adam optimizer with parameter-specific learning rates.
    
    Args:
        gaussians: Object containing Gaussian attributes.
        isotropic: Boolean flag for isotropic mode.
        
    Returns:
        torch.optim.Adam optimizer instance.
    """
    # Step 1: Verify trainability
    # Check if all required parameters are marked as trainable
    # [G] Assuming gaussians.check_if_trainable() validates the state
    if not gaussians.check_if_trainable():
        raise RuntimeError("Not all parameters are marked as trainable.")
        
    # Pitfall: In isotropic mode, quaternions are excluded from optimization
    if isotropic:
        # [G] Ensure quaternions are not in the trainable list
        # This check is implicit in check_if_trainable or handled separately
        pass
        
    # Step 2: Define parameter groups with specific learning rates
    # Note: Learning rates differ between [S1] and [S2, unofficial]. 
    # This implementation follows [S2, unofficial] values for consistency with the 
    # "pre-activation" and specific group naming.
    
    param_groups = []
    
    # Opacities
    # [S2, unofficial] uses 0.00085 for opacities, 0.05 for pre_act_opacities
    # Assuming 'opacities' refers to the stored pre-activation values
    param_groups.append({
        'params': gaussians.opacities,
        'lr': 0.00085  # [S2, unofficial]
    })
    
    # Scales
    # [S2, unofficial] uses 0.001 for scales, 0.05 for pre_act_scales
    param_groups.append({
        'params': gaussians.scales,
        'lr': 0.001  # [S2, unofficial]
    })
    
    # Colours
    # [S2, unofficial] uses 0.02 for colours, 0.05 for RGB colors
    param_groups.append({
        'params': gaussians.colours,
        'lr': 0.02  # [S2, unofficial]
    })
    
    # Means
    # [S2, unofficial] uses 0.00015 for means, 0.05 for 3D positions
    param_groups.append({
        'params': gaussians.means,
        'lr': 0.00015  # [S2, unofficial]
    })
    
    # Step 3: Initialize Adam optimizer
    # Base lr is 0.0; actual rates are in param_groups
    optimizer = torch.optim.Adam(
        param_groups,
        lr=0.0,
        eps=1e-15,
        betas=(0.9, 0.999)
    )
    
    return optimizer
```

**Edge Cases and Pitfalls:**
- **Isotropic Mode:** Quaternions are excluded from optimization. If marked as trainable when `isotropic=True`, `check_if_trainable` raises a `RuntimeError` [S2, unofficial].
- **Precision:** The epsilon value is set to $1 \times 10^{-15}$ to handle potential numerical instability in small gradients [S1][S2, unofficial].
- **Disagreement:** Learning rates for the same parameters (e.g., means, opacities) differ significantly between [S1] and [S2, unofficial]. [S1] reports higher rates for means ($1.9 \times 10^{-4}$) and opacities ($8 \times 10^{-3}$), while [S2, unofficial] reports lower rates for means (0.00015) and opacities (0.00085). Implementers should choose based on the specific training scenario (e.g., X-ray vs. standard RGB) [S1][S2, unofficial].

## Operating characteristics
- **Time Complexity:** Initialization is $O(N)$ where $N$ is the number of parameter groups. The optimizer step complexity is dominated by the Adam update, which is $O(1)$ per parameter per iteration.
- **Memory Complexity:** $O(N)$ for storing the first and second moment estimates for each parameter.
- **Strengths:** Stabilizes training by preventing large updates to sensitive parameters like position and scale [S2, unofficial].
- **Limitations:** Requires careful tuning of per-group rates. Disagreement between sources makes it difficult to provide a single "best" set of values [S1][S2, unofficial].
- **When to use:** When training 3D Gaussians with Adam, especially if divergence is observed with uniform learning rates.

## Background
Originated from the 3D Gaussian Splatting framework [S2, unofficial]. The X-Gaussian variant adapted these hyperparameters for X-ray novel view synthesis [S1]. The method is part of the broader effort to optimize radiance fields using differentiable rendering [S1][S2, unofficial].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
[S2] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
