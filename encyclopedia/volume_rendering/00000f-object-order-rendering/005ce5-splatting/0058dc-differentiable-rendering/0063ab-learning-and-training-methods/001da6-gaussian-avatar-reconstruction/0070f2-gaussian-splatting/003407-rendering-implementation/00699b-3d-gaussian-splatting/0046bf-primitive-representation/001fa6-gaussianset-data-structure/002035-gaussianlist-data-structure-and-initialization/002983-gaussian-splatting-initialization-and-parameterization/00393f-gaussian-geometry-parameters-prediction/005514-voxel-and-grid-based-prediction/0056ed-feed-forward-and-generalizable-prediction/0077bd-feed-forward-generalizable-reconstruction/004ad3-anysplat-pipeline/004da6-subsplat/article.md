# SubSplat

Also called: Sub-pixel Gaussian Reparameterization, Sub-pixel Gaussian Reparameterizer

## Overview
SubSplat is a method for high-resolution novel-view synthesis that resolves the trade-off between network computational cost and rendering fidelity in pixel-aligned 3D Gaussian Splatting [S1]. It solves the problem of insufficient Gaussian density and artifacts when using low-resolution backbone inputs by introducing a Sub-pixel Gaussian Reparameterizer (SPGR) [S1]. Unlike prior methods limited by fixed grids or image-space upsamplers, SubSplat subdivides each grid-anchored primary Gaussian into multiple fine-grained sub-pixel primitives, restoring structural density directly from low-resolution features while maintaining stable network latency [S1].

## How it works
SubSplat builds on the [AnySplat Pipeline](12847-anysplat-pipeline.md) by refining the Gaussian generation stage [S1]. It takes low-resolution multi-view features and primary Gaussian anchors as input, and outputs a set of sub-pixel Gaussians suitable for high-resolution rendering [S1].

### Inputs and Outputs
*   **Inputs:**
    *   $N$ primary Gaussians $\mathcal{A} = \{\mathbf{a}_p\}_{p=1}^N$, where $\mathbf{a}_p = [\boldsymbol{\mu}_p, \mathbf{s}_p, q_p, \alpha_p] \in \mathbb{R}^{11}$ [S1].
    *   Spherical Harmonic (SH) colors $\mathcal{H} = \{\mathbf{h}_p \in \mathbb{R}^{C \times D_{\mathrm{sh}}}\}_{p=1}^N$ [S1]. Default: $C=3, D_{\mathrm{sh}}=25$ (degree-4 SH) [S1].
    *   Multi-view feature maps $\mathcal{F} = \{\Phi_{v,\ell}\}_{v=1..V, \ell=1..L}$, where $\Phi_{v,\ell} \in \mathbb{R}^{H_\ell \times W_\ell \times D_\ell}$ [S1].
    *   Fixed camera intrinsics and extrinsics [S1].
*   **Outputs:**
    *   $K$ sub-pixel Gaussians per primary Gaussian, defined by position $\boldsymbol{\mu}_{p,k}$, scale $\mathbf{s}_{p,k}$, quaternion $q_{p,k}$, opacity $\alpha_k$, and SH color $\mathbf{h}_{p,k}$ for $k=1 \dots K$ [S1].
    *   $K$ is the squared ratio of target to input resolution (e.g., $K=4$ for 2x, $K=16$ for 4x) [S1].

### Data Structures
*   **Primary Gaussian State:** $\mathbf{a}_p$ contains world-space position $\boldsymbol{\mu}_p$ (3), per-axis scales $\mathbf{s}_p$ (3), unit quaternion $q_p$ (4), and opacity $\alpha_p$ (1) [S1].
*   **Embeddings:** Geometry and appearance queries are aligned to a shared embedding dimension $E=128$ [S1].
*   **Feature Maps:** Multi-view features $\Phi_{v,\ell}$ are sampled via the deformable attention operator using bilinear weights $g(a, b) = \max(0, 1-|a-b|)$ [S1].

### Steps

