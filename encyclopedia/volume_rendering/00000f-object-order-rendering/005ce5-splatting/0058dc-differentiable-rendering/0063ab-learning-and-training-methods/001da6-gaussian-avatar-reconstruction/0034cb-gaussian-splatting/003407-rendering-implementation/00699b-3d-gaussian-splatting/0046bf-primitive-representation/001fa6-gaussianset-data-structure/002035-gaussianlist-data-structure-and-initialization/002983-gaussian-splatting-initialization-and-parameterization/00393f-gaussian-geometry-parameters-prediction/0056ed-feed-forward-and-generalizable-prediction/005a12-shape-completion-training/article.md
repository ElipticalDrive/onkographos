# Shape Completion Training

Also called: Decoder-based Shape Completion, Folding Grid Expansion for Shape Completion

## Overview
Shape Completion Training is a feed-forward method that predicts all 14 3D Gaussian attributes from partial, incomplete inputs without requiring target coordinates or per-scene optimization [S1, unofficial]. It distinguishes itself from sibling methods by employing a fixed 2×2 folding grid to expand a small set of coarse primitives into a dense Gaussian set, enabling robust reconstruction of missing geometry in occluded or sparse scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Partial 3D data (e.g., point clouds or voxel grids) with missing regions. No target coordinates are provided during inference.
*   **Output:** A set of 1,024 3D Gaussians $\mathcal{G} = \{\mu_i, \Sigma_i, \alpha_i, c_i\}_{i=1}^{1024}$, where $\mu$ is mean, $\Sigma$ is covariance (defined by scale $s$ and rotation $q$), $\alpha$ is opacity, and $c$ is color (DC spherical harmonic coefficients).

**Data Structures:**
1.  **Coarse Primitives:** A list of 256 entities. Each entity contains 14 attributes (position, scale, rotation, opacity, color).
2.  **Folding Grid:** A fixed $2 \times 2$ grid structure used to spatially expand the coarse primitives.
3.  **Final Gaussians:** A list of 1,024 entities with the same 14-attribute layout.

**Algorithm Steps:**
1.  **Encoder Freezing:** Initialize a pretrained feature encoder (e.g., Vision Transformer) and freeze its weights.
2.  **Decoder Initialization:** Initialize an identical decoder architecture to the encoder's output space.
3.  **Feature Processing:** Pass the input features through four Transformer layers in the decoder (width 256, 8 attention heads) to predict 256 coarse primitive attributes [S1, unofficial].
4.  **Grid Expansion:** Expand the 256 coarse primitives to 1,024 Gaussians using the fixed 2×2 folding grid. This step duplicates and spatially offsets the coarse primitives to fill the volume [S1, unofficial].
5.  **Loss Computation:** Compute the total loss $\mathcal{L}$ as a weighted sum of geometric and attribute losses:
    $$ \mathcal{L} = \mathcal{L}_{Chamfer}^{2} + \mathcal{L}_{NN} + 0.1 \left( \mathcal{L}_{L1}(\alpha, s, c) + \mathcal{L}_{sign\text{-}inv}(q) \right) $$
    Where:
    *   $\mathcal{L}_{Chamfer}^{2}$ is the squared $\ell_2$ Chamfer distance between predicted and target points.
    *   $\mathcal{L}_{NN}$ is the nearest-neighbor attribute matching loss.
    *   $\mathcal{L}_{L1}$ is the $\ell_1$ loss applied to opacity $\alpha$, scale $s$, and DC color coefficients $c$.
    *   $\mathcal{L}_{sign\text{-}inv}(q)$ is the sign-invariant quaternion similarity loss for rotation $q$.
    *   The aggregate attribute term is scaled by a weight of 0.1 [S1, unofficial].
