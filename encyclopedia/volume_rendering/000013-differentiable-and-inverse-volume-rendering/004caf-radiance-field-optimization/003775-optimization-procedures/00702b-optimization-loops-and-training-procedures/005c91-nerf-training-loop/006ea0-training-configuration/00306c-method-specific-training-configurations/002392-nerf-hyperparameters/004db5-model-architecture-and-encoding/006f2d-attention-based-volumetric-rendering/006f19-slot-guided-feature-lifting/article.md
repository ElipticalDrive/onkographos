# Slot-guided Feature Lifting

Also called: SlotLifter

## Overview
Slot-guided Feature Lifting is a method for learning object-centric radiance fields that encodes scene representations from source views using Slot-Attention, lifts 2D features to 3D to capture fine-grained details, and uses a point-slot mapping to decode 3D representations for rendering color and segmentation masks [S1]. It differs from sibling methods like uORF, uOCF, and sVORF by computing only a single radiance field rather than $K$ fields (one per slot), which significantly reduces training time and memory overhead [S1].

## How it works
**Inputs:** A set of $L$ source views $\{I_l\}_{l=1}^L$ (typically $L=1$ for single-view input) and their associated camera parameters.
**Outputs:** Rendered color images $\hat{C}(r)$ and segmentation masks for a target view.

**Data Structures:**
*   $F^{2D}_l \in \mathbb{R}^{H \times W \times D_f}$: 2D feature maps extracted from source view $l$.
*   $S \in \mathbb{R}^{K \times D_s}$: Object-centric scene features (slots), where $K$ is the number of objects.
*   $P \in \mathbb{R}^{N \times 3}$: 3D points sampled along rays in the target view.
*   $E_p \in \mathbb{R}^{N \times D_p}$: Positional embeddings for 3D points.
*   $F_{lift} \in \mathbb{R}^{N \times L \times D_f}$: Lifted feature field containing 2D features projected from all source views.
*   $F_p \in \mathbb{R}^{N \times D_p}$: Final 3D point features.
*   $A_p \in \mathbb{R}^{N \times K}$: Attention weights used for density estimation.
*   $W_p \in \mathbb{R}^{N \times (K+1)}$: Point-slot mapping weights.

**Steps:**

1.  **Extract 2D Features:** Extract feature maps $F^{2D}_l$ from each source view $I_l$ using a shared encoder network [S1].
2.  **Slot-Attention Encoding:** Obtain object-centric scene features $S = \{s_1, \dots, s_K\}$ from the 2D features $\{F^{2D}_l\}$ using Slot-Attention [S1]. The slots are initialized with learnable parameters rather than random sampling to improve optimization [S1].
3.  **Ray Sampling:** For a target view, sample $N$ points $P = \{p_1, \dots, p_N\}$ along each ray $r$ [S1].
4.  **Feature Lifting:** Project each 3D point $p$ onto the image coordinates $\pi(p)$ of each source view to retrieve corresponding 2D features. Construct the lifted feature field $F_{lift}$:
    $$
    F_{lift}(p) = \left[ F^{2D}_1[\pi(p)], \dots, F^{2D}_L[\pi(p)] \right]
    $$
    where $F^{2D}_l[\pi(p)]$ denotes the feature vector at pixel coordinates $\pi(p)$ in view $l$ [S1].
5.  **Feature Fusion:** Pool the multi-view lifted features to obtain 3D point features $F_p$. For multi-view input:
    $$
    F_p = \text{MLP}\left( \left[ \text{Mean}(F_{lift}), \text{Var}(F_{lift}) \right] \right) + E_p
    $$
    For single-view input ($L=1$), ignore the variance term:
    $$
    F_p = \text{MLP}(F_{lift}) + E_p
    $$
    Here, $E_p$ preserves spatial information [S1].
6.  **Random Masking (Training Only):** To prevent degenerate reliance on lifted features, randomly mask $F_p$ using only $E_p$. The masking ratio follows a cosine annealing schedule from 0.99 to 0 over 30,000 steps [S1].
7.  **Point-Slot Mapping:** Calculate point-slot mapping weights $W_p$ using a cross-attention-based allocation transformer between $F_p$ and $S$ [S1].
8.  **Density Estimation:** Compute attention weights $A_p$ for density prediction:
    $$
    A_p = q(F_p) \cdot k(S_1)^T
    $$
    where $S_1$ refers to the $K$ object slots (ignoring the empty slot). Estimate density $\sigma_i$ for the $i$-th point:
    $$
    \sigma_i = \sum_{j=1}^{K+1} W_p^{i,j} \odot \text{ReLU}(A_p^{i,j})
    $$
    where $\odot$ is the Hadamard product, and the sum includes the empty slot (index $K+1$) [S1].