1.  **Initialization:** Define $N$ primary Gaussians $\mathcal{A}$ and SH colors $\mathcal{H}$ from the backbone output. Camera metadata is kept separate [S1].
2.  **Feature Aggregation:**
    *   **Geometry Query:** Encode primary Gaussian attributes into a 128-dimensional embedding using per-attribute MLPs $\phi_x, \phi_s, \phi_q, \phi_\alpha$, then project with linear $W_g$:
        $$ \mathbf{z}^g_p = W_g(\phi_x(\boldsymbol{\mu}_p) + \phi_s(\mathbf{s}_p) + \phi_q(q_p) + \phi_\alpha(\alpha_p)) $$
        [S1]
    *   **Geometry Aggregation:** Perform 3-stage deformable attention [S1]. Initialize $\mathbf{f}^{g,0}_p = \mathbf{0}$ [S1]. For $t=0,1,2$:
        $$ \mathbf{f}^{g,t+1}_p = \mathrm{DA}(\mathcal{F}; \mathbf{z}^g_p, \mathbf{f}^{g,t}_p, \mathcal{P}(\boldsymbol{\mu}_p), \mathbf{a}_p) $$
        Final geometry descriptor $\mathbf{f}^g_p = \mathbf{f}^{g,3}_p$ [S1].
    *   **Appearance Query:** Encode flattened SH coefficients $\mathrm{vec}(\mathbf{h}_p)$ into a 128-dimensional embedding:
        $$ \mathbf{z}^a_p = \phi_c(\mathrm{vec}(\mathbf{h}_p)) $$
        [S1]
    *   **Appearance Aggregation:** Perform 1-stage deformable attention conditioned on the geometry descriptor:
        $$ \mathbf{f}^a_p = \mathrm{DA}(\mathcal{F}; \mathbf{z}^a_p, \mathbf{f}^g_p, \mathcal{P}(\boldsymbol{\mu}_p), \mathbf{a}_p) $$
        [S1]
3.  **Primary View Selection:** For each anchor $\boldsymbol{\mu}_p$, project into all cameras [S1]. Select the primary view $v^*$ with the smallest positive depth $z_v > 0$ [S1].
4.  **Geometry Reparameterization:**
    *   Predict offsets using geometry head $g_\theta: \mathbb{R}^E \to \mathbb{R}^{K \times 9}$:
        $$ [\Delta u_k, \Delta v_k, \Delta z_k, \Delta \log \mathbf{s}_k, \Delta \mathbf{r}_k]_{k=1}^K = g_\theta(\mathbf{f}^g_p) $$
        [S1]
    *   **Position:** Back-project sub-pixel offsets $(\Delta u_k, \Delta v_k)$ using selected depth $z_{\mathrm{sel}}$ and focal lengths $f_x, f_y$ [S1]. Bound depth residual via $\tanh$:
        $$ \Delta \hat{z}_k = \tanh(\Delta z_k) $$
        $$ \boldsymbol{\mu}_{p,k} = \boldsymbol{\mu}_p + R_{c2w} \begin{bmatrix} z_{\mathrm{sel}}\Delta u_k / f_x \\ z_{\mathrm{sel}}\Delta v_k / f_y \\ 0 \end{bmatrix} + \Delta \hat{z}_k R_{c2w} \begin{bmatrix} 0 \\ 0 \\ 1 \end{bmatrix} $$
        where $R_{c2w}$ is the camera-to-world rotation matrix [S1].
    *   **Scale:** Refine scales element-wise:
        $$ \mathbf{s}_{p,k} = \mathbf{s}_p \odot \exp(\Delta \log \mathbf{s}_k) $$
        [S1]
    *   **Rotation:** Convert axis-angle residual $\Delta \mathbf{r}_k$ to quaternion $q(\Delta \mathbf{r}_k)$, compose, and normalize:
        $$ q_{p,k} = \mathrm{norm}(q_p \otimes q(\Delta \mathbf{r}_k)) $$
        [S1]
    *   **Covariance:** Compute covariance matrix:
        $$ \Sigma_{p,k} = R(q_{p,k}) \mathrm{diag}(\mathbf{s}_{p,k}^2) R(q_{p,k})^\top $$
        [S1]
