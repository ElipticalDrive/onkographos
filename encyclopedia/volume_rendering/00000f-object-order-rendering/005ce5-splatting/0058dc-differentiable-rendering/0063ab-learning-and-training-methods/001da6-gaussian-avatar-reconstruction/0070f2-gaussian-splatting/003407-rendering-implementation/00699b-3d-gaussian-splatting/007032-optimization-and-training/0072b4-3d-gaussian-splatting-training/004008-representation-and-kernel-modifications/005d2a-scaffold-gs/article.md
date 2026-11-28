# Scaffold-GS

Also called: Scaffold Gaussian Splatting, Structured 3D Gaussians, Structured 3D Gaussians for View-Adaptive Rendering

## Overview
Scaffold-GS is a Gaussian-based scene representation that utilizes anchor points to establish a hierarchical and region-aware 3D scene structure, predicting neural Gaussian attributes on-the-fly based on viewing direction and distance [S4]. It addresses the issues of vanilla 3D Gaussian Splatting (3D-GS) where Gaussians freely drift and split, leading to high storage costs and poor robustness to view changes [S4]. Unlike vanilla 3D-GS, Scaffold-GS exploits scene structure to guide and constrain the distribution of 3D Gaussians while allowing local adaptation to varying viewing angles and distances [S4]. It achieves comparable or superior PSNR/SSIM to 3D-GS with significantly reduced storage (4.4x–10.2x reduction) and real-time rendering speeds (~100 FPS at 1K resolution) [S4].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse Structure-from-Motion (SfM) point cloud $P \in \mathbb{R}^{M \times 3}$, camera poses $\{x_c\}$, and training images [S4].
*   **Output:** Rendered image $C(x')$ at pixel position $x'$, optimized anchor parameters, and MLP weights [S4].

**Data Structures:**
1.  **Anchor Point $v$:** Located at voxel center $x_v \in \mathbb{R}^3$ [S4].
    *   Local context feature: $f_v \in \mathbb{R}^{32}$ [S4].
    *   Scaling factor: $l_v \in \mathbb{R}^3$ [S4].
    *   Learnable offsets: $O_v \in \mathbb{R}^{k \times 3}$, where $k$ is the number of neural Gaussians spawned per anchor [S4].
2.  **Feature Bank:** For each anchor $v$, a multi-resolution feature bank $\{f_v, f_{v\downarrow1}, f_{v\downarrow2}\}$ is maintained, where $\downarrow n$ denotes $f_v$ down-sampled by $2^n$ factors [S4].
3.  **Neural Gaussians:** Spawned dynamically from anchors [S4]. Each neural Gaussian $i$ has position $\mu_i$, opacity $\alpha_i$, color $c_i$, quaternion $q_i$, and scale $s_i$ [S4].

**Parameters:**
*   $k$: Number of neural Gaussians spawned per visible anchor. Default: 10 [S4].
*   $\epsilon$: Voxel size for initial grid construction [S4].
*   $\epsilon_g$: Base voxel size for gradient quantification during growing. Default: Not explicitly given, but thresholds are relative to $\epsilon$ [S4].
*   $\tau_g$: Gradient threshold for anchor growing. Default: $64\epsilon$. For intricate/texture-less scenes: $16\epsilon$ [S4].
*   $\tau_\alpha$: Opacity threshold for rasterization. Only neural Gaussians with $\alpha \ge \tau_\alpha$ are rasterized [S4].
*   $N$: Number of training iterations over which gradients/opacity are averaged for growing/pruning. Default: 100 [S4].
*   $\lambda_{SSIM}$: Weight for SSIM loss. Value: 0.2 [S4].
*   $\lambda_{vol}$: Weight for volume regularization loss. Value: 0.001 [S4].
*   MLP Architecture: All MLPs ($F_\alpha, F_c, F_q, F_s, F_w$) are 2-layer MLPs with ReLU activation. Hidden unit dimensions are all 32 [S4].

**Method Steps:**

1.  **Anchor Initialization:**
    Voxelize the input SfM point cloud $P$ to create voxel centers $V \in \mathbb{R}^{N \times 3}$ using voxel size $\epsilon$:
    $$ V = \{ \lfloor P / \epsilon \rfloor \cdot \epsilon \} $$
    Remove duplicate entries [S4]. Each voxel center $v$ becomes an anchor point equipped with $f_v \in \mathbb{R}^{32}$, $l_v \in \mathbb{R}^3$, and $O_v \in \mathbb{R}^{k \times 3}$ [S4].

2.  **Neural Gaussian Derivation (On-the-fly Prediction):**
    For each visible anchor $v$ within the viewing frustum:
    a.  **Calculate View-Dependent Context:**
        Compute relative distance $\delta_{vc}$ and viewing direction $\vec{d}_{vc}$ between anchor $x_v$ and camera $x_c$:
        $$ \delta_{vc} = \|x_v - x_c\|_2 $$
        $$ \vec{d}_{vc} = \frac{x_v - x_c}{\|x_v - x_c\|_2} $$
        [S4]
    b.  **Feature Blending:**
        Create feature bank $\{f_v, f_{v\downarrow1}, f_{v\downarrow2}\}$ [S4]. Predict view-dependent weights $\{w, w_1, w_2\}$ using MLP $F_w$:
        $$ \{w, w_1, w_2\} = \text{Softmax}(F_w(\delta_{vc}, \vec{d}_{vc})) $$
        Form integrated anchor feature $\hat{f}_v$:
        $$ \hat{f}_v = w \cdot f_v + w_1 \cdot f_{v\downarrow1} + w_2 \cdot f_{v\downarrow2} $$
        [S4]
    c.  **Spawn Neural Gaussians:**
        Spawn $k$ neural Gaussians from anchor $x_v$ [S4]. Calculate positions $\{\mu_0, \dots, \mu_{k-1}\}$:
        $$ \{\mu_0, \dots, \mu_{k-1}\} = x_v + \{O_0, \dots, O_{k-1}\} \cdot l_v $$
        [S4]
    d.  **Decode Attributes:**
        Decode attributes in one-pass from $\hat{f}_v$, $\delta_{vc}$, and $\vec{d}_{vc}$ via MLPs $F_\alpha, F_c, F_q, F_s$ [S4]. Example for opacity:
        $$ \{\alpha_0, \dots, \alpha_{k-1}\} = F_\alpha(\hat{f}_v, \delta_{vc}, \vec{d}_{vc}) $$
        Colors $\{c_i\}$, quaternions $\{q_i\}$, and scales $\{s_i\}$ are derived similarly [S4].
    e.  **Filtering:**
        Filter out neural Gaussians where $\alpha_i < \tau_\alpha$ [S4].

3.  **Rendering:**
    Project 3D Gaussians to 2D Gaussians $G'_i(x')$ [S4]. Sort by depth. Compute color via alpha-blending:
    $$ C(x') = \sum_{i \in N} c_i \sigma_i \prod_{j=1}^{i-1} (1 - \sigma_j) $$
    where $\sigma_i = \alpha_i G'_i(x')$ [S4].

