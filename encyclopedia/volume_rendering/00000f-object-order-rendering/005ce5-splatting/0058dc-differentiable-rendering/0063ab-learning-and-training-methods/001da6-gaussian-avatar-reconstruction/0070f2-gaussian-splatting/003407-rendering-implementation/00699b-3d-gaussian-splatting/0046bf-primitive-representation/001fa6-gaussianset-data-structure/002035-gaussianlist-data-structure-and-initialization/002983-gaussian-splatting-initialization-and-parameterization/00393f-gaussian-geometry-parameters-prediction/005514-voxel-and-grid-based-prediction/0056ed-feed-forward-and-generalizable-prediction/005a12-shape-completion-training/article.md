# Shape Completion Training

Also called: Decoder-based Shape Completion, Folding Grid Expansion for Shape Completion

## Overview
Shape Completion Training is a feed-forward method that predicts all 14 3D Gaussian attributes from partial inputs without requiring target coordinates or per-scene optimization [S1, unofficial]. It distinguishes itself from sibling methods by employing a fixed 2×2 folding grid to expand a small set of coarse primitives into a dense Gaussian set, enabling reconstruction of missing geometry in occluded or sparse scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Partial 3D data (e.g., point clouds or voxel grids) with missing regions. No target coordinates are provided during inference [S1, unofficial].
*   **Output:** A set of 1,024 3D Gaussians $\mathcal{G} = \{\mu_i, \Sigma_i, \alpha_i, c_i\}_{i=1}^{1024}$, where $\mu$ is mean, $\Sigma$ is covariance (defined by scale $s$ and rotation $q$), $\alpha$ is opacity, and $c$ is color (DC spherical harmonic coefficients) [S1, unofficial].

**Data Structures:**
1.  **Coarse Primitives:** A list of 256 entities. Each entity contains 14 attributes (position, scale, rotation, opacity, color) [S1, unofficial].
2.  **Folding Grid:** A fixed $2 \times 2$ grid structure used to spatially expand the coarse primitives [S1, unofficial].
3.  **Final Gaussians:** A list of 1,024 entities with the same 14-attribute layout [S1, unofficial].

**Algorithm Steps:**
1.  **Encoder Freezing:** Initialize a pretrained feature encoder and freeze its weights [S1, unofficial].
2.  **Decoder Initialization:** Initialize an identical decoder architecture to the encoder's output space [S1, unofficial].
3.  **Feature Processing:** Pass the input features through four Transformer layers in the decoder (width 256, 8 attention heads) to predict 256 coarse primitive attributes [S1, unofficial].
4.  **Grid Expansion:** Expand the 256 coarse primitives to 1,024 Gaussians using the fixed 2×2 folding grid [S1, unofficial].
5.  **Loss Computation:** Compute the total loss $\mathcal{L}$ as a weighted sum of geometric and attribute losses:
    $$ \mathcal{L} = \mathcal{L}_{Chamfer}^{2} + \mathcal{L}_{NN} + 0.1 \left( \mathcal{L}_{L1}(\alpha, s, c) + \mathcal{L}_{sign\text{-}inv}(q) \right) $$
    Where:
    *   $\mathcal{L}_{Chamfer}^{2}$ is the squared $\ell_2$ Chamfer distance between predicted and target points [S1, unofficial].
    *   $\mathcal{L}_{NN}$ is the nearest-neighbor attribute matching loss [S1, unofficial].
    *   $\mathcal{L}_{L1}$ is the $\ell_1$ loss applied to opacity $\alpha$, scale $s$, and DC color coefficients $c$ [S1, unofficial].
    *   $\mathcal{L}_{sign\text{-}inv}(q)$ is the sign-invariant quaternion similarity loss for rotation $q$ [S1, unofficial].
    *   The aggregate attribute term is scaled by a weight of 0.1 [S1, unofficial].
6.  **Optimization:** Optimize the decoder weights using AdamW for 80 epochs [S1, unofficial].
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
        self.transformer_blocks = nn.ModuleList([
            nn.TransformerEncoderLayer(d_model=width, nhead=heads) 
            for _ in range(layers)
        ])
        self.attr_proj = nn.Linear(width, 14)
        self.num_coarse = num_coarse
        self.num_final = num_final
        self.expansion_factor = num_final // num_coarse

    def forward(self, x):
        # x: [Batch, Coarse_Primitives, Width]
        for block in self.transformer_blocks:
            x = block(x)
        coarse_attrs = self.attr_proj(x) # [Batch, 256, 14]
        final_attrs = self.expand_folding_grid(coarse_attrs)
        return final_attrs

    def expand_folding_grid(self, coarse_attrs):
        # coarse_attrs: [B, 256, 14]
        expanded = coarse_attrs.repeat_interleave(self.expansion_factor, dim=1) # [B, 1024, 14]
        # Fixed 2x2 grid offsets for position (first 3 dims)
        offsets = torch.tensor([
            [-0.5, -0.5, 0], [0.5, -0.5, 0],
            [-0.5, 0.5, 0], [0.5, 0.5, 0]
        ], device=coarse_attrs.device).float()
        pos = expanded[:, :, :3]
        others = expanded[:, :, 3:]
        grid_indices = torch.arange(self.expansion_factor, device=coarse_attrs.device)
        grid_offsets = offsets[grid_indices]
        grid_offsets = grid_offsets.view(1, 1, 4, 3)
        pos_expanded = pos.view(pos.shape[0], 256, 4, 3)
        pos_final = pos_expanded + grid_offsets
        pos_final = pos_final.view(pos.shape[0], -1, 3)
        return torch.cat([pos_final, others], dim=-1)

