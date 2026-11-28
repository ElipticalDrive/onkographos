# ActiveNeRF Model Architecture

Also called: ActiveNeRF MLP structure, Uncertainty-aware NeRF network

## Overview
ActiveNeRF extends the standard NeRF architecture by predicting not only volume density and view-dependent color but also a prediction variance (uncertainty) for each sampled point [S1]. This modification allows the rendering pipeline to weigh contributions based on the model's confidence, facilitating active view selection strategies where new camera poses are chosen to minimize uncertainty in poorly observed regions [S1]. Unlike sibling architectures that modify input spaces or use explicit primitives, ActiveNeRF retains the coordinate-based implicit representation but augments the output space with a scalar uncertainty field [S1].

## How it works
The model is a multi-layer perceptron (MLP) that maps 3D spatial coordinates and viewing directions to density, color, and variance [S1].

**Inputs and Outputs**
*   **Inputs:** 3D position $\mathbf{x} = (x, y, z)$ and viewing direction $\mathbf{d} = (d_x, d_y, d_z)$.
*   **Outputs:** Volume density $\sigma \in \mathbb{R}$, RGB color $\mathbf{c} \in \mathbb{R}^3$, and prediction variance $\nu \in \mathbb{R}$ [S1].

**Data Structures and Layout**
The network consists of a shared feature encoder and specialized output heads [S1]:
1.  **Positional Encoding:** Separate encoding functions for coordinates and directions.
2.  **Feature Encoder:** An 8-layer fully connected (FC) MLP with a residual connection applied to the 4th layer [S1].
3.  **Density Head:** A single FC layer taking the encoder's output to predict $\sigma$ [S1].
4.  **Variance Head:** A single FC layer with a softplus activation function taking the encoder's output to predict $\nu$ [S1].
5.  **Color Head:** A single FC layer taking the concatenation of the latent feature vector and the encoded viewing direction to predict $\mathbf{c}$ [S1].

**Positional Encoding**
The positional encoding function $\gamma(p)$ maps a scalar input $p$ to a vector of sinusoidal frequencies [S1]:
$$
\gamma(p) = (\sin(2^0\pi p), \cos(2^0\pi p), \dots, \sin(2^{L-1}\pi p), \cos(2^{L-1}\pi p))
$$
where $L$ is the number of frequency bands [S1].

**Parameters**
*   $L=10$: Number of frequency bands for 3D coordinates $\mathbf{x}$ [S1].
*   $L=4$: Number of frequency bands for viewing directions $\mathbf{d}$ [S1].

**Algorithm Steps**
1.  Encode the 3D coordinates $\mathbf{x}$ using $\gamma$ with $L=10$.
2.  Encode the viewing direction $\mathbf{d}$ using $\gamma$ with $L=4$.
3.  Pass the encoded coordinates through the 8-layer MLP encoder. Let the output of the $i$-th layer be $h_i$. For $i=4$, apply the residual connection: $h_4 = \text{MLP}(h_3) + h_3$ [S1].
4.  Extract the latent feature vector $\mathbf{z}$ from the final layer of the encoder [S1].
5.  Compute volume density: $\sigma = \text{FC}_{\sigma}(\mathbf{z})$ [S1].
6.  Compute variance: $\nu = \text{softplus}(\text{FC}_{\nu}(\mathbf{z}))$ [S1].
7.  Concatenate the latent feature $\mathbf{z}$ with the encoded viewing direction $\gamma(\mathbf{d})$ to form $\mathbf{z}_{\text{concat}} = [\mathbf{z}, \gamma(\mathbf{d})]$ [S1].
8.  Compute color: $\mathbf{c} = \text{FC}_{\mathbf{c}}(\mathbf{z}_{\text{concat}})$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.nn.functional as F

