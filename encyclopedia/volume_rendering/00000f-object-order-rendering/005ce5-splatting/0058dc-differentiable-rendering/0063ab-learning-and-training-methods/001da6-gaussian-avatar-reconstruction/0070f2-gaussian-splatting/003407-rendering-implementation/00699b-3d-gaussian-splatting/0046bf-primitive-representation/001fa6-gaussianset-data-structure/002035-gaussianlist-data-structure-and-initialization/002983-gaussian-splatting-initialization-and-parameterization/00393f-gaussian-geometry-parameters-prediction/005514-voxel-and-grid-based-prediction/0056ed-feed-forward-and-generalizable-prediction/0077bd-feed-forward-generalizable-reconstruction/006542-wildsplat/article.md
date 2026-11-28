# WildSplat

Also called: WildSplat: Feedforward Gaussian Splatting from Unposed In-the-Wild Images

## Overview
WildSplat is a feedforward 3D Gaussian Splatting framework for appearance-conditioned novel-view synthesis from sparse, unposed, in-the-wild images [S1]. It explicitly decouples geometry from appearance to handle inconsistent photometric conditions, such as varying weather and lighting, which typically cause entangled representations in existing methods to fail [S1]. Unlike optimization-based methods that require precomputed poses and dense inputs, or other feedforward methods that assume photometric consistency, WildSplat reconstructs a single appearance-invariant 3D geometry and renders it with target appearance conditioned on a reference image in a single forward pass [S1].

## How it works
**Inputs and Outputs:**
The model takes a set of $N$ context images $\{I_i\}$ and a single reference image $I_{ref}$ [S1]. It outputs:
1.  3D Gaussian geometry attributes $\mathcal{G}$, consisting of positions $\mu \in \mathbb{R}^{G \times 3}$, rotations $q \in \mathbb{R}^{G \times 4}$, scales $s \in \mathbb{R}^{G \times 3}$, and opacities $\alpha \in \mathbb{R}^G$, where $G$ is the number of Gaussians [S1].
2.  Conditioned spherical harmonics coefficients $\mathcal{C}_{ref} \in \mathbb{R}^{G \times 3 \times (k+1)}$ of degree $k$, conditioned on $I_{ref}$ [S1].
3.  Camera poses $\{p_i\}$ for the context views [S1].

**Data Structures:**
*   **Patch Tokens:** $T_i^{(0)} = E_{geo}(I_i) \in \mathbb{R}^{P \times D}$, where $P$ is the number of patches and $D$ is the feature dimension [S1].
*   **Camera Tokens:** Learnable tokens $C_i \in \mathbb{R}^{5 \times D}$ concatenated with patch tokens [S1].
*   **Content Features:** Aggregated multi-scale representations $\mathbf{F}_{content}$ extracted from the geometry branch, serving as the geometric scaffold [S1].
*   **Reference Tokens:** $T_{ref} \in \mathbb{R}^{P_a \times D}$, local dense appearance tokens from the reference image, where $P_a$ is the number of reference patches [S1].
*   **Overlap/Scale Matrices:** $S$ (overlap ratio) and $R$ (scale/depth ratio) derived from SfM points for view sampling [S1].

**Parameters:**
*   Loss weights: $\lambda_1 = 10$ (pose), $\lambda_2 = 1.0$ (MSE), $\lambda_3 = 0.05$ (SSIM), $\lambda_4 = 0.05$ (LPIPS) [S1].
*   View sampling thresholds: $\tau_s$ (overlap), $\tau_r$ (scale), $\tau_n$ (neighbor count) [S1].
*   Number of sampled target views for training: $M$ [S1].
*   Number of neighbors to sample per step: $K$ [S1].

**Steps:**