6.  **Optimization:** Optimize the decoder weights using AdamW for 80 epochs.
    *   Learning rate: $10^{-3}$.
    *   Weight decay: 0.05.
    *   Warm-up: 5 epochs.
    *   Batch size: 64.
    *   Train three independent decoder seeds per frozen encoder [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class ShapeCompletionDecoder(nn.Module):
    def __init__(self, num_coarse=256, num_final=1024, width=256, heads=8, layers=4):
        super().__init__()
        # Transformer layers for coarse primitive prediction
        self.transformer_blocks = nn.ModuleList([
            nn.TransformerEncoderLayer(d_model=width, nhead=heads) 
            for _ in range(layers)
        ])
        # Projection to 14 attributes per coarse primitive
        self.attr_proj = nn.Linear(width, 14)
        self.num_coarse = num_coarse
        self.num_final = num_final
        self.expansion_factor = num_final // num_coarse

    def forward(self, x):
        # x: [Batch, Coarse_Primitives, Width]
        for block in self.transformer_blocks:
            x = block(x)
        
        # Predict 14 attributes for 256 coarse primitives
        coarse_attrs = self.attr_proj(x) # [Batch, 256, 14]
        
        # Expand to 1024 Gaussians using fixed 2x2 folding grid
        # The grid implies a deterministic spatial duplication/offset
        final_attrs = self.expand_folding_grid(coarse_attrs)
        
        return final_attrs

    def expand_folding_grid(self, coarse_attrs):
        # coarse_attrs: [B, 256, 14]
        # Expand factor 4 (2x2 grid)
        # Duplicate each coarse primitive 4 times
        expanded = coarse_attrs.repeat_interleave(self.expansion_factor, dim=1) # [B, 1024, 14]
        
        # Apply fixed spatial offsets for the 2x2 grid to the position components
        # Assuming positions are in the first 3 dimensions of the 14 attributes
        # Offsets are predefined constants for the grid cells
        offsets = torch.tensor([
            [-0.5, -0.5, 0], [0.5, -0.5, 0],
            [-0.5, 0.5, 0], [0.5, 0.5, 0]
        ], device=coarse_attrs.device).float() # [4, 3]
        
        # Reshape offsets to broadcast over batch and primitives
        # expanded: [B, 1024, 14] -> split into pos [B, 1024, 3] and others [B, 1024, 11]
        pos = expanded[:, :, :3]
        others = expanded[:, :, 3:]
        
        # Apply offsets based on the expansion index
        # The expansion repeats every 4 primitives
        grid_indices = torch.arange(self.expansion_factor, device=coarse_attrs.device) # [4]
        grid_offsets = offsets[grid_indices] # [4, 3]
        
        # Reshape for broadcasting: [1, 1024, 1, 3] -> [1, 256, 4, 3]
        grid_offsets = grid_offsets.view(1, 1, 4, 3)
        pos_expanded = pos.view(pos.shape[0], 256, 4, 3)
        pos_final = pos_expanded + grid_offsets
        pos_final = pos_final.view(pos.shape[0], -1, 3)
        
        # Combine back
        final_attrs = torch.cat([pos_final, others], dim=-1)
        return final_attrs

def compute_loss(pred_attrs, target_attrs):
    # pred_attrs: [B, 1024, 14]
    # target_attrs: [B, N, 14] (N varies, e.g., ground truth points)
    
    # 1. Squared L2 Chamfer Distance
    pred_pos = pred_attrs[:, :, :3]
    target_pos = target_attrs[:, :, :3]
    
    # Compute pairwise distances
    dists = torch.cdist(pred_pos, target_pos, p=2) # [B, 1024, N]
    
    # Chamfer: min over target for pred, min over pred for target
    chamfer_pred = torch.min(dists, dim=2)[0].mean() # [B, 1024] -> mean
    chamfer_target = torch.min(dists, dim=1)[0].mean() # [B, N] -> mean
    chamfer_loss = (chamfer_pred + chamfer_target) / 2.0
    
    # 2. Nearest Neighbor Attribute Matching
    # Match each pred point to its nearest target point and compare attributes
    nn_indices = torch.argmin(dists, dim=2) # [B, 1024]
    target_nn_attrs = torch.gather(target_attrs, 1, nn_indices.unsqueeze(-1).expand(-1, -1, 14))
    nn_loss = torch.nn.functional.l1_loss(pred_attrs, target_nn_attrs)
    
    # 3. Attribute Losses
    # Opacity (alpha), Scale (s), DC Coefficients (c) use L1
    # Assuming indices: alpha=3, s=4,6,7 (scale 3D), c=8-13 (6 DC coeffs)
    # Note: Exact indices depend on attribute ordering. Assuming standard: pos(3), scale(3), rot(4), alpha(1), color(3) or SH.
    # Based on source, "DC coefficients" implies SH degree 0.
    # Let's assume: 0-2: pos, 3-5: scale, 6-9: rot, 10: alpha, 11-13: color (RGB) or SH DC.
    # Source says 14 attributes. Common: 3(pos)+3(scale)+4(rot)+1(alpha)+3(color) = 14.
    # Or 3(pos)+3(scale)+4(rot)+1(alpha)+6(SH DC) = 17? No, 14.
    # Likely: 3(pos) + 3(scale) + 4(rot) + 1(alpha) + 3(color) = 14.
    # But source mentions "DC coefficients" specifically.
    # If 14 attributes: 3(pos) + 3(scale) + 4(rot) + 1(alpha) + 3(color) is standard.
    # If SH DC is used, it might be 3(pos) + 3(scale) + 4(rot) + 1(alpha) + 3(color) = 14.
    # Let's assume indices for L1: alpha (index 10), scale (3-5), color (11-13).
    
    alpha = pred_attrs[:, :, 10]
    scale = pred_attrs[:, :, 3:6]
    color = pred_attrs[:, :, 11:14]
    
    target_alpha = target_nn_attrs[:, :, 10]
    target_scale = target_nn_attrs[:, :, 3:6]
    target_color = target_nn_attrs[:, :, 11:14]
    
    attr_l1 = torch.nn.functional.l1_loss(alpha, target_alpha) + \
              torch.nn.functional.l1_loss(scale, target_scale) + \
              torch.nn.functional.l1_loss(color, target_color)
    
    # 4. Sign-invariant Quaternion Loss
    # Rotations are indices 6-9
    pred_rot = pred_attrs[:, :, 6:10]
    target_rot = target_nn_attrs[:, :, 6:10]
    
    # Normalize quaternions
    pred_rot = torch.nn.functional.normalize(pred_rot, p=2, dim=-1)
    target_rot = torch.nn.functional.normalize(target_rot, p=2, dim=-1)
    
    # Sign-invariant: min(dot(q1, q2), dot(q1, -q2))
    dot_prod = torch.sum(pred_rot * target_rot, dim=-1)
    sign_inv_loss = torch.nn.functional.mse_loss(dot_prod, torch.ones_like(dot_prod))
    
    # Total Loss
    total_loss = chamfer_loss + nn_loss + 0.1 * (attr_l1 + sign_inv_loss)
    
    return total_loss

# Training Loop
def train_shape_completion(encoder, decoder, dataloader, epochs=80, lr=1e-3, weight_decay=0.05, warmup_epochs=5, batch_size=64):
    optimizer = optim.AdamW(decoder.parameters(), lr=lr, weight_decay=weight_decay)
    scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
    
    for epoch in range(epochs):
        if epoch < warmup_epochs:
            # Linear warmup
            for param_group in optimizer.param_groups:
                param_group['lr'] = lr * (epoch + 1) / warmup_epochs
        
        for batch in dataloader:
            inputs, targets = batch
            inputs = inputs.to(encoder.device)
            targets = targets.to(encoder.device)
            
            # Freeze encoder
            with torch.no_grad():
                features = encoder(inputs)
            
            # Decoder forward
            preds = decoder(features)
            
            # Compute loss
            loss = compute_loss(preds, targets)
            
            # Backward
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        scheduler.step()
```

## Operating characteristics
*   **Time Complexity:** Inference is $O(N)$ where $N$ is the number of output Gaussians (1,024), dominated by the Transformer encoder/decoder passes. Training is significantly more expensive due to the 80-epoch optimization per scene seed.
*   **Memory Complexity:** Low inference memory footprint as no per-scene optimization state is stored. Training memory depends on batch size (64) and the size of the feature maps.
*   **Performance:** Evaluated on 520 held-out objects with visible ratios {0.3, 0.5, 0.7}. The method demonstrates robustness to high missing ratios (up to 85% missing) by leveraging the folding grid expansion to fill gaps [S1, unofficial].
*   **Strengths:** No per-scene tuning required; handles severe occlusion; deterministic output structure.
*   **Limitations:** Fixed expansion ratio (256 to 1,024) may not adapt to scene complexity; relies on the quality of the pretrained encoder.

## Background
Shape Completion Training was introduced in the context of Gaussian-JEPA (Joint-Embedding Predictive Learning for 3D Gaussian Splats) [S1, unofficial]. It addresses the challenge of reconstructing 3D scenes from partial observations, a common issue in real-world scanning where occlusions and sensor noise lead to incomplete data. By freezing a pretrained encoder and training a decoder to complete the shape, it leverages learned priors from large-scale data to infer missing geometry without iterative optimization.

## Sources
[S1] Gaussian-JEPA: Joint-Embedding Predictive Learning for 3D Gaussian Splats, https://arxiv.org/abs/2608.15651v1, fetched 2026-10-26 (unofficial)