4.  **Anchor Refinement (Growing and Pruning):**
    Performed every $N$ iterations.
    a.  **Growing Operation:**
        Quantize neural Gaussians into multi-resolution voxels of size $\epsilon_g^{(m)} = \epsilon_g / 4^{m-1}$ for levels $m \in \{1, 2, 3\}$ [S4]. Compute averaged gradients $\nabla_g$ over $N$ iterations [S4]. If $\nabla_g > \tau_g^{(m)} = \tau_g \cdot 2^{m-1}$, deploy a new anchor at the voxel center if none exists [S4]. Apply random elimination to candidates to regulate addition [S4].
    b.  **Pruning Operation:**
        Accumulate opacity values of neural Gaussians associated with an anchor over $N$ iterations [S4]. If accumulated opacity $< 0.5$, remove the anchor [S4].

5.  **Loss Function:**
    Optimize learnable parameters and MLPs using:
    $$ L = L_1 + \lambda_{SSIM} L_{SSIM} + \lambda_{vol} L_{vol} $$
    [S4]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def softmax(x):
    e_x = np.exp(x - np.max(x))
    return e_x / e_x.sum(axis=-1, keepdims=True)

def downsample_1d(f, factor):
    return f[::factor]

class MLP:
    def __init__(self, in_dim, hidden_dim, out_dim, layers=2):
        self.weights = [np.random.randn(in_dim if i == 0 else hidden_dim, hidden_dim) for i in range(layers)]
        self.biases = [np.zeros(hidden_dim) for _ in range(layers)]
        self.out_weight = np.random.randn(hidden_dim, out_dim)
        self.out_bias = np.zeros(out_dim)
    def forward(self, x):
        h = x
        for w, b in zip(self.weights, self.biases):
            h = np.maximum(0, h @ w + b)
        return h @ self.out_weight + self.out_bias