5.  **Appearance Reparameterization:**
    *   Predict opacity logits $\ell_k$ and color gain $\gamma_k$ using appearance head $h_\phi: \mathbb{R}^E \to \mathbb{R}^{K \times 2}$:
        $$ [\ell_k, \gamma_k]_{k=1}^K = h_\phi(\mathbf{f}^a_p) $$
        [S1]
    *   **Projected Area:** Compute pinhole Jacobian $B$ at $\boldsymbol{\mu}_{p,k}$ in camera coordinates (where $x_c, y_c, z_c$ are camera-frame coords):
        $$ B = \begin{bmatrix} f_x/z_c & 0 & -f_x x_c/z_c^2 \\ 0 & f_y/z_c & -f_y y_c/z_c^2 \end{bmatrix} $$
        Compute world-space projection Jacobian $J_{v^*}(\boldsymbol{\mu}_{p,k}) = B R^{v^*}_{\mathrm{w2c}}$ [S1]. Estimate projected area:
        $$ A_k \approx \sqrt{\det(J_{v^*}(\boldsymbol{\mu}_{p,k}) \Sigma_{p,k} J_{v^*}(\boldsymbol{\mu}_{p,k})^\top)} $$
        [S1]
    *   **Opacity Allocation:** Compute weights to preserve integrated opacity:
        $$ w_k = \mathrm{softmax}(\ell_k), \quad \tilde{w}_k = \frac{w_k}{A_k + \varepsilon} $$
        $$ \alpha_k = \alpha_p \cdot \frac{\tilde{w}_k}{\sum_j \tilde{w}_j} $$
        [S1]
    *   **Color Modulation:** Scale SH coefficients:
        $$ \mathbf{h}_{p,k} = \gamma_k \mathbf{h}_p $$
        [S1]
    *   **Anti-aliasing:** If $A_k < A_{\mathrm{min}}$, adjust in-plane scales to ensure minimum area, preventing flickering [S1].
6.  **Training:** Minimize loss $\mathcal{L} = \mathcal{L}_{\ell_2} + \lambda \mathcal{L}_{\mathrm{LPIPS}}$ with $\lambda = 0.05$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def axis_angle_to_quaternion(axis_angle):
    angle = torch.norm(axis_angle, dim=-1, keepdim=True)
    axis = axis_angle / (angle + 1e-8)
    half_angle = angle / 2
    return torch.cat([torch.cos(half_angle), torch.sin(half_angle) * axis], dim=-1)

def quaternion_multiply(q1, q2):
    w1, x1, y1, z1 = q1[..., 0], q1[..., 1], q1[..., 2], q1[..., 3]
    w2, x2, y2, z2 = q2[..., 0], q2[..., 1], q2[..., 2], q2[..., 3]
    return torch.stack([w1*w2 - x1*x2 - y1*y2 - z1*z2,
                        w1*x2 + x1*w2 + y1*z2 - z1*y2,
                        w1*y2 - x1*z2 + y1*w2 + z1*x2,
                        w1*z2 + x1*y2 - y1*x2 + z1*w2], dim=-1)

def quaternion_normalize(q):
    return q / (torch.norm(q, dim=-1, keepdim=True) + 1e-8)

def quaternion_to_rotation_matrix(q):
    w, x, y, z = q[..., 0], q[..., 1], q[..., 2], q[..., 3]
    R = torch.zeros(*q.shape[:-1], 3, 3, device=q.device)
    R[..., 0, 0] = 1 - 2*(y**2 + z**2)
    R[..., 0, 1] = 2*(x*y - w*z)
    R[..., 0, 2] = 2*(x*z + w*y)
    R[..., 1, 0] = 2*(x*y + w*z)
    R[..., 1, 1] = 1 - 2*(x**2 + z**2)
    R[..., 1, 2] = 2*(y*z - w*x)
    R[..., 2, 0] = 2*(x*z - w*y)
    R[..., 2, 1] = 2*(y*z + w*x)
    R[..., 2, 2] = 1 - 2*(x**2 + y**2)
    return R

def compute_covariance(s, q):
    R = quaternion_to_rotation_matrix(q)
    return R @ torch.diag_embed(s**2) @ R.transpose(-1, -2)

def bilinear_sample(features, coords):
    x, y = coords[..., 0], coords[..., 1]
    x0, y0 = torch.floor(x).long(), torch.floor(y).long()
    x1, y1 = x0 + 1, y0 + 1
    # Triangular kernel g(a, b) = max(0, 1 - |a - b|) per evidence
    w00 = torch.clamp(1 - torch.abs(x - x0), min=0) * torch.clamp(1 - torch.abs(y - y0), min=0)
    w10 = torch.clamp(1 - torch.abs(x - x1), min=0) * torch.clamp(1 - torch.abs(y - y0), min=0)
    w01 = torch.clamp(1 - torch.abs(x - x0), min=0) * torch.clamp(1 - torch.abs(y - y1), min=0)
    w11 = torch.clamp(1 - torch.abs(x - x1), min=0) * torch.clamp(1 - torch.abs(y - y1), min=0)
    return (w00.unsqueeze(-1).unsqueeze(-1) * features[:, :, y0, x0] +
            w10.unsqueeze(-1).unsqueeze(-1) * features[:, :, y0, x1] +
            w01.unsqueeze(-1).unsqueeze(-1) * features[:, :, y1, x0] +
            w11.unsqueeze(-1).unsqueeze(-1) * features[:, :, y1, x1])

