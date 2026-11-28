# InfiniSplat v2

Also called: InfiniSplat

## Overview
InfiniSplat v2 is a feed-forward Gaussian reconstruction method for arbitrary input views that decouples geometry and attribute prediction, uses ground-truth (GT) camera poses for alignment during training, and employs large-angle target view supervision to improve rendering quality under large viewpoint changes [S1, unofficial]. It differs from its parent [Feed-forward Generalizable Reconstruction](16746-feed-forward-generalizable-reconstruction.md) and sibling [DA3-GS](16423-splatt3r.md) by freezing the geometry branch to prevent geometric degradation caused by joint optimization, and by using GT poses to avoid pose errors that misguide the network under large-angle supervision [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ input RGB images $I_C = \{I_i\}_{i \in C}$ and optional pose conditions $Q_C$ (camera intrinsics and poses). In the pose-free setting, $Q_C = \emptyset$ [S1, unofficial].
*   **Outputs:** A set of $M$ 3D Gaussians $\mathcal{G} = \{(\mu_m, \xi_m)\}_{m=1}^M$, where $\mu_m \in \mathbb{R}^3$ is the center and $\xi_m = (\alpha_m, s_m, q_m, c_m)$ contains opacity, three-axis scale, rotation (quaternion), and zeroth-order spherical-harmonic (SH) color [S1, unofficial].

**Data Structures:**
*   **Gaussian Set:** A list of $M$ Gaussians. Each Gaussian stores $\mu_m, \alpha_m, s_m, q_m, c_m$.
*   **Hidden State:** A 16-dimensional vector per Gaussian, updated during Stage II refinement [S1, unofficial].
*   **KNN Graph:** A global graph storing neighbor indices for each Gaussian, reused across refinement steps [S1, unofficial].

**Parameters:**
*   Voxel merging side length: $0.004$ in the scale-normalized reconstruction frame [S1, unofficial].
*   Opacity pruning threshold $\tau_\alpha$: $0.01$ [S1, unofficial].
*   Mask thresholds: $\tau_{abs} = 0.03$, $\tau_{rel} = 0.01$ [S1, unofficial].
*   Learning rates: Stage I: $10^{-6}$ (last 10 attribute blocks), $2 \times 10^{-4}$ (Gaussian head). Stage II: $2 \times 10^{-4}$ [S1, unofficial].
*   AdamW: $\beta = (0.9, 0.95)$, global batch size 16, L2 gradient clipping at 1.0 [S1, unofficial].

**Algorithm:**

The method consists of two stages. Stage I initializes the scene using a dual-branch architecture. Stage II refines attributes using input-view feedback.

### Stage I: Initialization

1.  **Feature Extraction:**
    *   Initialize two branches, $\Phi_{geo}$ and $\Phi_{attr}$, from pretrained DA3 weights [S1, unofficial].
    *   Freeze $\Phi_{geo}$. Fine-tune only the last $K=10$ Transformer blocks of $\Phi_{attr}$ and the Gaussian head [S1, unofficial].
    *   Compute predicted depth $\hat{D}$ and cameras $\hat{\Pi}_C$ using $\Phi_{geo}(I_C; Q_C)$ [S1, unofficial].
    *   Compute multi-layer patch features $F_C$ using $\Phi_{attr}(I_C; Q_C)$ [S1, unofficial].

2.  **Scale Alignment:**
    *   Normalize predicted depths and translations by $s_{norm} > 0$ [S1, unofficial].
    *   Fit an Umeyama similarity transform from the GT trajectory to the normalized predicted trajectory using only input-camera center correspondences. Retain the scale factor $s_C$ [S1, unofficial].
    *   Align GT poses: $R^\star_v = R_v$ and $t^\star_v = s_C t_v$ [S1, unofficial].

3.  **Point Construction and Pruning:**
    *   Construct initial Gaussian centers $\mu_m$ from normalized predicted z-depth, GT intrinsics, and scale-aligned GT poses [S1, unofficial].
    *   Map $F_C$ to per-pixel attributes $\xi$ using a DPT decoder [S1, unofficial].
    *   Merge Gaussians using voxel merging with side length $0.004$ [S1, unofficial].
    *   Prune Gaussians with opacity $\alpha_m < \tau_\alpha = 0.01$ [S1, unofficial].
    *   Let the initial scene be $\mathcal{G}_0$ with $M$ Gaussians.

4.  **Rendering and Loss:**
    *   Render $\mathcal{G}_0$ into input views $C$ and target views $T$ using aligned cameras $\hat{\Pi}^\star_v$: $\hat{I}_{bv} = \mathcal{R}(\mathcal{G}_0; \hat{\Pi}^\star_v)$ [S1, unofficial].
    *   Construct visibility masks $M_v$ for target pixels:
        *   Retain if GT depth $> 10^{-6}$ and passes validity mask.
        *   Retain if at least one input view places the point in front of the camera, within bounds, with valid sampled depth $d_{src}$, and $|z_{proj} - d_{src}| \le \max(\tau_{abs}, \tau_{rel} d_{src})$ [S1, unofficial].
    *   Compute LPIPS spatial distance map $d_v = \text{LPIPS}_{spatial}(M_v \odot \hat{I}_{bv}, M_v \odot I_v)$, zeroing invalid regions before normalization [S1, unofficial].
    *   Compute Stage I loss $\mathcal{L}_I$:
        $$ \mathcal{L}_I = \frac{\sum_{v \in V} \sum_{p} M_v(p) [e_v(p) + d_v(p)]}{\max(1, \sum_{v \in V} \sum_{p} M_v(p))} $$
        where $e_v(p)$ is the RGB-channel-averaged absolute error [S1, unofficial].
    *   Optimize $\Phi_{attr}$ and Gaussian head for 25,000 steps (2,000 warmup, cosine decay to 0.1 peak) [S1, unofficial].

### Stage II: Refinement

1.  **Architecture:**
    *   Freeze $\mathcal{G}_0$ and $\Phi_{attr}$ backbone. Train a DPT feature projection (1x1 conv to 8 channels) and a refiner [S1, unofficial].
    *   Refiner input: Concatenation of image features, current attributes, gradient feedback, neighbor distances, and 16-dim hidden state, projected to 64 dims [S1, unofficial].
    *   Process through four KNN Transformer blocks (single-head attention over 8 neighbors, LayerNorm, MLP, residual) [S1, unofficial].
    *   Output: MLP predicts increments for attributes and hidden state [S1, unofficial].

2.  **Feedback and Updates:**
    *   Render current scene into input views. Compute feedback loss $\ell_{fb} = 0.8 L_1 + 0.2 L_{DSSIM}$, where $L_{DSSIM} = (1 - \text{SSIM})/2$ [S1, unofficial].
    *   Compute gradients w.r.t attributes, normalize, and detach [S1, unofficial].
    *   Update attributes:
        *   Scales: $s_{new} = s_{old} \exp(\Delta \log s)$ [S1, unofficial].
        *   Opacity-logit, local rotation-vector, RGB-logit: $a \tanh(\text{raw})$ with $a = 0.1, 0.25, 1.0$ respectively [S1, unofficial].
        *   Hidden state: $0.1 \tanh(\text{raw})$ [S1, unofficial].
        *   Clamp logit inputs to $[10^{-6}, 1-10^{-6}]$ [S1, unofficial].
    *   Centers, counts, and correspondence remain fixed [S1, unofficial].

3.  **Multi-step Objective:**
    *   Sample $K_r \in \{1, 2, 3\}$ updates during training, use 3 at inference [S1, unofficial].
    *   First-order surrogate loss: Let $x_k$ be updated parameters. Compute rendering gradients on leaf copies $z_k = \text{sg}(x_k)$ to form $r_k = \text{sg}(\nabla_z \mathcal{L}_{render}(z_k))$ [S1, unofficial].
    *   Outer loss per view: $\ell_{out} = 0.8 L_1 + 0.2 L_{DSSIM} + 0.5 L_{LPIPS}$ [S1, unofficial].
    *   Multi-step objective:
        $$ \mathcal{L}_{II} = \sum_{k=1}^{K_r} 0.9^{K_r-k} \mathcal{L}_k^{render} + 0.1 \mathcal{L}_{opacity} $$
        where $\mathcal{L}_{opacity} = \frac{1}{M} \sum_{m=1}^M [\text{logit}_\epsilon(\alpha_{m, K_r}) - \text{logit}_\epsilon(\alpha_{m, 0})]^2$ with $\epsilon=10^{-6}$ [S1, unofficial].

4.  **Chunked Execution:**
    *   Reuse global KNN graph. Sort points along longest voxel-coordinate axis.
    *   Expand core sets of at most 200,000 points with four-hop neighborhoods [S1, unofficial].
    *   Recursively bisect chunks exceeding 300,000 points by point count along longest axes, recomputing neighborhoods [S1, unofficial].
    *   Retain only core-point outputs; use gradient checkpointing [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class InfiniSplatV2:
    def __init__(self, num_gaussians, hidden_dim=16, num_knn=8, num_transformer_blocks=4):
        # Stage I components
        self.phi_geo = load_da3_frozen() # Frozen geometry branch
        self.phi_attr = load_da3_trainable() # Trainable attribute branch (last 10 blocks)
        self.gaussian_head = DPTDecoder() # Maps features to attributes
        
        # Stage II components
        self.feature_proj = nn.Conv2d(in_channels=..., out_channels=8, kernel_size=1) # 1x1 conv
        self.hidden_init = nn.Linear(..., hidden_dim) # Linear projection for hidden state
        self.refiner = KNNRefiner(hidden_dim=64, num_blocks=num_transformer_blocks, num_knn=num_knn)
        
        # Parameters
        self.voxel_side = 0.004
        self.alpha_thresh = 0.01
        self.tau_abs = 0.03
        self.tau_rel = 0.01
        self.eps = 1e-6
        
    def stage_i_init(self, images, gt_poses=None):
        # 1. Feature Extraction
        # If gt_poses is None, use predicted poses for alignment, but GT for construction
        if gt_poses is not None:
            # Use GT poses for construction to avoid pose errors
            predicted_depth, _ = self.phi_geo(images, None) # Q_C is optional
            # Use GT poses for alignment calculation
            # ... (Umeyama transform logic to get s_C) ...
            aligned_poses = scale_gt_poses(gt_poses, s_C)
        else:
            # Pose-free: reference cameras only for eval frame
            predicted_depth, _ = self.phi_geo(images, None)
            # ... (scale alignment logic) ...
            aligned_poses = get_reference_poses()
            
        features = self.phi_attr(images, None)
        
        # 2. Point Construction
        # Back-project depth to 3D points using aligned poses and intrinsics
        centers = back_project_depth(predicted_depth, aligned_poses, intrinsics)
        
        # Decode attributes
        raw_attrs = self.gaussian_head(features)
        
        # 3. Voxel Merging and Pruning
        gaussians = merge_voxels(centers, raw_attrs, side_length=self.voxel_side)
        gaussians = prune_opacity(gaussians, threshold=self.alpha_thresh)
        
        return gaussians

    def stage_i_loss(self, gaussians, images, target_images, target_poses):
        # Render into input and target views
        input_views = render(gaussians, input_poses)
        target_views = render(gaussians, target_poses)
        
        # Construct masks
        masks = construct_visibility_masks(target_images, gaussians, target_poses, 
                                           tau_abs=self.tau_abs, tau_rel=self.tau_rel)
        
        # Compute LPIPS
        lpips_map = compute_lpips_spatial(masks * input_views, masks * target_images)
        
        # Compute L1
        l1_map = torch.abs(input_views - target_images)
        
        # Aggregate
        loss = aggregate_loss(l1_map, lpips_map, masks)
        return loss

    def stage_i_optimize(self, images, target_images, target_poses, optimizer, steps=25000):
        for step in range(steps):
            gaussians = self.stage_i_init(images, target_poses)
            loss = self.stage_i_loss(gaussians, images, target_images, target_poses)
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            
    def stage_i_refine(self, gaussians, images, target_poses, optimizer, steps=5000):
        # Freeze gaussians and phi_attr
        for _ in range(steps):
            # Render and compute feedback loss
            rendered = render(gaussians, input_poses)
            fb_loss = compute_feedback_loss(rendered, images) # 0.8 L1 + 0.2 LDSSIM
            
            # Compute gradients
            grads = torch.autograd.grad(fb_loss, gaussians.attributes, retain_graph=True)[0]
            grads = normalize_and_detach(grads)
            
            # Update attributes
            gaussians = update_attributes(gaussians, grads)
            
            # Compute outer loss for supervision
            outer_loss = compute_outer_loss(gaussians, images, target_images, target_poses)
            outer_loss.backward()
            optimizer.step()
            optimizer.zero_grad()

    def stage_i_refine_multi_step(self, gaussians, images, target_images, target_poses, optimizer, steps=5000):
        for step in range(steps):
            # Sample Kr
            Kr = random.choice([1, 2, 3])
            
            current_gaussians = gaussians
            total_loss = 0
            
            for k in range(1, Kr + 1):
                # Render and compute feedback
                rendered = render(current_gaussians, input_poses)
                fb_loss = compute_feedback_loss(rendered, images)
                
                # Get gradients
                grads = torch.autograd.grad(fb_loss, current_gaussians.attributes, retain_graph=True)[0]
                grads = normalize_and_detach(grads)
                
                # Update attributes
                current_gaussians = update_attributes(current_gaussians, grads)
                
                # Compute surrogate loss
                # First-order surrogate
                x_k = current_gaussians.attributes
                z_k = x_k.detach() # sg(x_k)
                render_grads = torch.autograd.grad(render(z_k, input_poses), z_k, retain_graph=True)[0]
                r_k = render_grads.detach() # sg(grad)
                
                # L_proxy = sg(L) + <x - sg(x), r>
                # Gradient of L_proxy w.r.t trainable params theta (refiner)
                # dL_proxy/dx = r_k
                # So we pass r_k to the refiner as gradient feedback
                
                # Compute outer loss
                outer_loss = compute_outer_loss(current_gaussians, images, target_images, target_poses)
                
                # Weighted sum
                weight = 0.9 ** (Kr - k)
                total_loss += weight * outer_loss
            
            # Opacity regularization
            opacity_loss = compute_opacity_reg(gaussians, current_gaussians)
            total_loss += 0.1 * opacity_loss
            
            total_loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            
            # Update main gaussians
            gaussians = current_gaussians

def compute_feedback_loss(rendered, target):
    l1 = torch.mean(torch.abs(rendered - target))
    ssim = compute_ssim(rendered, target)
    ldssim = (1 - ssim) / 2
    return 0.8 * l1 + 0.2 * ldssim

def compute_outer_loss(gaussians, images, target_images, target_poses):
    rendered = render(gaussians, target_poses)
    l1 = torch.mean(torch.abs(rendered - target_images))
    ssim = compute_ssim(rendered, target_images)
    ldssim = (1 - ssim) / 2
    lpips = compute_lpips(rendered, target_images)
    return 0.8 * l1 + 0.2 * ldssim + 0.5 * lpips

def compute_opacity_reg(g0, gk):
    # logit_epsilon(alpha) = log(alpha / (1 - alpha + eps))
    alpha_0 = g0.attributes.alpha
    alpha_k = gk.attributes.alpha
    logit_0 = torch.log(alpha_0 / (1 - alpha_0 + 1e-6))
    logit_k = torch.log(alpha_k / (1 - alpha_k + 1e-6))
    return torch.mean((logit_k - logit_0) ** 2)

def update_attributes(gaussians, grads):
    # grads are normalized and detached
    # Scales: s_new = s_old * exp(delta_log_s)
    # delta_log_s is derived from grads
    # Opacity, Rotation, RGB: a * tanh(raw)
    # Hidden state: 0.1 * tanh(raw)
    # ... implementation details ...
    pass
```

## Operating characteristics
*   **Time Complexity:** Stage I is dominated by the forward pass of the DA3 backbone and DPT decoder. Stage II involves iterative KNN queries and transformer blocks, with chunked execution to manage memory. The global KNN graph is reused [S1, unofficial].
*   **Memory Complexity:** Depends on the number of Gaussians $M$. Chunked execution limits peak memory by processing subsets of points (core sets up to 200,000, chunks bisected if >300,000) [S1, unofficial].
*   **Performance:** Achieves state-of-the-art mean LPIPS and DISTS on six datasets in both pose-conditioned and pose-free settings. In the pose-conditioned setting, reduces LPIPS and DISTS by 25.8% and 32.9% respectively compared to the strongest baseline [S1, unofficial].
*   **Strengths:** Decoupled geometry and attribute prediction prevents geometric degradation. GT pose alignment avoids pose errors under large-angle supervision. Large-angle target views constrain Gaussian shapes better than local views.
*   **Limitations:** Requires GT camera poses for training alignment (though inference can be pose-free). Joint optimization of geometry and attributes is avoided, which may limit flexibility in some scenarios [S1, unofficial].

## Background
InfiniSplat v2 was introduced by Lin et al. in 2026 as an improvement over DA3-GS and InfiniSplat [S1, unofficial]. It addresses limitations in Gaussian attribute learning, specifically flying-point artifacts and pose errors under large viewpoint changes [S1, unofficial].

## Sources
[S1] InfiniSplat v2: Analyzing and Improving Feed-Forward 3DGS from Any Views, https://arxiv.org/abs/2610.20341, fetched 2026-10-29 (unofficial)