class Anchor:
    def __init__(self, x_v, k=10):
        self.x_v = x_v
        self.f_v = np.zeros(32)
        self.l_v = np.ones(3)
        self.O_v = np.random.randn(k, 3)
        self.gradients = np.zeros(k)
        self.opacity_history = np.zeros(100)

def scaffold_gs_training_loop(anchors, cameras, gt_images, 
                              k=10, epsilon_g=0.1, tau_g=64.0, tau_alpha=0.01, N=100,
                              lambda_ssim=0.2, lambda_vol=0.001):
    # Initialize MLPs (2-layer, hidden dim 32, ReLU)
    F_w = MLP(4, 32, 3)
    F_alpha = MLP(39, 32, k)
    F_c = MLP(39, 32, k * 3)
    F_q = MLP(39, 32, k * 4)
    F_s = MLP(39, 32, k * 3)
    
    optimizer = SGD(params=anchors + [F_w, F_alpha, F_c, F_q, F_s], lr=0.001)
    
    for iteration in range(30000):
        for camera, gt_img in zip(cameras, gt_images):
            gaussians = []
            for v in anchors:
                # [G] Standard view frustum culling check
                if not is_in_frustum(v.x_v, camera):
                    continue
                    
                delta_vc = np.linalg.norm(v.x_v - camera.x_c)
                d_vc = (v.x_v - camera.x_c) / delta_vc
                
                f_v_down1 = downsample_1d(v.f_v, 2)
                f_v_down2 = downsample_1d(v.f_v, 4)
                
                weights = softmax(F_w.forward(np.concatenate([delta_vc, d_vc])))
                f_hat_v = weights[0]*v.f_v + weights[1]*f_v_down1 + weights[2]*f_v_down2
                
                positions = v.x_v + v.O_v * v.l_v
                
                alphas = F_alpha.forward(np.concatenate([f_hat_v, delta_vc, d_vc]))
                colors = F_c.forward(np.concatenate([f_hat_v, delta_vc, d_vc]))
                quats = F_q.forward(np.concatenate([f_hat_v, delta_vc, d_vc]))
                scales = F_s.forward(np.concatenate([f_hat_v, delta_vc, d_vc]))
                
                for i in range(k):
                    if alphas[i] >= tau_alpha:
                        gaussians.append({'mu': positions[i], 'alpha': alphas[i], 
                                          'c': colors[i], 'q': quats[i], 's': scales[i]})
                        
            # [G] Tile-based rasterization and alpha blending of 2D Gaussians
            rendered_img = rasterize_and_blend(gaussians, camera)
            
            loss_l1 = np.mean(np.abs(rendered_img - gt_img))
            loss_ssim = compute_ssim(rendered_img, gt_img)
            loss_vol = compute_volume_reg(anchors)
            
            loss = loss_l1 + lambda_ssim * loss_ssim + lambda_vol * loss_vol
            optimizer.step(loss)
            
            # Track gradients and opacity for refinement
            for i in range(k):
                v.gradients[i] = alphas[i] # Placeholder for actual gradient accumulation
                v.opacity_history[iteration % 100] = alphas[i]
            
            if iteration % N == 0:
                # Growing
                voxel_grads = {}
                for v in anchors:
                    for i in range(k):
                        grad = v.gradients[i]
                        voxel_id = tuple(np.floor((v.x_v + v.O_v[i]*v.l_v) / epsilon_g) * epsilon_g)
                        voxel_grads[voxel_id] = voxel_grads.get(voxel_id, 0) + grad
                        
                for voxel_id, grad in voxel_grads.items():
                    if grad > tau_g:
                        if not any(np.allclose(a.x_v, voxel_id) for a in anchors):
                            anchors.append(Anchor(voxel_id, k))
                            
                # Pruning
                anchors = [v for v in anchors if np.sum(v.opacity_history) >= 0.5]

