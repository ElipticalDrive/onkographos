# Visibility and Outgoing Radiance MLP


## Overview
This method replaces costly recursive path tracing for indirect illumination by learning continuous functions for visibility and outgoing radiance, enabling efficient view synthesis and inverse rendering [S1]. It differs from its parent [General Radiance Field Rendering via MLP](8683-general-radiance-field-rendering-via-mlp.md) by explicitly decoupling the visibility term and the outgoing radiance field into separate learnable MLPs, avoiding the need for expensive sphere tracing during forward rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Observed images, geometry (surface points $\mathbf{x}$, normals $\mathbf{\hat{n}}$), and viewing/lighting directions ($\boldsymbol{\omega}_i, \boldsymbol{\omega}_o$).
*   **Outputs:** Visibility value $v$ and outgoing radiance $L_o$.

**Data Structures:**
The method utilizes two primary MLPs:
1.  **Visibility MLP ($V$):** Maps surface point location $\mathbf{x}$ and direction $\boldsymbol{\omega}_i$ to visibility $v$.
2.  **Outgoing Radiance Field MLP ($R$):** Maps surface point location $\mathbf{\hat{x}}$, normal $\mathbf{\hat{n}}$, and viewing direction $\boldsymbol{\omega}_o$ to outgoing radiance $L_o$.

Both networks share the same architecture: 4 layers with 512 hidden units [S1].

**Positional Encoding:**
Input coordinates are encoded using positional encoding :
*   3D locations: 10 frequency components [S1].
*   Directions: 4 frequency components [S1] [G: Standard positional encoding $\gamma(p) = [\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)]$].

**Training Procedure:**
1.  **Visibility MLP Training:** Train $V$ using a small number of sampled rays above surface points to learn occlusion [S1].
2.  **Outgoing Radiance Field Training:** Train $R$ using a view synthesis method [27] from observed images and geometry [S1].
3.  **Rendering:** Query $R$ to obtain the outgoing radiance of the second intersection (cumulative result of multiple bounces):
    $$ L_o(\mathbf{\hat{x}'}, -\boldsymbol{\omega}_i) = R(\mathbf{\hat{x}'}, \mathbf{\hat{n}'}, -\boldsymbol{\omega}_i) \quad (6) $$
    [S1]

**Parameters:**
*   **Optimizer:** Adam [S1] .
*   **Learning Rate:** $5 \times 10^{-4}$ [S1].
*   **Framework:** PyTorch [S1].
*   **Loss Function:** The full loss in the final stage is:
    $$ \ell = \lambda_{\text{recon}}\ell_{\text{recon}} + \lambda_{\text{KL}}\ell_{\text{KL}} + \lambda_{\text{s}}\ell_{\text{s}} \quad (12) $$
    [S1]
    *   $\lambda_{\text{recon}} = 1.0$
    *   $\lambda_{\text{KL}} = 0.01$
    *   $\lambda_{\text{s}} = 0.1$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class PositionalEncoding(nn.Module):
    def __init__(self, num_freq_components):
        super().__init__()
        self.num_freq_components = num_freq_components

    def forward(self, x):
        # x shape: (..., D)
        # Output shape: (..., D * 2 * num_freq_components)
        freq_bands = 2.0 ** torch.arange(self.num_freq_components, device=x.device).float()
        x_expanded = x.unsqueeze(-1) * freq_bands.unsqueeze(0) # (..., D, num_freq)
        x_sin = torch.sin(x_expanded)
        x_cos = torch.cos(x_expanded)
        return torch.cat([x_sin, x_cos], dim=-1).view(x.shape[0], -1)

class VisibilityMLP(nn.Module):
    def __init__(self, loc_freq=10, dir_freq=4):
        super().__init__()
        self.loc_enc = PositionalEncoding(loc_freq)
        self.dir_enc = PositionalEncoding(dir_freq)
        # Input dim: 3*2*10 + 3*2*4 = 60 + 24 = 84
        input_dim = 3 * 2 * loc_freq + 3 * 2 * dir_freq
        hidden_dim = 512
        self.layers = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 1) # Output v
        )

    def forward(self, x, omega_i):
        # x: (N, 3), omega_i: (N, 3)
        x_enc = self.loc_enc(x)
        omega_i_enc = self.dir_enc(omega_i)
        combined = torch.cat([x_enc, omega_i_enc], dim=-1)
        return self.layers(combined).squeeze(-1)

class OutgoingRadianceMLP(nn.Module):
    def __init__(self, loc_freq=10, dir_freq=4):
        super().__init__()
        self.loc_enc = PositionalEncoding(loc_freq)
        self.norm_enc = PositionalEncoding(dir_freq) # For normal
        self.dir_enc = PositionalEncoding(dir_freq)   # For viewing direction
        # Input dim: 3*2*10 + 3*2*4 + 3*2*4 = 60 + 24 + 24 = 108
        input_dim = 3 * 2 * loc_freq + 3 * 2 * dir_freq + 3 * 2 * dir_freq
        hidden_dim = 512
        self.layers = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3) # Output L_o (RGB)
        )

    def forward(self, x_hat, n_hat, omega_o):
        # x_hat: (N, 3), n_hat: (N, 3), omega_o: (N, 3)
        x_enc = self.loc_enc(x_hat)
        n_enc = self.norm_enc(n_hat)
        omega_o_enc = self.dir_enc(omega_o)
        combined = torch.cat([x_enc, n_enc, omega_o_enc], dim=-1)
        return self.layers(combined)

# Training loop example
def train_visibility_model():
    model = VisibilityMLP()
    optimizer = optim.Adam(model.parameters(), lr=5e-4)
    # 1. Train using a small number of sampled rays above surface points [S1]
    # 2. Minimize visibility prediction error against ground truth occlusion
    pass

def train_radiance_model():
    model = OutgoingRadianceMLP()
    optimizer = optim.Adam(model.parameters(), lr=5e-4)
    # 1. Train using view synthesis method [27] from observed images and geometry [S1]
    # 2. Minimize loss: l = 1.0 * l_recon + 0.01 * l_KL + 0.1 * l_s [S1]
    pass
```

## Subtopics

### Visibility Parameterization
Re-parameterizes visibility as an MLP to avoid time-consuming sphere tracing during forward rendering [S1]. It is trained using a small number of sampled rays above surface points [S1].

## Operating characteristics
*   **Time Complexity:** Training takes approximately 1 hour for the second stage and 2 hours for the third stage on a single RTX 3090 GPU [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides a compact and continuous representation of visibility, significantly reducing rendering time compared to recursive path tracing [S1].
*   **Limitations:** Requires pre-trained geometry and observed images for training [S1].
*   **When to use:** When efficient view synthesis and inverse rendering are required, and recursive path tracing is too slow [S1].
*   **When not to use:** When high-fidelity dynamic indirect illumination without pre-training is needed [G].

## Background
The method originates from "Modeling Indirect Illumination for Inverse Rendering" (CVPR 2022) [S1]. It addresses the computational bottleneck of calculating indirect illumination via recursive path tracing by learning it as a continuous function [S1].

## Sources
[S1] Modeling Indirect Illumination for Inverse Rendering, https://doi.org/10.1109/cvpr52688.2022.01809, fetched 2026-10-26
[S2] Fourier Features Let Networks Learn High Frequency Functions in Low Dimensional Domains, https://arxiv.org/abs/2006.10739, fetched 2026-10-26
[S10] Adam: A Method for Stochastic Optimization, https://arxiv.org/abs/1412.6980, fetched 2026-10-26