1.  **Geometry Branch (Pose and Geometry Prediction):**
    *   Process each context image $I_i$ independently through a DINOv2-based geometry encoder $E_{geo}$ to obtain initial patch tokens $T_i^{(0)}$ [S1].
    *   Concatenate learnable camera tokens $C_i$ with $T_i^{(0)}$ [S1].
    *   Refine tokens through $L$ alternating attention blocks to produce refined patch tokens $T_i^{(k)}$ and camera tokens $C_i^{(k)}$ for each layer $k$ [S1].
    *   Extract and aggregate multi-scale representations from a subset of transformer layers $\mathcal{K} = \{l_1, \dots, l_K\}$ to form the content features $\mathbf{F}_{content}$ [S1].
    *   Predict camera poses $\{p_i\}$ and Gaussian geometry $\mathcal{G}$ ($\mu, q, s, \alpha$) from the refined tokens [S1].

2.  **Appearance Branch (Conditioned Appearance Prediction):**
    *   Extract reference appearance tokens $T_{ref}$ from $I_{ref}$ using an appearance encoder [S1].
    *   **Global Pre-modulation:** To prevent original photometric properties from interfering, use the global descriptor $T_{[CLS]}$ from the content features. Decode $T_{[CLS]}$ via an AdaLN-Zero layer into scale $\gamma$, shift $\beta$, and gating $\eta$ [S1].
    *   Modulate the content features $\mathbf{F}_{content}$ (treated as queries $Q$) using the reference tokens $T_{ref}$ (keys $K$, values $V$) in a cross-attention mechanism, pre-modulated by $\gamma, \beta, \eta$ [S1].
    *   Output conditioned spherical harmonics coefficients $\mathcal{C}_{ref}$ [S1].

3.  **Conditioned Rasterization:**
    *   Render novel views $\{\hat{I}_i\}$ using tile-based differentiable rasterization $\mathcal{R}$ of 3D Gaussian Splatting [S1]:
        $$ \{\hat{I}_i\} = \mathcal{R}(\mathcal{G}, \mathcal{C}_{ref}, \{p_i\}) $$
    *   Here, $\mathcal{G}$ provides the geometry, $\mathcal{C}_{ref}$ provides the appearance, and $\{p_i\}$ provides the camera poses [S1].

4.  **Geometry-guided View Sampling (Training Input Selection):**
    *   Construct an overlap matrix $S$ and scale matrix $R$ using Structure-from-Motion (SfM) points [S1].
    *   Filter valid view pairs where $S \geq \tau_s$ and $R \geq \tau_r$ [S1].
    *   Identify valid starting rows $C$ where the sum of valid columns exceeds $\tau_n$ [S1].
    *   Initialize a queue with a uniform sample $c$ from $C$ and an index set $I = \{c\}$ [S1].
    *   Iteratively pop $u$ from the queue, sample up to $K$ neighbors from valid unvisited views, push them to the queue, and add to $I$ until $N$ views are selected [S1].

