# Forward LBS Network

Also called: Neural Linear Blend Skinning, Forward Skin Network

## Overview
The Forward LBS Network is a method for deforming canonical 3D Gaussians into posed space by predicting learnable skinning weights for any query point, replacing fixed geometric weights with a neural affinity model [S1]. It differs from backward LBS variants by operating directly on canonical points without requiring global canonicalization relative to the SMPL frame, and it generally achieves higher reconstruction quality (PSNR) than deterministic SMPL weight baselines [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ query points $\hat{\mathbf{X}} = \{\hat{\mathbf{x}}^{(i)}\}_{i=1}^N$ in canonical space, where $\hat{\mathbf{x}}^{(i)} \in \mathbb{R}^3$.
*   **Input:** A set of 24 rigid bone transformation matrices $\{\mathbf{B}_b\}_{b=1}^{24}$, where each $\mathbf{B}_b$ is a $4 \times 4$ rotation-translation matrix derived from the SMPL model [S1].
*   **Output:** A set of deformed points $\bar{\mathbf{X}} = \{\bar{\mathbf{x}}^{(i)}\}_{i=1}^N$ in posed space.

**Data Structures**
*   **Network Parameters:** $\boldsymbol{\sigma}_{\omega}$ representing the weights of the forward LBS network $f_{\boldsymbol{\sigma}_{\omega}}$.
*   **Bone Transformations:** $\mathbf{B}_b \in \mathbb{R}^{4 \times 4}$ for $b=1 \dots 24$. The set includes 23 local transformations and one global transformation [S1].

**Algorithm**
1.  **Network Definition:** The network $f_{\boldsymbol{\sigma}_{\omega}}: \mathbb{R}^3 \rightarrow [0, 1]^{24}$ predicts a skinning weight vector for any canonical point. It consists of 4 hidden layers, each with 128 channels [S1].
2.  **Activation and Normalization:** The network uses weight normalization [68] and Softplus activation with $\beta = 100$ [S1].
3.  **Weight Prediction:** For each query point $\hat{\mathbf{x}}^{(i)}$, the network outputs a weight vector $\mathbf{w}^{(i)} = f_{\boldsymbol{\sigma}_{\omega}}(\hat{\mathbf{x}}^{(i)})$. The weights are constrained such that $\mathbf{w}^{(i)} \in [0, 1]^{24}$ and $\sum_{b=1}^{24} w_b^{(i)} = 1$ [S1].
4.  **Deformation:** The deformed point $\bar{\mathbf{x}}^{(i)}$ is computed using the linear blend skinning formula:
    $$
    \bar{\mathbf{x}}^{(i)} = \left(\sum_{b=1}^{24} f_{\boldsymbol{\sigma}_{\omega}}(\hat{\mathbf{x}}^{(i)})_b \mathbf{B}_b\right) \hat{\mathbf{x}}^{(i)}
    $$
    where $\hat{\mathbf{x}}^{(i)}$ is treated as a homogeneous coordinate for the matrix multiplication [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ForwardLBSNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # 4 hidden layers, 128 channels, weight normalization, Softplus(beta=100)
        # Input: 3 (canonical x, y, z)
        # Output: 24 (bone weights)
        self.layers = nn.ModuleList([
            nn.utils.weight_norm(nn.Linear(3, 128)),
            nn.utils.weight_norm(nn.Linear(128, 128)),
            nn.utils.weight_norm(nn.Linear(128, 128)),
            nn.utils.weight_norm(nn.Linear(128, 128))
        ])
        self.output_layer = nn.Linear(128, 24)
        self.activation = nn.Softplus(beta=100)

    def forward(self, canonical_points):
        """
        canonical_points: Tensor of shape (N, 3)
        Returns: Tensor of shape (N, 24) representing skinning weights
        """
        x = canonical_points
        for layer in self.layers:
            x = self.activation(layer(x))
        weights = self.output_layer(x)
        # Ensure weights sum to 1 and are in [0, 1]
        # Note: Softplus ensures positive values; softmax or normalization ensures sum=1
        # The source specifies Softplus activation and sum=1 constraint.
        # Implementation detail: Apply softmax to ensure sum=1 and valid probabilities
        weights = torch.softmax(weights, dim=-1)
        return weights

def deform_point(canonical_point, bone_transforms, weights):
    """
    canonical_point: Tensor of shape (4,) homogeneous coordinate [x, y, z, 1]
    bone_transforms: Tensor of shape (24, 4, 4)
    weights: Tensor of shape (24,)
    Returns: Deformed point in posed space
    """
    # Weighted sum of bone transformations
    # W = sum(w_b * B_b)
    W = torch.zeros(4, 4, device=canonical_point.device)
    for b in range(24):
        W += weights[b] * bone_transforms[b]
    
    # Apply transformation
    deformed_point = torch.matmul(W, canonical_point)
    return deformed_point

def forward_lbs(canonical_points, bone_transforms):
    """
    canonical_points: Tensor of shape (N, 3)
    bone_transforms: Tensor of shape (24, 4, 4)
    Returns: Deformed points of shape (N, 3)
    """
    network = ForwardLBSNetwork()
    # weights: (N, 24)
    weights = network(canonical_points)
    
    deformed_points = []
    for i in range(canonical_points.shape[0]):
        # Convert to homogeneous
        pt_homo = torch.cat([canonical_points[i], torch.tensor([1.0])])
        deformed_pt = deform_point(pt_homo, bone_transforms, weights[i])
        deformed_points.append(deformed_pt[:3])
    
    return torch.stack(deformed_points)
```

**Initialization**
The network is initialized by meta-learning on skinning weights of canonical meshes from the CAPE dataset [S1].
*   **Meta-learning Algorithm:** Reptile [S1] with 24 inner steps.
*   **Inner Learning Rate:** $1 \times 10^{-4}$.
*   **Outer Learning Rate:** $1 \times 10^{-5}$.
*   **Optimizer:** Adam [S1].
*   **Batch Size:** 4.
*   **Outer Loop Steps:** 100,000 [S1].
*   The resulting model is used as initialization for per-subject optimization on the ZJU-MoCap dataset [S1].

## Subtopics

### Neural Linear Blend Skinning
A neural network $f_{\boldsymbol{\sigma}_{\omega}}(\hat{\mathbf{x}}): \mathbb{R}^3 \rightarrow [0, 1]^{24}$ that predicts skinning weights for any point in space, replacing fixed weights with learned affinities to 24 bone transformations [S1]. It uses 23 local transformations and one global transformation based on the SMPL model [S1]. The weight vector $\mathbf{w}^{(i)}$ must sum to 1 ($\sum_{b=1}^{24} w_b^{(i)} = 1$) [S1].

## Operating characteristics
*   **Performance:** Learned forward LBS achieves the highest Peak Signal-to-Noise Ratio (PSNR) among the compared variants, followed by deterministic SMPL weights, with backward LBS yielding the lowest PSNR [S1].
*   **Complexity:** The method involves a forward pass through a small MLP (4 layers, 128 channels) per point, followed by a weighted sum of 24 $4 \times 4$ matrices. This is computationally efficient for real-time or differentiable rendering pipelines.
*   **Limitations:** The method relies on the quality of the meta-initialized weights and may require per-subject optimization for high-fidelity results on specific subjects [S1].

## Background
The Forward LBS Network is part of the ARAH (Animatable Volume Rendering of Articulated Human SDFs) framework [S1]. It builds on the concept of neural linear blend skinning introduced in prior works [12, 48, 67, 78] as cited in [S1]. The approach was developed to enable high-quality animatable human reconstruction from monocular videos by learning spatially varying skinning weights [S1].

## Sources
[S1] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
