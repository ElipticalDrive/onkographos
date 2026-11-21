# HollowNeRF

Also called: HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation

## Overview
HollowNeRF is a compression method for Neural Radiance Fields (NeRF) that automatically sparsifies the feature grid during training by using a trainable 3D saliency mask to guide efficient feature pruning, redistributing hash collisions to improve rendering quality with fewer parameters [S1, S2]. It sets itself apart from standard hash-grid NeRFs like Instant-NGP by explicitly learning which spatial regions are necessary for rendering, thereby suppressing features in empty or invisible areas rather than relying on static priors or post-hoc pruning [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D coordinate $x = (x, y, z)$, ground truth images for loss computation.
*   **Outputs:** Density $\sigma$ and color $c$ for volume rendering.

**Data Structures:**
1.  **Multi-level Hash Grid ($H$):** The standard feature hash grid from Instant-NGP, storing dense feature vectors [S1].
2.  **Trainable 3D Saliency Grid ($G$):** A coarse 3D tensor of resolution $T \times T \times T$. It stores trainable scalar values representing the saliency of features near each grid voxel [S1]. $T$ is typically much smaller than the resolution of the dense feature grid [S1].
3.  **MLP:** A shallow neural network (fully connected layers) that decodes features into density and color [S1].

**Numbered Steps:**
1.  **Feature Fetch:** Given input coordinate $x$, fetch the corresponding feature vector $f$ from the multi-level hash grid $H$, following the Instant-NGP method [S1].
2.  **Saliency Prediction:**
    *   Determine the 8 voxels in the saliency grid $G$ surrounding $x$.
    *   Perform trilinear interpolation on the values of these 8 voxels to obtain a smoothed saliency distribution value $p_{raw}$ [S1].
    *   Apply a sigmoid function to $p_{raw}$ to predict the saliency weight $p$:
        $$ p = \sigma_{sigmoid}(p_{raw}) $$
        where $p \in [0, 1]$ [S1].
3.  **Feature Scaling:** Scale the fetched feature $f$ by the saliency weight $p$ to get the weighted feature $v$:
    $$ v = p \cdot f $$
    This suppresses features in regions where $p \approx 0$ (empty or invisible regions) [S1].
4.  **Zero-Skipping Gate:** Feed the weighted feature $v$ into the MLP. The MLP is designed or gated such that if $v = 0$, the output density $\sigma = 0$. This ensures that pruned regions do not contribute to the volume rendering integral [S1].
5.  **Rendering and Loss:** Use the predicted density $\sigma$ and color to perform volume rendering. Compare the rendered image to the ground truth to obtain a loss $L$ [S1].
6.  **ADMM Pruning:** During training, use an Alternating Direction Method of Multipliers (ADMM) pruner to sparsify the saliency value distribution. The ADMM pruner enforces a sparsity constraint on the saliency grid $G$ when optimizing the loss $L$, explicitly pruning saliency weights of empty/invisible regions to exactly 0 [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class HollowNeRF:
    def __init__(self, hash_grid, saliency_resolution_T, mlp):
        """
        hash_grid: Instant-NGP style hash grid structure
        saliency_resolution_T: int, resolution T of the 3D saliency grid G
        mlp: nn.Module, the decoder network
        """
        self.hash_grid = hash_grid
        self.T = saliency_resolution_T
        self.mlp = mlp
        
        # Initialize trainable 3D saliency grid G
        # Shape: (1, T, T, T) or similar tensor format
        self.G = nn.Parameter(torch.zeros(1, T, T, T))
        
        # ADMM variables for sparsity constraint
        # Z is the auxiliary variable for the L0 norm approximation or hard thresholding
        self.Z = nn.Parameter(torch.zeros(1, T, T, T))
        self.Y = nn.Parameter(torch.zeros(1, T, T, T)) # Lagrange multiplier
        self.lambda_param = 1.0 # Sparsity penalty weight

    def get_saliency_weight(self, x):
        """
        x: Tensor of shape (N, 3), coordinates
        Returns: Tensor of shape (N, 1), saliency weights p
        """
        # Normalize x to [0, 1] range if necessary, assuming G covers [0,1]^3
        # Scale x to grid coordinates [0, T-1]
        x_grid = x * (self.T - 1)
        
        # Trilinear interpolation of G at x_grid
        # G is (1, T, T, T). We need to sample at x_grid.
        # Using grid_sample or manual interpolation.
        # Assuming x_grid is normalized to [-1, 1] for grid_sample or similar.
        # Here we assume a helper for trilinear interpolation on tensor G.
        
        # Reshape for interpolation: (1, 1, T, T, T) -> (1, T, T, T)
        # Standard grid_sample expects (N, C, H, W) or (N, C, D, H, W)
        # Let's assume a standard 3D interpolation function exists or implement it.
        
        # Normalize to [-1, 1] for grid_sample
        x_norm = (x_grid / (self.T - 1)) * 2 - 1
        x_norm = x_norm.unsqueeze(0).unsqueeze(0).unsqueeze(0) # (1, 1, 1, N, 3)
        
        # G shape: (1, 1, T, T, T) for grid_sample (N, C, D, H, W)
        G_expanded = self.G.unsqueeze(0) # (1, 1, T, T, T)
        
        # Sample
        # grid_sample output shape: (1, 1, 1, N)
        p_raw_sampled = F.grid_sample(G_expanded, x_norm, mode='trilinear', align_corners=True)
        p_raw = p_raw_sampled.squeeze() # (N,)
        
        # Sigmoid
        p = torch.sigmoid(p_raw)
        return p.unsqueeze(-1) # (N, 1)

    def forward(self, x):
        """
        x: Tensor of shape (N, 3)
        Returns: density (N, 1), color (N, 3)
        """
        # 1. Fetch feature from hash grid
        f = self.hash_grid(x) # (N, D)
        
        # 2. Predict saliency weight
        p = self.get_saliency_weight(x) # (N, 1)
        
        # 3. Scale feature
        v = p * f # (N, D)
        
        # 4. Zero-skipping gate: if v is zero, force sigma to zero
        # The MLP itself might handle this if initialized correctly, 
        # or we explicitly mask.
        # Assuming MLP decodes v to (sigma, color)
        out = self.mlp(v)
        sigma = out[..., 0:1]
        color = out[..., 1:]
        
        # Explicit zero-skipping gate as described
        # If p is 0, v is 0. If v is 0, sigma should be 0.
        # We can enforce this by masking sigma where p is close to 0
        # However, the source says "ensure v=0 maps to sigma=0", 
        # which is often a property of the MLP initialization or activation.
        # We implement the gate explicitly for safety.
        mask = (p > 1e-6).float()
        sigma = sigma * mask
        
        return sigma, color

    def admm_step(self, lr_admm=0.01):
        """
        Perform one step of ADMM to sparsify G.
        Minimize ||G - Z||^2 + lambda * ||Z||_0 (approximated)
        Subject to G = Z.
        """
        # Update Z: Soft thresholding or hard thresholding for sparsity
        # Z = shrink(G + Y, lambda)
        # For L0, hard thresholding is used: Z_ij = G_ij + Y_ij if |G_ij + Y_ij| > lambda, else 0
        # Or simpler: Z = 0 if |G| < threshold, else G.
        # The source mentions "sparsify saliency value distribution... enforcing sparsity constraint".
        # Standard ADMM for L0:
        
        # 1. Update Z
        Z_update = self.G + self.Y
        # Hard thresholding for sparsity
        threshold = self.lambda_param
        self.Z.data = torch.where(torch.abs(Z_update) > threshold, Z_update, torch.zeros_like(Z_update))
        
        # 2. Update G (minimize ||G - Z + Y||^2) -> G = Z - Y
        self.G.data = self.Z - self.Y
        
        # 3. Update Y (dual ascent)
        self.Y.data = self.Y + (self.G - self.Z)

    def train_step(self, x, y_gt, optimizer, admm_optimizer):
        sigma, color = self.forward(x)
        # Volume rendering logic would go here to produce image
        # loss = L(image, y_gt)
        # Placeholder for loss calculation
        loss = 0.0 
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # ADMM step to prune saliency grid
        admm_optimizer.step(self.admm_step)
```

**Parameters:**
*   $T$: Resolution of the trainable 3D saliency grid $G$. Typically much smaller than the resolution of the dense feature grid [S1].
*   $\lambda$: Sparsity penalty weight in the ADMM pruner. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Reflective Surfaces:** HollowNeRF encounters difficulties in handling objects with reflective surfaces, which are rendered inadequately [S2].
*   **Precision:** The zero-skipping gate relies on $p$ reaching exactly 0 or sufficiently close to 0 to ensure $\sigma=0$. The ADMM pruner is used to explicitly drive these weights to 0 [S1].

## Operating characteristics
*   **Parameter Count:** HollowNeRF uses 31% of the parameters of Instant-NGP while delivering comparable rendering quality [S1].
*   **Accuracy:** It can achieve up to 1dB PSNR accuracy gain using only 56% of the parameters compared to Instant-NGP [S1].
*   **Strengths:** Automatically sparsifies the feature grid during training without requiring prior knowledge of object shape. Reduces parameter count significantly while maintaining or improving rendering quality [S1, S2].
*   **Limitations:** Struggles with reflective surfaces [S2]. The ADMM pruning adds complexity to the training loop [S1].

## Background
HollowNeRF was proposed in 2023 as a novel compression solution for hashgrid-based NeRFs [S1]. It addresses the issue that existing solutions rely on prior knowledge of object shape or costly repeated model evaluations to estimate object shape during training [S1]. The method leverages the sparsity in 3D scenes to redistribute hash collisions, improving the cost-accuracy tradeoff [S1].

## Sources
[S1] HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation, https://doi.org/10.1109/iccv51070.2023.00322, fetched 2026-10-26
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