5.  **Joint Multi-Reference Supervision:**
    *   Randomly sample $M$ images $\{I_m\}$ from the batch as target references [S1].
    *   Extract conditioned appearance $\{\mathcal{C}_{ref,m}\}$ for each $I_m$ [S1].
    *   Render images $\{\hat{I}_m\}$ conditioned on their respective appearance [S1].
    *   Apply color jittering to input images and random cropping to references prior to feature extraction to prevent appearance leakage [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class WildSplat:
    def __init__(self, num_gaussians, sh_degree, num_layers, feature_dim):
        self.num_gaussians = num_gaussians
        self.sh_degree = sh_degree
        self.num_layers = num_layers
        self.feature_dim = feature_dim
        
        # Geometry Encoder (DINOv2-based)
        self.geo_encoder = DINOv2Encoder()
        
        # Learnable Camera Tokens
        self.camera_tokens = nn.Parameter(torch.randn(5, feature_dim))
        
        # Transformer Blocks for refinement
        self.transformer_blocks = nn.ModuleList([
            TransformerBlock() for _ in range(num_layers)
        ])
        
        # Appearance Encoder
        self.appearance_encoder = AppearanceEncoder()
        
        # AdaLN-Zero for pre-modulation
        self.adaln_zero = AdaLNZero(feature_dim)
        
        # Cross-Attention for appearance injection
        self.cross_attn = CrossAttention(feature_dim)
        
        # Heads for geometry prediction
        self.geo_head = GeometryHead(num_gaussians)
        
        # Heads for SH coefficients prediction
        self.sh_head = SHHead(num_gaussians, sh_degree)

    def forward(self, context_images, reference_image):
        """
        context_images: List of N images [B, 3, H, W]
        reference_image: Image [B, 3, H, W]
        """
        N = len(context_images)
        
        # 1. Geometry Branch
        # Encode context images
        patch_tokens_list = []
        for i in range(N):
            # E_geo(I_i) -> [B, P, D]
            tokens = self.geo_encoder(context_images[i])
            patch_tokens_list.append(tokens)
            
        # Concatenate camera tokens and refine
        # C_i shape: [B, 5, D]
        camera_tokens_expanded = self.camera_tokens.unsqueeze(0).expand(N, -1, -1, -1)
        
        refined_tokens_list = []
        for i in range(N):
            # Concatenate [B, P, D] and [B, 5, D] along patch dim
            combined = torch.cat([patch_tokens_list[i], camera_tokens_expanded[i]], dim=1)
            refined = combined
            for block in self.transformer_blocks:
                refined = block(refined)
            refined_tokens_list.append(refined)
            
        # Extract content features from specific layers (aggregation logic omitted for brevity)
        # F_content shape: [B, P_content, D]
        content_features = self.aggregate_content_features(refined_tokens_list)
        
        # Predict Geometry and Poses
        # geo_head takes content features or refined tokens
        geometry = self.geo_head(content_features) # Returns mu, q, s, alpha
        poses = self.extract_poses(refined_tokens_list) # Returns p_i
        
        # 2. Appearance Branch
        # Encode reference image
        # T_ref shape: [B, P_a, D]
        ref_tokens = self.appearance_encoder(reference_image)
        
        # Global Pre-modulation
        # Get CLS token from content features (or global pool)
        cls_token = content_features[:, 0, :] # Assuming first token is CLS
        # AdaLN-Zero outputs gamma, beta, eta
        gamma, beta, eta = self.adaln_zero(cls_token)
        
        # Modulate content features
        # T_tilde = gamma * F_content + beta (simplified AdaLN)
        # Note: Source says "modulate the appearance information within the content feature"
        # using gamma, beta, eta. Usually AdaLN applies to the query/key/value or the feature itself.
        # Assuming modulation of content features (Queries) before attention.
        modulated_content = gamma.unsqueeze(1) * content_features + beta.unsqueeze(1)
        
        # Cross-Attention
        # Q = modulated_content, K = V = ref_tokens
        # Output shape: [B, P_content, D]
        attended_features = self.cross_attn(modulated_content, ref_tokens, ref_tokens)
        
        # Predict Conditioned SH Coefficients
        c_ref = self.sh_head(attended_features) # Returns C_ref
        
        # 3. Rasterization
        # Render using 3DGS rasterizer
        # R(G, C_ref, {p_i})
        rendered_images = rasterize_3dgs(geometry, c_ref, poses)
        
        return rendered_images, geometry, c_ref, poses

    def compute_loss(self, rendered_images, gt_images, poses, gt_poses=None):
        # Photometric Loss
        l_mse = F.mse_loss(rendered_images, gt_images)
        l_ssim = ssim_loss(rendered_images, gt_images)
        l_lpips = lpips_loss(rendered_images, gt_images)
        
        l_rgb = 1.0 * l_mse + 0.05 * l_ssim + 0.05 * l_lpips
        
        # Pose Loss (Distillation loss from AnySplat)
        if gt_poses is not None:
            l_pose = pose_distillation_loss(poses, gt_poses)
        else:
            l_pose = torch.tensor(0.0)
            
        total_loss = l_rgb + 10.0 * l_pose
        return total_loss

    def geometry_guided_view_sampling(self, sfm_points, num_views, k_neighbors, tau_s, tau_r, tau_n):
        """
        Selects N views based on overlap and scale consistency using SfM points.
        """
        # Construct overlap matrix S and scale matrix R
        # S[i, j] = ratio of shared 3D points between view i and j
        # R[i, j] = minimum depth ratio between view i and j
        S = compute_overlap_matrix(sfm_points)
        R = compute_scale_matrix(sfm_points)
        
        # Filter valid pairs
        valid_mask = (S >= tau_s) & (R >= tau_r)
        
        # Find valid starting rows
        col_sums = valid_mask.sum(dim=1)
        valid_starts = torch.where(col_sums > tau_n)[0]
        
        if len(valid_starts) == 0:
            # Fallback or error handling
            return torch.arange(num_views)
            
        # Initialize queue and index set
        start_idx = valid_starts[torch.randint(len(valid_starts), (1,))]
        queue = [start_idx]
        selected_indices = {start_idx}
        
        while len(selected_indices) < num_views and queue:
            u = queue.pop(0)
            # Sample up to K neighbors from valid unvisited views
            candidates = torch.where(valid_mask[u] & (~torch.tensor(list(selected_indices))))[0]
            if len(candidates) > 0:
                num_to_sample = min(k_neighbors, len(candidates))
                sampled = candidates[torch.randperm(len(candidates))[:num_to_sample]]
                for v in sampled:
                    queue.append(v.item())
                    selected_indices.add(v.item())
                    
        return list(selected_indices)
```

**Edge Cases and Pitfalls:**
*   **Varying Focal Lengths:** Standard sampling relying on camera extrinsics fails on in-the-wild datasets where minor pose shifts alter captured content due to varying focal lengths. Mitigated by overlap-based sampling using SfM points [S1].
*   **Photometric Interference:** Content features from the geometry branch retain original photometric properties, potentially interfering with target appearance conditioning. Mitigated by global pre-modulation using AdaLN-Zero to shift features into the target appearance domain [S1].
*   **Appearance Leakage:** Without proper regularization, the geometry might adapt to the reference appearance. Mitigated by joint multi-reference training and color jittering/random cropping [S1].

## Operating characteristics
*   **Time Complexity:** Single forward pass for reconstruction and rendering, significantly faster than optimization-based methods (NeRF-W, GS-W) which require per-scene optimization [S1].
*   **Memory Complexity:** Depends on the number of Gaussians $G$ and the feature dimensions of the transformer backbone. Not explicitly given in the sources.
*   **Measured Performance:**
    *   On MegaScenes with 12 views: PSNR 19.20, SSIM 0.683, LPIPS 0.210 [S1].
    *   On Phototourism with 12 views: PSNR 21.47, SSIM 0.788, LPIPS 0.137 [S1].
*   **Scaling:** Works with sparse inputs (4, 8, 12 views). Performance improves with more views but remains efficient compared to dense-input methods [S1].
*   **Strengths:** Handles unposed, in-the-wild images; decouples geometry and appearance; fast feedforward inference; robust to photometric variations [S1].
*   **Limitations:** Requires SfM points for view sampling during training (though not necessarily for inference if poses are predicted); performance may degrade if overlap is insufficient [S1].

## Background
WildSplat was introduced by X. Zhang et al. in 2024 as the first feedforward 3D Gaussian Splatting framework capable of appearance-conditioned novel-view synthesis for unposed in-the-wild images [S1]. It addresses the limitations of previous feedforward methods (pixelSplat, MVSplat) that assume photometric consistency and optimization-based methods that require precomputed poses [S1]. The method builds on VGGT-like architectures for geometry estimation and leverages DINOv2 for feature extraction [S1].

## Sources
[S1] WildSplat: Feedforward Gaussian Splatting from Unposed In-the-Wild Images, https://doi.org/10.1007/978-3-032-37324-3_2, fetched 2026-10-29