def compute_loss(pred_attrs, target_attrs):
    # pred_attrs: [B, 1024, 14], target_attrs: [B, N, 14]
    pred_pos = pred_attrs[:, :, :3]
    target_pos = target_attrs[:, :, :3]
    dists = torch.cdist(pred_pos, target_pos, p=2) # [B, 1024, N]
    
    chamfer_pred = torch.min(dists, dim=2)[0].mean()
    chamfer_target = torch.min(dists, dim=1)[0].mean()
    chamfer_loss = (chamfer_pred + chamfer_target) / 2.0
    
    nn_indices = torch.argmin(dists, dim=2)
    target_nn_attrs = torch.gather(target_attrs, 1, nn_indices.unsqueeze(-1).expand(-1, -1, 14))
    nn_loss = torch.nn.functional.l1_loss(pred_attrs, target_nn_attrs)
    
    # Attribute losses: alpha (idx 10), scale (idx 3:6), color (idx 11:14) [G]
    attr_l1 = torch.nn.functional.l1_loss(pred_attrs[:, :, 10], target_nn_attrs[:, :, 10]) + \
              torch.nn.functional.l1_loss(pred_attrs[:, :, 3:6], target_nn_attrs[:, :, 3:6]) + \
              torch.nn.functional.l1_loss(pred_attrs[:, :, 11:14], target_nn_attrs[:, :, 11:14])
    
    # Sign-invariant quaternion loss (idx 6:10) [G]
    pred_rot = torch.nn.functional.normalize(pred_attrs[:, :, 6:10], p=2, dim=-1)
    target_rot = torch.nn.functional.normalize(target_nn_attrs[:, :, 6:10], p=2, dim=-1)
    dot_prod = torch.sum(pred_rot * target_rot, dim=-1)
    sign_inv_loss = torch.nn.functional.mse_loss(dot_prod, torch.ones_like(dot_prod))
    
    return chamfer_loss + nn_loss + 0.1 * (attr_l1 + sign_inv_loss)

def train_shape_completion(encoder, decoder, dataloader, epochs=80, lr=1e-3, weight_decay=0.05, warmup_epochs=5, batch_size=64):
    optimizer = optim.AdamW(decoder.parameters(), lr=lr, weight_decay=weight_decay)
    scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
    device = next(decoder.parameters()).device
    for epoch in range(epochs):
        if epoch < warmup_epochs:
            for param_group in optimizer.param_groups:
                param_group['lr'] = lr * (epoch + 1) / warmup_epochs
        for batch in dataloader:
            inputs, targets = batch
            inputs = inputs.to(device)
            targets = targets.to(device)
            with torch.no_grad():
                features = encoder(inputs)
            preds = decoder(features)
            loss = compute_loss(preds, targets)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
        scheduler.step()
```

## Operating characteristics
*   **Time Complexity:** Inference is feed-forward with no per-scene optimization state [S1, unofficial]. Training requires 80 epochs per decoder seed [S1, unofficial].
*   **Memory Complexity:** Low inference memory footprint as no per-scene optimization state is stored [S1, unofficial]. Training memory depends on batch size (64) and the size of the feature maps [S1, unofficial].
*   **Performance:** Evaluated on 520 held-out objects with visible ratios {0.3, 0.5, 0.7} and three crop seeds [S1, unofficial]. Render metrics use the seed-0 decoder at 50% visibility and four fixed views per object [S1, unofficial].
*   **Strengths:** No per-scene tuning required; handles severe occlusion; deterministic output structure [S1, unofficial].
*   **Limitations:** Fixed expansion ratio (256 to 1,024) may not adapt to scene complexity; relies on the quality of the pretrained encoder [S1, unofficial].

## Background
Shape Completion Training was introduced in the context of Gaussian-JEPA (Joint-Embedding Predictive Learning for 3D Gaussian Splats) [S1, unofficial]. It addresses the challenge of reconstructing 3D scenes from partial observations, a common issue in real-world scanning where occlusions and sensor noise lead to incomplete data [S1, unofficial]. By freezing a pretrained encoder and training a decoder to complete the shape, it leverages learned priors from large-scale data to infer missing geometry without iterative optimization [S1, unofficial].

## Sources
[S1] Gaussian-JEPA: Joint-Embedding Predictive Learning for 3D Gaussian Splats, https://arxiv.org/abs/2608.15651v1, fetched 2026-10-26 (unofficial)
