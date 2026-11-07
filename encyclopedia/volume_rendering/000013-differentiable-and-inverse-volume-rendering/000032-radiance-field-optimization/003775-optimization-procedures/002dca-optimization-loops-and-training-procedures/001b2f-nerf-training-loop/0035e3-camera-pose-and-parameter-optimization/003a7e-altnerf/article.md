# AltNeRF

Also called: Alternating Depth-Pose Optimization

## Overview
AltNeRF is a framework for creating resilient Neural Radiance Field (NeRF) representations from monocular videos without relying on known camera poses [S1]. It solves the problem of joint optimization in highly non-convex landscapes by alternating between a Scene Prior Module (SPM) that provides robust depth and pose estimates, and a Scene Representation Module (SRM) that refines these estimates using NeRF [S1]. Unlike methods that strictly align rendered depth with priors, AltNeRF uses error-tolerant regularization to handle the imprecision of self-supervised depth estimates [S1].

## How it works
AltNeRF consists of two main components: the Scene Prior Module (SPM) and the Scene Representation Module (SRM), coordinated by an alternating algorithm [S1].

**Inputs and Outputs:**
*   **Input:** A monocular video sequence (frames $I_t$) and camera intrinsics $K$ [S1].
*   **Output:** Refined camera poses $P_i$, fine depth maps $D_i$, and the NeRF representation $f_n$ [S1].

**Data Structures:**
1.  **SPM Depth Network ($f_d$):**
    *   Encoder: ResNet-50 (fully-connected layer removed) [S1].
    *   Decoder: Ten $3 \times 3$ convolutional layers (two per scale) with bilinear up-sampling [S1].
    *   Input: Image $I$. Output: Depth map $D$ [S1].
2.  **SPM Pose Network ($f_p$):**
    *   Architecture: ResNet-34 [S1].
    *   Output: Vector of 9 elements. First 6 elements are continuous rotation representation (axis-angle); last 3 are translation [S1].
    *   Input: Pair of images $(I_t, I_s)$. Output: Relative pose $P_{t \to s}$ [S1].
3.  **SRM NeRF Network ($f_n$):**
    *   Architecture: Eight fully-connected layers with skip connections for density output; one linear layer for color [S1].
    *   Input: 3D point $p \in \mathbb{R}^3$ and viewing direction $d \in \mathbb{R}^3$. Output: Density $\sigma$ and color $c$ [S1].

**Parameters:**
*   $\gamma$: Balance factor for depth regularization. Values: 0.08 (LLFF, CO3D), 0.15 (ScanNet, Captures) [S1].
*   $S_r$: SRM iterations per alternation. Value: 50K [S1].
*   $S_p$: SPM fine-tuning iterations per alternation. Value: 500 [S1].
*   Alternations: Total alternating cycles. Value: 2 [S1].
*   Learning Rates:
    *   SPM Pretrain: $10^{-4}$ [S1].
    *   SPM Fine-tune: $5.0 \times 10^{-5}$ [S1].
    *   SRM NeRF: Initial $10^{-3}$, exponentially decays to $10^{-4}$ [S1].
    *   SRM Pose: Initial $10^{-5}$, linearly increases to $2.0 \times 10^{-3}$ after 1K iterations, then exponentially decays to $10^{-5}$ [S1].

**Formulas:**
*   **Volume Rendering (NeRF):**
    $$ \hat{C}(r) = \int_{t_n}^{t_f} T(t)\sigma(t)c(t) dt, \quad T(t) = \exp\left(-\int_{t_n}^t \sigma(s) ds\right) $$
    where $r = o + td$ is a ray, $t_n, t_f$ are near/far bounds [S1].
*   **Scene Depth:**
    $$ \hat{D}(r) = \int_{t_n}^{t_f} T(t)\sigma(t)t dt $$
    Mean terminating distance of ray $r$ [S1].
*   **Photometric Loss (SMDE):**
    $$ \mathcal{L}_p(I_t, \hat{I}_t) = (1 - \text{SSIM}(I_t, \hat{I}_t)) + \frac{\alpha}{2}(1 - \alpha)\|I_t - \hat{I}_t\|_1 $$
    where $\alpha = 0.85$ [S1].
*   **Edge-aware Smoothness Loss:**
    $$ \mathcal{L}_s = |\partial_x D|e^{-|\partial_x I|} + |\partial_y D|e^{-|\partial_y I|} $$
    where $\partial_x, \partial_y$ are image gradients [S1].
*   **Distilling Loss (SPM):**
    $$ \mathcal{L}_r = 1 - \text{SSIM}(D, D_r) + 0.1 \times (E_r \oplus E / \text{size}(E)) $$
    where $D_r$ is reference depth, $E_r, E$ are occluding boundary maps, $\oplus$ is XOR [S1].
*   **Error-Tolerant Depth Regularization (SRM):**
    $$ \mathcal{L}_e = H\left(\max\left(\frac{\|\hat{D} - D\|_1}{\hat{D} + D} - \epsilon, 0\right)\right) $$
    where $H(\cdot)$ is Huber loss, $\epsilon$ is tolerance [S1].