class SubSplatSPGR:
    def __init__(self, E=128, C=3, D_sh=25, K=4):
        self.E = E
        self.K = K
        self.phi_x = nn.Linear(3, E)
        self.phi_s = nn.Linear(3, E)
        self.phi_q = nn.Linear(4, E)
        self.phi_alpha = nn.Linear(1, E)
        self.W_g = nn.Linear(E, E)
        self.phi_c = nn.Linear(C * D_sh, E)
        self.g_theta = nn.Linear(E, K * 9)
        self.h_phi = nn.Linear(E, K * 2)

    def encode_geometry(self, mu, s, q, alpha):
        z_sum = self.phi_x(mu) + self.phi_s(s) + self.phi_q(q) + self.phi_alpha(alpha)
        return self.W_g(z_sum)

    def encode_appearance(self, h):
        return self.phi_c(h.view(h.size(0), -1))

    def deformable_aggregation(self, features, z_query, f_prev, mu, extrinsics, fx, fy, num_stages):
        f_curr = f_prev
        for _ in range(num_stages):
            R_w2c = extrinsics[:, :3, :3]
            t = extrinsics[:, :3, 3]
            mu_cam = torch.bmm(R_w2c, (mu - t).unsqueeze(-1)).squeeze(-1)
            x = mu_cam[..., 0] * fx / mu_cam[..., 2]
            y = mu_cam[..., 1] * fy / mu_cam[..., 2]
            phi_feat = bilinear_sample(features, torch.stack([x, y], dim=-1))
            attn = torch.softmax((f_curr + z_query) * phi_feat, dim=-1)
            f_curr = (attn * phi_feat).sum(dim=-1)
        return f_curr

    def geometry_reparam(self, f_g, mu, s, q, z_sel, fx, fy, R_c2w):
        offsets = self.g_theta(f_g).view(-1, self.K, 9)
        du, dv, dz = offsets[:, :, 0], offsets[:, :, 1], offsets[:, :, 2]
        dlog_s, dr = offsets[:, :, 3:6], offsets[:, :, 6:9]
        
        dz_hat = torch.tanh(dz)
        dx = z_sel * du / fx
        dy = z_sel * dv / fy
        
        offset_vec = torch.stack([dx, dy, torch.zeros_like(dx)], dim=-1)
        offset_world = torch.matmul(offset_vec, R_c2w.transpose(1, 2))
        depth_vec = torch.stack([torch.zeros_like(dz_hat), torch.zeros_like(dz_hat), dz_hat], dim=-1)
        depth_world = torch.matmul(depth_vec, R_c2w.transpose(1, 2))
        
        mu_p = mu.unsqueeze(1).expand(-1, self.K, -1)
        mu_new = mu_p + offset_world + depth_world
        
        s_new = s.unsqueeze(1) * torch.exp(dlog_s)
        q_res = axis_angle_to_quaternion(dr)
        q_new = quaternion_normalize(quaternion_multiply(q.unsqueeze(1), q_res))
        
        return mu_new, s_new, q_new

    def appearance_reparam(self, f_a, mu, s, q, alpha, h, R_w2c, fx, fy):
        preds = self.h_phi(f_a).view(-1, self.K, 2)
        l, gamma = preds[:, :, 0], preds[:, :, 1]
        
        covs_new = compute_covariance(s.unsqueeze(1), q.unsqueeze(1))
        
        mu_cam = torch.bmm(R_w2c, (mu.unsqueeze(1) - 0).unsqueeze(-1)).squeeze(-1)
        xc, yc, zc = mu_cam[..., 0], mu_cam[..., 1], mu_cam[..., 2]
        B = torch.stack([fx/zc, torch.zeros_like(zc), -fx*xc/(zc**2),
                         torch.zeros_like(zc), fy/zc, -fy*yc/(zc**2)], dim=-1).view(-1, self.K, 2, 3)
        J = torch.matmul(B, R_w2c)
        areas = torch.sqrt(torch.det(J @ covs_new @ J.transpose(-1, -2)).clamp(min=1e-6))
        
        w = torch.softmax(l, dim=-1)
        eps = 1e-6
        w_tilde = w / (areas + eps)
        alpha_new = alpha.unsqueeze(1) * (w_tilde / w_tilde.sum(dim=-1, keepdim=True))
        
        h_new = gamma.unsqueeze(-1).unsqueeze(-1) * h.unsqueeze(1)
        return alpha_new, h_new

    def forward(self, mu, s, q, alpha, h, features, extrinsics, intrinsics):
        z_g = self.encode_geometry(mu, s, q, alpha)
        z_a = self.encode_appearance(h)
        
        fx, fy = intrinsics[..., 0, 0], intrinsics[..., 1, 1]
        
        f_g = self.deformable_aggregation(features, z_g, torch.zeros_like(z_g), mu, extrinsics, fx, fy, 3)
        f_a = self.deformable_aggregation(features, z_a, f_g, mu, extrinsics, fx, fy, 1)
        
        mu_cam = torch.bmm(extrinsics[:, :3, :3], (mu - extrinsics[:, :3, 3:4]).squeeze(-1)).squeeze(-1)
        depths = mu_cam[..., 2]
        valid_depths = torch.where(depths > 0, depths, torch.inf)
        v_star = torch.argmin(valid_depths, dim=-1)
        
        B_idx = torch.arange(mu.size(0))
        R_c2w = extrinsics[B_idx, v_star, :3, :3]
        z_sel = depths[B_idx, v_star]
        
        mu_new, s_new, q_new = self.geometry_reparam(f_g, mu, s, q, z_sel, fx, fy, R_c2w)
        alpha_new, h_new = self.appearance_reparam(f_a, mu, s, q, alpha, h, R_c2w.transpose(1, 2), fx, fy)
        
        return mu_new, s_new, q_new, alpha_new, h_new
