# Tiny MLP for Density and Color Estimation
Also called: Rip-NeRF MLP
## Overview
The Tiny MLP is a lightweight neural network component in Rip-NeRF that estimates the density $\tau$ and view-dependent color $c$ of a sampled sphere $S$ from Ripmap-encoded features and view direction. It differs from its parent [InstantNGP Network Architecture](5049-instantngp-network-architecture.md) by using a fixed, shallow architecture (5 layers) optimized for half-precision fused operations, and from its sibling [HyperINR](10461-hyperinr.md) by relying on explicit geometric feature extraction rather than hypernetworks. It aligns with Tri-MipRF [Hu et al. 2023] for consistency in handling Ripmap inputs.

## How it works
**Inputs and Outputs:**
- Input feature vector $f$: 160-dimensional vector from Ripmap Encoding.
- Input view direction $d$: 3D unit vector.
- Output density $\tau$: Scalar.
- Output color $c$: 3D vector (RGB).

**Data Structures:**
- MLP weights: Stored in half-precision (float16) for fused operations via tiny-cuda-nn [G].
- Spherical Harmonics (SH) coefficients: Precomputed for view direction encoding.

**Steps:**
1. **Geometric Feature Extraction:** Pass $f$ through the first two MLP layers to produce a 15-dimensional geometric feature vector $f_{geo}$ and the density $\tau$.
2. **View Direction Encoding:** Encode the view direction $d$ using spherical harmonics to produce a view-dependent feature vector $d_{sh}$.
3. **Color Estimation:** Concatenate $f_{geo}$ and $d_{sh}$, then pass through the final three MLP layers to estimate the color $c$.

**Formulas:**
- Density output: $\tau = \text{TruncExp}(W_2 \cdot \text{ReLU}(W_1 \cdot f + b_1) + b_2)$, where $\text{TruncExp}(x) = \min(\max(x, \epsilon), \text{max\_val})$ [S1].
- Color output: $c = \text{MLP}_{final}([f_{geo}; d_{sh}])$ [S1].

**Parameters:**
- MLP width: 128 neurons per hidden layer [S1].
- Activation: ReLU for hidden layers; truncated exponential for density output [S1].
- Feature dimension: 160 [S1].
- Geometric feature dimension: 15 [S1].

**Edge Cases and Pitfalls:**
- Precision: Use half-precision (float16) for all operations to match tiny-cuda-nn optimizations [S1].
- Density Clamping: Ensure density $\tau$ is clamped to avoid numerical instability in volume rendering integration [G].
- View Direction Normalization: Ensure $d$ is normalized before SH encoding [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class TinyMLP(nn.Module):
    def __init__(self, feature_dim=160, width=128, sh_degree=3):
        super().__init__()
        # Layer 1: 160 -> 128
        self.fc1 = nn.Linear(feature_dim, width)
        # Layer 2: 128 -> 128 (produces f_geo and intermediate for tau)
        self.fc2 = nn.Linear(width, width)
        # Layer 3: 128 -> 15 (f_geo) + 128 (intermediate for tau)
        self.fc3 = nn.Linear(width, 15 + width)
        # Layer 4: 15 + sh_dim -> 128
        sh_dim = (sh_degree + 1) ** 2 * 3
        self.fc4 = nn.Linear(15 + sh_dim, width)
        # Layer 5: 128 -> 3 (color)
        self.fc5 = nn.Linear(width, 3)
        
        # Density head from fc3's second part
        self.fc_tau = nn.Linear(width, 1)
        
        self.relu = nn.ReLU()
        self.sh_degree = sh_degree

    def forward(self, f, d):
        # Step 1: Geometric feature extraction
        x = self.relu(self.fc1(f))
        x = self.relu(self.fc2(x))
        x = self.fc3(x)
        
        f_geo = x[:, :15]
        x_tau = x[:, 15:]
        
        # Density output with truncated exponential
        tau = self.fc_tau(x_tau)
        tau = torch.clamp(tau, min=1e-6, max=1e6) # Truncated Exp approximation [G]
        
        # Step 2: View direction encoding
        d_sh = encode_spherical_harmonics(d, self.sh_degree)
        
        # Step 3: Color estimation
        concat = torch.cat([f_geo, d_sh], dim=-1)
        x_color = self.relu(self.fc4(concat))
        c = self.fc5(x_color)
        
        return tau, c

def encode_spherical_harmonics(d, degree):
    # Standard SH encoding for degree 3 (16 coefficients per channel, 3 channels)
    # Simplified for brevity; actual implementation uses precomputed SH basis
    x, y, z = d[:, 0:1], d[:, 1:2], d[:, 2:3]
    # ... SH basis computation ...
    return sh_coeffs
```

## Subtopics

### [Cross-Branch Interactions](11904-cross-branch-interactions.md)
Uses multi-head attention to model interactions between colors, densities, and neighboring points. Choose when view-independent assumptions fail.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as MLP depth is fixed (5 layers) and width is small (128).
- **Memory Complexity:** $O(1)$, with parameters $\approx 160 \times 128 + 128^2 + 128 \times 15 + 128 \times 128 + 128 \times 3 \approx 100$ KB.
- **Performance:** Optimized for half-precision fused operations via tiny-cuda-nn, enabling real-time rendering on modern GPUs [S1].
- **Strengths:** Low latency, minimal memory footprint, suitable for dynamic scenes.
- **Limitations:** Limited capacity for complex view-dependent effects compared to larger MLPs.
- **When to Use:** Real-time neural rendering with limited computational resources.
- **When Not to Use:** High-fidelity offline rendering requiring complex material models.

## Background
Originated in Rip-NeRF [S1] (2024) as a lightweight alternative to larger MLPs in NeRF variants. It builds on [InstantNGP Network Architecture](5049-instantngp-network-architecture.md) by simplifying the architecture for Ripmap-encoded features. It aligns with Tri-MipRF [Hu et al. 2023] for consistency in handling multi-resolution features.

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