*   **SRM Total Loss:**
    $$ \mathcal{L}_{sr} = \mathcal{L}_c + \gamma \cdot \mathcal{L}_e $$
    where $\mathcal{L}_c = \|\hat{C}(r) - C(r)\|_2$ is reconstruction loss [S1].
*   **SPM Total Loss:**
    $$ \mathcal{L}_{pt} = \mathcal{L}_p + \mathcal{L}_r + 1.0 \times 10^{-3} \cdot \mathcal{L}_s $$
    [S1].
*   **Multi-view Uncertainty:**
    $$ U_t = \|D_t - D_{s \to t}\|_1 $$
    Used to weight losses via Softmin [S1].

**Algorithm Steps:**

1.  **Pretrain SPM:** Train $f_d$ and $f_p$ on a large dataset (e.g., NYUv2, VOID) using $\mathcal{L}_{pt}$ [S1].
2.  **Initialize Poses:** Set world coordinate system to first frame $I_0$ (identity). Compute initial poses $P_i$ using chain rule: $P_i = P_{i-1 \to i} \times P_{i-1}$, where $P_{i-1 \to i}$ is predicted by $f_p$ [S1].
3.  **Initialize SRM:** Use $P_0$ from SPM as initialization. Define residual pose $\Delta P$. Refined pose $\hat{P} = \Delta P \times P$ [S1].
4.  **Alternating Loop (for $i = 0$ to $N_{alt}-1$):**
    a.  **SRM Step (Scene Representation Module):**
        i.   Set $\Delta P$ learning rate to low value.
        ii.  Run $S_r$ iterations optimizing $f_n$ and $\Delta P$ using $\mathcal{L}_{sr}$ [S1].
        iii. Warmup: Linearly increase $\Delta P$ learning rate to original value after 1K iterations [S1].
        iv.  Output refined depth $\hat{D}_{i+1}$ and pose $\hat{P}_{i+1}$ [S1].
    b.  **SPM Step (Scene Prior Module):**
        i.   Fine-tune $f_d$ for $S_p$ iterations.
        ii.  Replace reference depth $D_r$ in $\mathcal{L}_r$ with $\hat{D}_{i+1}$ [S1].
        iii. Use multi-view consistency check to weight losses with Softmin($U_t$) [S1].
        iv.  Output updated depth $D_{i+1}$ and pose $P_{i+1}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.nn.functional import huber_loss