```

### Parameters
*   **K:** Number of sub-pixel primitives per primary Gaussian. $K = (\text{target\_res} / \text{input\_res})^2$. Typical values: 4 (2x), 16 (4x) [S1].
*   **$\lambda$:** LPIPS loss weight. Default: 0.05 [S1].
*   **C, $D_{\mathrm{sh}}$:** SH dimensions [S1]. Default: $C=3, D_{\mathrm{sh}}=25$ [S1].
*   **E:** Embedding dimension. Default: 128 [S1].

### Edge Cases and Pitfalls
*   **Depth Instability:** Unbounded depth residuals cause instability [S1]. Mitigated by bounding $\Delta z_k$ via $\tanh$ [S1].
*   **Aliasing/Flickering:** Extremely small primitives ($A_k < A_{\mathrm{min}}$) cause flickering [S1]. Mitigated by enforcing minimum area $A_{\mathrm{min}}$ and adjusting in-plane scales [S1].
*   **Opacity Preservation:** Sub-pixel primitives must sum to primary opacity [S1]. Mitigated by renormalization $\alpha_k = \alpha_p \cdot (\tilde{w}_k / \sum_j \tilde{w}_j)$ [S1].

## Operating characteristics
*   **Time and Memory Complexity:** SubSplat maintains stable backbone latency by using low-resolution inputs [S1]. The reparameterization stage adds negligible computational overhead compared to the backbone [S1]. It achieves over 3x latency reduction compared to full-resolution baselines [S1].
*   **Measured Performance:** Evaluated on RealEstate10K and ACID datasets, reconstructing from 256x256 input and rendering at 512x512 (2x) and 1024x1024 (4x) [S1].
*   **Strengths:** High-fidelity reconstruction at extended scales; superior efficiency; content-aware density scaling; mitigates quadratic cost of high-resolution inputs [S1].
*   **Limitations:** Relies on the quality of the backbone features; requires deformable attention which can be complex to implement efficiently [S1].
*   **When to use:** When high-resolution output is required from sparse, low-resolution inputs, and inference latency is a constraint [S1].
*   **When not to use:** When per-scene optimization is acceptable and latency is not a concern, or when input resolution is already high enough to support direct high-res Gaussian prediction [S1].

## Background
SubSplat was proposed by Jiun Lee et al. in 2024 [S1]. It addresses the limitations of pixel-aligned Gaussian methods that are constrained by fixed grids or suffer from high computational costs at high resolutions [S1]. It uses the same input structure as MVSplat and incorporates a deformable-attention operator following the formulation used in GaussianFormer and PixelGaussian [S1]. It is used for novel-view synthesis in computer vision, specifically for sparse-view reconstruction [S1].

## Sources
[S1] SubSplat: High-Resolution Pixel-Aligned 3DGS via Sub-pixel Gaussian Reparameterization, https://doi.org/10.1007/978-3-032-37032-7_25, fetched 2026-10-27