class ActiveNeRFMLP(nn.Module):
    def __init__(self, feature_dim=256):
        super(ActiveNeRFMLP, self).__init__()
        # Feature encoder: 8 FC layers
        self.fc1 = nn.Linear(3 * 2 * 10, feature_dim) # 3 coords * 2 (sin/cos) * 10 bands
        self.fc2 = nn.Linear(feature_dim, feature_dim)
        self.fc3 = nn.Linear(feature_dim, feature_dim)
        self.fc4 = nn.Linear(feature_dim, feature_dim)
        self.fc5 = nn.Linear(feature_dim, feature_dim)
        self.fc6 = nn.Linear(feature_dim, feature_dim)
        self.fc7 = nn.Linear(feature_dim, feature_dim)
        self.fc8 = nn.Linear(feature_dim, feature_dim)
        
        # Output heads
        self.fc_sigma = nn.Linear(feature_dim, 1)
        self.fc_var = nn.Linear(feature_dim, 1)
        # Color head input: feature_dim + (3 * 2 * 4)
        self.fc_color = nn.Linear(feature_dim + 3 * 2 * 4, 3)

    def positional_encoding(self, x, L):
        """
        Applies positional encoding to input x.
        x: tensor of shape (..., 3)
        L: number of frequency bands
        Returns: tensor of shape (..., 3 * 2 * L)
        """
        # x shape: (..., 3)
        # freq_bands shape: (L,)
        freq_bands = torch.tensor([2**i for i in range(L)], dtype=x.dtype, device=x.device)
        # Expand x to (..., 3, 1) and freq_bands to (1, 1, L)
        # Resulting shape before view: (..., 3, L)
        scaled_x = x.unsqueeze(-1) * (math.pi * freq_bands)
        # Apply sin and cos
        pe = torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)
        # Reshape to (..., 3 * 2 * L)
        return pe.view(*x.shape[:-1], -1)

    def forward(self, x, d):
        """
        x: (N, 3) 3D coordinates
        d: (N, 3) viewing directions
        """
        # 1. Encode coordinates (L=10)
        x_enc = self.positional_encoding(x, L=10)
        
        # 2. Encode directions (L=4)
        d_enc = self.positional_encoding(d, L=4)
        
        # 3. Process through 8-layer MLP with residual on layer 4
        h = F.relu(self.fc1(x_enc))
        h = F.relu(self.fc2(h))
        h = F.relu(self.fc3(h))
        h_pre4 = F.relu(self.fc4(h))
        h = h_pre4 + h # Residual connection on 4th layer
        h = F.relu(self.fc5(h))
        h = F.relu(self.fc6(h))
        h = F.relu(self.fc7(h))
        h = F.relu(self.fc8(h))
        
        # 4. Predict density
        sigma = self.fc_sigma(h)
        
        # 5. Predict variance with softplus
        var_raw = self.fc_var(h)
        variance = F.softplus(var_raw)
        
        # 6. Predict color
        z_concat = torch.cat([h, d_enc], dim=-1)
        color = self.fc_color(z_concat)
        
        return sigma, color, variance
```

**Edge Cases and Pitfalls**
*   **Variance Positivity:** The variance output must be non-negative. The source specifies using a softplus activation function for the variance head to ensure this [S1].
*   **Residual Connection:** The residual connection is explicitly applied only to the 4th layer of the 8-layer encoder [S1].
*   **Encoding Separation:** Coordinates and directions are encoded separately with different numbers of bands ($L=10$ vs $L=4$) before being processed by their respective branches [S1].

## Subtopics

### Uncertainty and Reliability
General approaches to estimating uncertainty in NeRFs, including Bayesian methods and ensemble techniques. ActiveNeRF is a specific instance using a single network with a variance output. [Uncertainty and Reliability](14487-uncertainty-and-reliability.md)

### Active View Selection
Strategies for choosing new camera poses based on the predicted uncertainty. ActiveNeRF uses its variance output to guide this selection process [S1]. [Sparse View and Few-Shot Optimization](14428-sparse-view-and-few-shot-optimization.md)

## Operating characteristics
*   **Complexity:** The computational complexity is dominated by the 8-layer MLP for the feature encoder, which is shared between density, variance, and color predictions. The additional variance head adds negligible overhead compared to the standard NeRF [S1].
*   **Performance:** The model is trained on a single RTX 2080 Ti GPU with a batch size of 1024 rays [S1].
*   **Strengths:** Provides a per-point uncertainty estimate that can be used for active learning and view selection, potentially improving reconstruction quality in sparse-view scenarios [S1].
*   **Limitations:** The uncertainty is a prediction variance from a single forward pass, which may not capture all sources of epistemic uncertainty compared to ensemble methods [S1].

## Background
ActiveNeRF was introduced in the paper "ActiveNeRF: Learning Where to See with Uncertainty Estimation" [S1]. It builds upon the original NeRF architecture [20] by incorporating uncertainty estimation to facilitate active view synthesis [S1].

## Sources
[S1] ActiveNeRF: Learning Where to See with Uncertainty Estimation, https://link.springer.com/chapter/10.1007/978-3-031-19827-4_14, fetched 2026-10-28