9.  **Color Prediction:** Add $F_p$ with $E_p$ and pass through an MLP to predict color $c_i$ [S1].
10. **Volume Rendering:** Render the final color $\hat{C}(r)$ using standard volume rendering:
    $$
    \hat{C}(r) = \sum_{i=1}^{N} T_i [1 - \exp(-\sigma_i \delta_i)] c_i
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and $\delta_i$ is the distance between adjacent volumes [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SlotGuidedFeatureLifting:
    def __init__(self, num_slots, feature_dim, hidden_dim):
        self.K = num_slots
        self.D_f = feature_dim
        self.D_p = hidden_dim
        # MLP for feature fusion and color prediction
        self.fusion_mlp = nn.Sequential(
            nn.Linear(2 * feature_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        self.color_mlp = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3) # RGB
        )
        # Learnable slot initialization
        self.slots = nn.Parameter(torch.randn(self.K, hidden_dim))
        # Positional embedding generator (e.g., sine/cosine)
        self.pos_embed = PositionalEmbedding(hidden_dim)
        # Masking schedule parameters
        self.max_mask_ratio = 0.99
        self.mask_steps = 30000
        self.current_step = 0

    def extract_2d_features(self, views):
        # views: [B, L, H, W, 3]
        # Returns F_2D: [B, L, H, W, D_f]
        # Implementation depends on specific encoder (e.g., ResNet)
        # [G] Placeholder for encoder logic
        pass

    def slot_attention(self, f_2d):
        # f_2d: [B, L, H, W, D_f]
        # Flatten spatial dims
        B, L, H, W, D_f = f_2d.shape
        f_flat = f_2d.view(B, L * H * W, D_f)
        
        # Slot-Attention mechanism
        # [G] Standard Slot-Attention implementation:
        # 1. Initialize slots with self.slots
        # 2. Attention: q=slots, k=f_flat, v=f_flat
        # 3. Update slots with mean-pooled attended values
        # 4. Return updated slots S: [B, K, D_s]
        slots = self.slots.unsqueeze(0).expand(B, -1, -1)
        # Simplified attention logic for brevity
        # In practice, use the specific Slot-Attention module from the paper
        return slots 

    def lift_features(self, f_2d, points_3d, camera_params):
        # f_2d: [B, L, H, W, D_f]
        # points_3d: [B, N, 3]
        # camera_params: projection matrices
        # Returns F_lift: [B, N, L, D_f]
        B, N, _ = points_3d.shape
        L = f_2d.shape[1]
        
        # Project 3D points to 2D coordinates for each view
        # [G] Implement projection using camera intrinsics/extrinsics
        # pi(p) -> (u, v) normalized to [-1, 1] or [0, 1]
        coords = project_to_2d(points_3d, camera_params) # [B, N, 2]
        
        # Sample features from F_2D using bilinear interpolation
        # F_lift[b, n, l, :] = F_2D[b, l, pi(p_n)]
        F_lift = torch.zeros(B, N, L, self.D_f, device=points_3d.device)
        for l in range(L):
            # Grid sample F_2D[:, l, :, :, :] at coords
            F_lift[:, :, l, :] = grid_sample(f_2d[:, l, :, :, :], coords)
        return F_lift

    def fuse_features(self, F_lift, points_3d, is_single_view=False):
        # F_lift: [B, N, L, D_f]
        # Returns F_p: [B, N, D_p]
        B, N, L, D_f = F_lift.shape
        
        # Mean pooling over views
        mean_feat = F_lift.mean(dim=2) # [B, N, D_f]
        
        if not is_single_view:
            # Variance pooling over views
            var_feat = F_lift.var(dim=2) # [B, N, D_f]
            fused = torch.cat([mean_feat, var_feat], dim=-1) # [B, N, 2*D_f]
        else:
            fused = mean_feat # [B, N, D_f]
            
        # MLP
        F_p = self.fusion_mlp(fused) # [B, N, D_p]
        
        # Add positional embeddings
        E_p = self.pos_embed(points_3d) # [B, N, D_p]
        F_p = F_p + E_p
        
        return F_p

    def get_mask_ratio(self):
        # Cosine annealing from 0.99 to 0 over 30k steps
        progress = min(self.current_step / self.mask_steps, 1.0)
        return self.max_mask_ratio * 0.5 * (1 + torch.cos(torch.tensor(progress * torch.pi)))

    def forward(self, views, points_3d, camera_params, is_single_view=False):
        # 1. Extract 2D features
        F_2D = self.extract_2d_features(views)
        
        # 2. Slot-Attention
        S = self.slot_attention(F_2D) # [B, K, D_s]
        
        # 3. Lift features
        F_lift = self.lift_features(F_2D, points_3d, camera_params)
        
        # 4. Fuse features
        F_p = self.fuse_features(F_lift, points_3d, is_single_view)
        
        # 5. Random Masking
        mask_ratio = self.get_mask_ratio()
        if self.training and mask_ratio > 0:
            mask = torch.rand(F_p.shape, device=F_p.device) < mask_ratio
            # Replace masked features with positional embeddings only
            E_p = self.pos_embed(points_3d)
            F_p = torch.where(mask.unsqueeze(-1), E_p, F_p)
            
        # 6. Point-Slot Mapping (Simplified)
        # [G] Implement cross-attention between F_p and S
        # A_p = q(F_p) @ k(S)^T
        # [G] Assume q and k are linear projections
        q_fp = F_p @ self.q_proj # [B, N, K]
        k_s = S @ self.k_proj # [B, K, K]
        A_p = torch.bmm(q_fp, k_s.transpose(1, 2)) # [B, N, K]
        
        # 7. Density Estimation
        # sigma_i = sum(W_p^{i, 1:K+1} o ReLU(A_p^{i, 1:K+1}))
        # [G] W_p is obtained from another MLP/Attention
        # For simplicity, assume W_p is learned or derived from A_p
        # The paper uses W_p from point-slot mapping module
        # Here we approximate with softmax of A_p for demonstration
        W_p = F.softmax(A_p, dim=-1) # [B, N, K]
        
        # ReLU on attention weights
        A_p_relu = F.relu(A_p)
        
        # Density calculation
        # Note: Paper sums over K+1 slots (including empty). 
        # Assuming W_p has shape [B, N, K+1] and A_p has [B, N, K+1]
        # For this implementation, we assume K slots + 1 empty slot
        # Extend A_p and W_p to K+1
        A_p_ext = torch.cat([A_p_relu, torch.zeros_like(A_p[:, :, :1])], dim=-1)
        W_p_ext = torch.cat([W_p, torch.zeros_like(W_p[:, :, :1])], dim=-1)
        
        sigma = torch.sum(W_p_ext * A_p_ext, dim=-1) # [B, N]
        
        # 8. Color Prediction
        # Add F_p with E_p (already done in fuse_features, but re-add if masked)
        # Pass into MLP
        c = self.color_mlp(F_p) # [B, N, 3]
        
        # 9. Volume Rendering
        # [G] Standard volume rendering loop
        # T_i = exp(-sum(sigma_j * delta_j))
        # C = sum(T_i * (1 - exp(-sigma_i * delta_i)) * c_i)
        # delta_i is distance between points
        # [G] Implement volume rendering
        pass

    def step(self):
        self.current_step += 1
```

**Parameters:**
*   **Optimizer:** Lion [S1].
*   **Learning Rate:** $5 \times 10^{-5}$ [S1].
*   **Iterations:** 250,000 [S1].
*   **Batch Size:** 4 [S1].
*   **Sampled Rays:** 1,024 rays per scene [S1].
*   **Masking Schedule:** Cosine annealing from 0.99 to 0 over 30,000 steps [S1].

**Edge Cases and Pitfalls:**
*   **Single-View Input:** When $L=1$, the variance term in feature fusion is ignored. The model must handle this branch explicitly [S1].
*   **Empty Slot:** The density calculation includes an "empty slot" (index $K+1$) to account for background or unassigned points. This slot is ignored in the slot-attention output $S_1$ but included in the density summation [S1].
*   **Degenerate Reliance:** Without random masking, the model may rely too heavily on lifted features, ignoring slots. The masking schedule is critical to enforce alignment between slots and 3D point grids [S1].

## Operating characteristics
*   **Time Complexity:** Training time is approximately 30 hours on a single Nvidia RTX 3090 GPU for the Room-Diverse dataset, which is 5x faster than uORF (6 days) [S1].
*   **Memory Complexity:** Uses a single radiance field, whereas uORF, uOCF, and sVORF compute $K$ fields (one per slot), leading to lower memory usage [S1].
*   **Strengths:** Faster convergence due to feature lifting providing detailed information; efficient rendering with fewer sampled rays (1,024 or even 256) due to no auxiliary losses on fully rendered images [S1].
*   **Limitations:** Requires Slot-Attention, which may be sensitive to initialization (mitigated by learnable slot initialization) [S1].

## Background
SlotLifter was introduced to improve object-centric radiance field learning by combining Slot-Attention with feature lifting. It addresses the limitations of NeRF, which lacks generalization and requires per-scene optimization [S1]. The method builds on [Attention-Based Volumetric Rendering](21739-attention-based-volumetric-rendering.md) by incorporating learnable embeddings and attention mechanisms. It was evaluated on synthetic (CLEVR-567, Room-Chair, Room-Diverse, Room-Texture) and real-world (Kitchen-Shiny, Kitchen-Matte) datasets [S1].

## Sources
[S1] SlotLifter: Slot-Guided Feature Lifting for Learning Object-Centric Radiance Fields, https://doi.org/10.1007/978-3-031-72673-6_15, fetched 2026-10-29
