# Gaussian Parameter Activation

Also called: Gaussian parameterization

## Overview
Gaussian Parameter Activation converts raw neural network outputs (pre-activations) into valid parameters for 3D Gaussian primitives, specifically scales, rotations, opacities, and positions. It differs from standard Density Activation [2421-density-activation.md] by handling multi-dimensional geometric attributes rather than a single scalar density, and from the Density Mapping Layer [7202-density-mapping-layer.md] by explicitly producing Gaussian splat parameters rather than volumetric density fields.

## How it works
The method maps unconstrained network outputs to constrained physical parameters. Two distinct parameterizations exist in the literature: the standard Splatting approach [S1, unofficial] and the ZipSplat approach [S2, unofficial].

### Inputs and Outputs
**Inputs:**
- `pre_act_scales`: Raw tensor of scale logits.
- `pre_act_quats`: Raw tensor of quaternion components.
- `pre_act_opacities`: Raw tensor of opacity logits.
- `pre_act_means`: Raw tensor of position coordinates (ZipSplat only).
- `pre_act_colors`: Raw tensor of color coefficients (ZipSplat only).

**Outputs:**
- `scales`: Valid scale values.
- `quaternions`: Unit-length rotation quaternions.
- `opacities`: Values in $[0, 1]$.
- `means`: Valid position coordinates (ZipSplat only).
- `colors`: Spherical harmonic coefficients (ZipSplat only).

### Data Structures
The raw inputs are typically tensors of shape $(N, D)$, where $N$ is the number of Gaussians and $D$ is the dimensionality of the parameter (e.g., 3 for scales, 4 for quaternions).

### Standard Splatting Activation [S1, unofficial]
This variant applies simple monotonic transformations to ensure valid ranges.

1. **Scales:** Compute scales by exponentiating pre-activations to ensure positivity.
   $$ \text{scales} = \exp(\text{pre\_act\_scales}) $$
2. **Rotations:** Normalize pre-activation quaternions to unit length.
   $$ \text{quats} = \text{normalize}(\text{pre\_act\_quats}) $$
3. **Opacities:** Apply sigmoid activation to bound opacities to $[0, 1]$.
   $$ \text{opacities} = \sigma(\text{pre\_act\_opacities}) $$

### ZipSplat Activation [S2, unofficial]
This variant uses more complex activations to handle specific training dynamics and constraints.

1. **Positions:** Encode positions using inverse-log activation on raw outputs clamped to $[-5, 5]$.
   $$ \text{means} = \operatorname{sign}(y)(\exp|y| - 1), \quad y = \operatorname{clamp}(\text{pre\_act\_means}, -5, 5) $$
2. **Scales:** Encode scales using softplus with a shift, clamped to $[10^{-6}, 15]$.
   $$ \text{scales} = \operatorname{clamp}(\text{softplus}(\text{pre\_act\_scales} - 4), 10^{-6}, 15) $$
3. **Rotations:** Normalize quaternions to unit length.
   $$ \text{quats} = \text{normalize}(\text{pre\_act\_quats}) $$
4. **Opacities:** Apply sigmoid with a bias of $-1.5$.
   $$ \text{opacities} = \sigma(\text{pre\_act\_opacities} - 1.5) $$
5. **Colors:** Encode colors as degree-1 spherical harmonics (12 coefficients).
6. **Initialization:** All 32 Gaussians of a token start from a shared template with a forward bias of $z=0.5$ [S2, unofficial].

### Pitfalls and Regularization [S2, unofficial]
- Scales must be kept in $[10^{-6}, 0.1]$.
- Opacities must remain above $0.01$.
- Positions must remain within a radius of 10.
- These constraints are enforced using squared-hinge penalties during optimization.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def gaussian_activation_standard(pre_act_scales, pre_act_quats, pre_act_opacities):
    """
    Standard Gaussian Parameter Activation [S1, unofficial].
    """
    # Scales: exponential to ensure positivity
    scales = torch.exp(pre_act_scales)
    
    # Rotations: normalize to unit length
    quats = F.normalize(pre_act_quats, p=2, dim=-1)
    
    # Opacities: sigmoid to bound to [0, 1]
    opacities = torch.sigmoid(pre_act_opacities)
    
    return scales, quats, opacities

def gaussian_activation_zipsplat(pre_act_means, pre_act_scales, pre_act_quats, pre_act_opacities):
    """
    ZipSplat Gaussian Parameter Activation [S2, unofficial].
    """
    # Positions: inverse-log activation with clamping
    y = torch.clamp(pre_act_means, min=-5.0, max=5.0)
    means = torch.sign(y) * (torch.exp(torch.abs(y)) - 1.0)
    
    # Scales: softplus with shift and clamping
    scales = torch.clamp(F.softplus(pre_act_scales - 4.0), min=1e-6, max=15.0)
    
    # Rotations: normalize to unit length
    quats = F.normalize(pre_act_quats, p=2, dim=-1)
    
    # Opacities: sigmoid with bias -1.5
    opacities = torch.sigmoid(pre_act_opacities - 1.5)
    
    return means, scales, quats, opacities
```

## Subtopics

### Gaussian Parameterization
A specific configuration used in ZipSplat [S2, unofficial] that employs inverse-log activation for positions, shifted softplus for scales, and biased sigmoid for opacities. It is chosen when precise control over scale ranges and position bounds is required, often in conjunction with squared-hinge penalties. It differs from the standard exponential/normalization approach by providing tighter control over the initial distribution and range of parameters.

## Operating characteristics
- **Time Complexity:** Linear in the number of Gaussians $N$. The operations (exp, sigmoid, normalize) are element-wise or simple vector normalizations.
- **Memory Complexity:** Linear in $N$. No additional large data structures are created beyond the output tensors.
- **Strengths:** Ensures physically valid parameters (positive scales, unit quaternions, bounded opacities). The ZipSplat variant [S2, unofficial] allows for better optimization dynamics through specific biases and clamping.
- **Limitations:** The standard variant [S1, unofficial] may require careful initialization or regularization to prevent scale explosion or vanishing. The ZipSplat variant [S2, unofficial] requires additional hyperparameter tuning (clamping bounds, biases).
- **When to use:** Use when rendering scenes with explicit 3D Gaussian primitives. Use ZipSplat [S2, unofficial] when higher fidelity is needed with fewer Gaussians (e.g., G=32 per token). Use standard [S1, unofficial] for simpler setups or compatibility with existing Splatting pipelines.

## Background
Gaussian Parameter Activation is a core component of 3D Gaussian Splatting [S1, unofficial] and its variants like ZipSplat [S2, unofficial]. It was developed to bridge the gap between neural network outputs and the geometric requirements of Gaussian splats. The ZipSplat variant [S2, unofficial] was introduced to improve rendering quality with fewer Gaussians by refining the parameterization strategy.

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