class AltNeRF:
    def __init__(self, frames, intrinsics_K, num_alternations=2, Sr=50000, Sp=500):
        self.frames = frames
        self.K = intrinsics_K
        self.num_alternations = num_alternations
        self.Sr = Sr
        self.Sp = Sp
        
        # Initialize SPM
        self.f_d = DepthNetwork() # ResNet-50 encoder, Conv decoder
        self.f_p = PoseNetwork()  # ResNet-34
        self.optimizer_spm = optim.Adam(list(self.f_d.parameters()) + list(self.f_p.parameters()), lr=1e-4)
        
        # Initialize SRM
        self.f_n = NeRFNetwork()  # 8 FC layers + skip, 1 linear
        # Pose parameterization: residual delta_P (9 elements: 6 rot, 3 trans)
        self.delta_P = nn.Parameter(torch.zeros(9))
        self.optimizer_srm = optim.Adam([
            {'params': self.f_n.parameters(), 'lr': 1e-3},
            {'params': self.delta_P, 'lr': 1e-5}
        ])
        
        # Hyperparameters
        self.gamma = 0.08 # Default, adjust per dataset
        self.alpha = 0.85
        self.epsilon = 0.0 # Tolerance coefficient, not explicitly given, assume 0 or small value [G]
        
        # Initial Pose Calculation
        self.poses = self._compute_initial_poses()

    def _compute_initial_poses(self):
        # Chain rule: P_i = P_{i-1->i} * P_{i-1}
        # P_0 is Identity
        poses = [torch.eye(4)]
        for i in range(1, len(self.frames)):
            # Predict relative pose between i-1 and i
            rel_pose_vec = self.f_p(self.frames[i-1], self.frames[i])
            rel_pose_mat = vec_to_se3(rel_pose_vec)
            prev_pose = poses[-1]
            poses.append(rel_pose_mat @ prev_pose)
        return poses

    def _render_depth(self, ray_origins, ray_directions, pose):
        # Convert pose to camera space if needed, or use directly
        # Sample points along ray
        t_vals = torch.linspace(t_n, t_f, N_samples)
        points = ray_origins[:, None, :] + ray_directions[:, None, :] * t_vals[None, :, None]
        
        # Query NeRF
        sigma, color = self.f_n(points, ray_directions)
        
        # Volume Rendering for Color
        weights = compute_weights(sigma, t_vals)
        color_rendered = torch.sum(weights[..., None] * color, dim=1)
        
        # Volume Rendering for Depth
        depth_rendered = torch.sum(weights * t_vals, dim=1)
        
        return color_rendered, depth_rendered

    def _compute_uncertainty(self, depth_map, pose):
        # Warp depth from neighboring views
        # Compute L1 difference
        # Return average of 4 smallest differences
        pass # Implementation details for warping omitted, assumed available [G]

    def step_spm(self, depth_prior, pose_prior):
        self.f_d.train()
        self.f_p.train()
        optimizer = self.optimizer_spm
        lr_scheduler = optim.lr_scheduler.ConstantLR(optimizer, factor=1.0) # Use fine-tune LR
        
        for _ in range(self.Sp):
            optimizer.zero_grad()
            
            # Forward pass SPM
            depth_pred = self.f_d(self.frames)
            rel_poses = [self.f_p(self.frames[i], self.frames[i+1]) for i in range(len(self.frames)-1)]
            
            # Compute losses
            # L_p: Photometric loss using depth and pose
            # L_r: Distilling loss using depth_prior
            # L_s: Smoothness loss
            
            loss = compute_spm_loss(depth_pred, rel_poses, self.K, depth_prior)
            loss.backward()
            optimizer.step()
            
        # Update internal SPM state if needed
        return depth_pred, self.poses # Simplified return

    def step_srm(self, depth_prior, pose_prior):
        self.f_n.train()
        self.optimizer_srm.param_groups[1]['lr'] = 1e-5 # Reset to low
        
        for step in range(self.Sr):
            self.optimizer_srm.zero_grad()
            
            # Warmup logic
            if step < 1000:
                # Linearly increase pose LR
                current_pose_lr = 1e-5 + (step / 1000) * (2e-3 - 1e-5)
                self.optimizer_srm.param_groups[1]['lr'] = current_pose_lr
            else:
                # Exponential decay
                self.optimizer_srm.param_groups[1]['lr'] = 2e-3 * (0.1 ** (step / 10000)) # Approximation [G]
                
            # Sample rays
            ray_origins, ray_directions = sample_rays(self.frames, pose_prior)
            
            # Render
            color_rendered, depth_rendered = self._render_depth(ray_origins, ray_directions, pose_prior)
            
            # Compute Loss
            # L_c: Reconstruction loss
            # L_e: Error-tolerant depth regularization
            # Uncertainty weighting
            uncertainty = self._compute_uncertainty(depth_rendered, pose_prior)
            weight = softmin(uncertainty)
            
            loss_c = huber_loss(color_rendered, ground_truth_colors) # Or MSE
            loss_e = huber_loss(depth_rendered, depth_prior) # Simplified, actual formula is complex [G]
            
            loss = loss_c + self.gamma * loss_e
            
            loss.backward()
            self.optimizer_srm.step()
            
        # Compute refined pose
        refined_pose = se3_vec_to_mat(self.delta_P) @ pose_prior
        return depth_rendered, refined_pose

    def train(self):
        # Pretrain SPM
        self.pretrain_spm()
        
        # Alternating Loop
        current_depth_prior = self._get_initial_depth_prior()
        current_pose_prior = self.poses
        
        for i in range(self.num_alternations):
            # SRM Step
            new_depth, new_pose = self.step_srm(current_depth_prior, current_pose_prior)
            
            # SPM Step
            updated_depth, updated_pose = self.step_spm(new_depth, new_pose)
            
            current_depth_prior = updated_depth
            current_pose_prior = updated_pose

    def pretrain_spm(self):
        # Standard SMDE training loop
        pass

def vec_to_se3(vec):
    # Convert 9-vector to 4x4 SE(3) matrix
    # First 6: axis-angle (3) + translation (3)
    # Last 3: translation
    pass

def se3_vec_to_mat(vec):
    # Convert 9-vector to 4x4 SE(3) matrix
    pass

def compute_spm_loss(depth_pred, rel_poses, K, depth_prior):
    # Compute L_p, L_r, L_s
    pass

def compute_weights(sigma, t_vals):
    # Standard NeRF weights
    pass

def softmin(x):
    return torch.softmax(-x, dim=0)
```

## Operating characteristics
*   **Time Complexity:** Training takes 150K-200K total iterations, costing 4.0-6.4 hours on a single RTX 3090 [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Scaling:** The method is designed for monocular videos. Performance scales with the number of frames and alternations [S1].
*   **Strengths:** Resilient to imprecise camera poses and dynamic objects. Uses error-tolerant regularization to handle depth prior inaccuracies [S1].
*   **Limitations:** Relies on the quality of the pre-trained SPM. Highly non-convex optimization requires careful learning rate scheduling (warmup) [S1].
*   **When to use:** When camera poses are unknown and only monocular video is available.
*   **When not to use:** When precise known poses are available (standard NeRF is sufficient).

## Background
AltNeRF was proposed in 2024 by researchers at the Thirty-Eighth AAAI Conference on Artificial Intelligence [S1]. It builds on Self-supervised Monocular Depth Estimation (SMDE) and NeRF [S1]. The method addresses the limitations of existing NeRF approaches that suffer from suboptimal outcomes due to lack of explicit 3D supervision and imprecise camera poses [S1]. It introduces an alternating algorithm to harmonize NeRF outputs with SMDE [S1].

## Sources
[S1] AltNeRF: Learning Robust Neural Radiance Field via Alternating Depth-Pose Optimization, https://doi.org/10.1609/aaai.v38i6.28360, fetched 2026-10-26