class SGD:
    def __init__(self, params, lr):
        self.params = params
        self.lr = lr
    def step(self, loss):
        for p in self.params:
            if hasattr(p, 'grad'):
                p.value -= self.lr * p.grad
```

## Subtopics

### [Scaffold-GS Integration](12513-scaffold-gs-integration.md)
Applies DART and AGE modules to the Scaffold-GS backbone, augmenting the anchor-seeding SfM cloud and gating primitive growth without modifying the rendering path or optimizer.

### [Topology-Aware Anchor Feature Aggregation](17247-topology-aware-anchor-feature-aggregation.md)
Aggregates features across hierarchical anchor levels to improve texture and edge consistency, enforcing topological containment relations.

## Operating characteristics
*   **Time Complexity:** Rendering speed is around 100 FPS at 1K resolution, comparable to original 3D-GS with little computational overhead [S4]. On-the-fly prediction is limited to anchors within the view frustum, and trivial Gaussians are filtered by opacity threshold $\tau_\alpha$ to cut computational load [S4].
*   **Memory Complexity:** Storage requirements are significantly reduced as only anchor points and MLP predictors are stored per scene [S4]. Reduction is 4.4x–10.2x compared to 3D-GS [S4].
*   **Performance:** Achieves comparable PSNR/SSIM to 3D-GS on Mip-NeRF360 but surpasses SOTA on Tanks&Temples and DeepBlending [S4].
*   **Strengths:** Robust to view changes, compact storage, real-time rendering, handles sparse initialization regions well via growing/pruning [S4].
*   **Limitations:** Relies on initial SfM points; performance may vary if SfM initialization is poor (though refinement helps) [S4].

## Background
Scaffold-GS was introduced by Tao Lu, Mulin Yu, Linning Xu, Yuanbo Xiangli, Limin Wang, Dahua Lin, and Bo Dai from Shanghai Artificial Intelligence Laboratory, The Chinese University of Hong Kong, Nanjing University, and Cornell University [S4]. The paper "Scaffold-GS: Structured 3D Gaussians for View-Adaptive Rendering" was published in CVPR 2024 (arXiv:2312.00109v1, Nov 30, 2023) [S4]. It builds on the 3D-GS framework [Parent] to address the lack of structural constraints and view-dependent robustness in vanilla Gaussian splatting [S4].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Compression in 3D Gaussian Splatting: A Survey of Methods, Trends, and Future Directions, https://doi.org/10.1109/TCSVT.2026.3676048, fetched 2026-10-26
[S3] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S4] Scaffold-GS: Structured 3D Gaussians for View-Adaptive Rendering, https://doi.org/10.1109/cvpr52733.2024.01952, fetched 2026-10-26
[S5] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
