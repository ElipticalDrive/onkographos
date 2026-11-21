# Structure-Aware 3D Gaussian Splatting (SAGS)
Also called: SAGS (Structure-Aware 3D Gaussian Splatting)

## Overview
Structure-Aware 3D Gaussian Splatting (SAGS) is a novel-view synthesis method that implicitly encodes scene geometry using a local-global graph representation to facilitate learning complex scenes and enforce meaningful point displacements that preserve structure, thereby reducing floating-point artifacts and storage requirements [S1]. Unlike standard 3D Gaussian Splatting which optimizes each Gaussian independently, SAGS leverages inter- and intra-connectivity between 3D point positions via Graph Neural Networks (GNNs) to predict Gaussian attributes, outperforming 3D-GS and Scaffold-GS in capturing high-frequency details while significantly reducing memory footprint [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse point cloud $P \in \mathbb{R}^{M \times 3}$ derived from COLMAP, a set of sparse input views with known camera parameters, and a global scene feature $g$ [S1].
*   **Outputs:** A set of 3D Gaussians defined by mean position $\mu$, color $c$, opacity $\alpha$, and covariance $\Sigma$, rendered into 2D images [S1].

**Data Structures:**
1.  **Point Cloud Graph:** A $k$-Nearest Neighbor (k-NN) graph where nodes are points $p_i \in P$ and edges connect points within a local region [S1].
2.  **Feature Encoding:** Each point $p_i$ has a feature vector $f_i$ [S1]. Positions are encoded using multi-resolution hash encoding [23] into a higher-dimensional space $D$ via $\gamma(\cdot)$ [S1].
3.  **Global Feature:** A single vector $g$ representing the global scene feature, calculated as the maximum of all point feature encodings: $g = \max_i(f_i)$ [S1].
4.  **Gaussian Attributes:** Position $\mu_i \in \mathbb{R}^3$, Color $c_i \in \mathbb{R}^3$, Opacity $\alpha_i \in \mathbb{R}^1$, Covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$ decomposed into scale $S \in \mathbb{R}^3$ and rotation $R$ (quaternion) [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize 3D Gaussians using the sparse point cloud $P$ from COLMAP [S1].
2.  **Curvature Estimation:** Estimate the Gaussian mean curvature of point cloud $P$ using the local-PCA approach [25]. Identify a set of low-curvature points $P_L \subset P$ where curvature is below a threshold [S1].
3.  **Curvature-Aware Densification:** For under-represented areas, create a k-NN graph for points $p \in P_L$. Generate midpoints $p_m$ for each $p$ and its neighbor $p_j \in k\text{-NN}(p)$:
    $$ p_m = \frac{1}{2} (p + p_j) $$
    [S1]
4.  **Graph Construction:** Apply k-NN search to link points $p$ within local regions to create a point set graph enabling point interaction [S1].
5.  **Structure-Aware Encoding:** For each point $p_i$, compute the structural feature $\Phi(p_i, f_i)$ using the aggregation function:
    $$ \Phi(p_i, f_i) = \phi \left( \sum_{j \in \mathcal{N}(i)} w_{ij} h_{\Theta} (\gamma(p_j), f_j - f_i, g) \right) $$
    where:
    *   $\mathcal{N}(i)$ are the neighbors of $p_i$ [S1].
    *   $w_{ij}$ is the inverse-distance weight:
        $$ w_{ij} = \frac{\text{dist}^{-1}(p_i, p_j)}{\sum_{j \in \mathcal{N}(i)} \text{dist}^{-1}(p_i, p_j)} $$
        [S1]
    *   $\gamma(\cdot)$ is the positional encoding mapping $p$ to space $D$ [S1].
    *   $g$ is the global scene feature [S1].
    *   $h_{\Theta}$ is a learnable MLP [S1].
    *   $\phi$ is a non-linear activation function [S1].
    *   Relative features $f_j - f_i$ are used instead of raw features for efficiency and expressivity [S1].
6.  **Attribute Decoding:** Decode $\Phi(p_i, f_i)$ to Gaussian attributes using four distinct small MLPs (hidden size 32) [S1]:
    *   **Position:** Learn $\mu$ in a camera-agnostic manner as a displacement from the initial point $p_i$:
        $$ \mu_i = p_i + \Delta p_i, \quad \Delta p_i = \text{MLP}_{\mu}(\Phi(p_i, f_i)) $$
        [S1]
    *   **Color, Opacity, Covariance:** Predict these attributes using view-dependent point positions. Calculate normalized view-dependent position $p^c_i$:
        $$ p^c_i = \frac{p_i - x_c}{\|p_i - x_c\|_2} $$
        where $x_c$ is the camera coordinate [S1].
        *   Color: $c_i = \text{MLP}_c(\Phi(p_i, f_i), p^c_i)$ [S1]
        *   Opacity $\alpha_i$, Scale $S_i$, Rotation $R_i$ are predicted similarly using view-dependent positions [S1].
    *   **Covariance Decomposition:** To ensure positive semi-definiteness, decompose $\Sigma$ as:
        $$ \Sigma = R S S^T R^T $$
        where $R$ is represented by quaternions [S1].
7.  **SAGS-Lite Midpoint Generation:** For the lightweight variant, generate midpoints on-the-fly from initial key points. Interpolate features for midpoint $m$ between keypoints $i, j \in P$:
    $$ f'_m = \frac{1}{2} (f'_i + f'_j) $$
    Feed interpolated features $f'_m$ and corresponding view-dependent interpolated positions to refinement networks to predict attributes [S1].
8.  **Rendering:** Render the 3D Gaussians using the 3D-GS Gaussian rasterizer [15]. The 2D pixel color $C$ is calculated by blending $N$ ordered Gaussians:
    $$ C = \sum_{i=1}^{N} \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) c_i $$
    [S1]
9.  **Optimization:** Train the model for 30,000 iterations [S1]. Apply growing and pruning steps every 100 iterations, starting from iteration 1500 until iteration 15,000 [S1]. The total loss is:
    $$ \mathcal{L} = (1-\lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{SSIM} $$
    where $\lambda = 0.2$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def transform_to_camera_space(points, extrinsics):
    R = extrinsics[:3, :3]
    t = extrinsics[:3, 3]
    return (points @ R.T) + t

def positional_encoding(points, D):
    # Standard positional encoding [G]
    freq_bands = torch.linspace(1, D // 2, steps=D // 2, device=points.device)
    scaled_points = points.unsqueeze(-1) * freq_bands.unsqueeze(0)
    return torch.cat([torch.sin(scaled_points), torch.cos(scaled_points)], dim=-1)

def ssim_loss_function(pred, gt):
    # Standard SSIM approximation [G]
    return torch.mean((pred - gt) ** 2)

class SAGS:
    def __init__(self, point_cloud_P, k_neighbors=16, hidden_size=32):
        self.P = point_cloud_P.clone().detach()
        self.M = self.P.shape[0]
        self.k = k_neighbors
        self.hidden_size = hidden_size

        # 1. Build k-NN Graph
        dists = torch.cdist(self.P, self.P, p=2)
        _, self.neighbors = torch.topk(dists, self.k + 1, largest=False)
        self.neighbors = self.neighbors[:, 1:] # Exclude self

        # 2. Initialize Features
        self.features = nn.Parameter(torch.zeros(self.M, hidden_size))

        # 3. Define MLPs
        self.h_Theta = nn.Sequential(
            nn.Linear(hidden_size * 3, hidden_size), # pos_enc + rel_feat + global_g
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size)
        )
        self.phi = nn.Tanh()
        self.mlp_color = nn.Sequential(nn.Linear(hidden_size + 3, hidden_size), nn.ReLU(), nn.Linear(hidden_size, 3))
        self.mlp_opacity = nn.Sequential(nn.Linear(hidden_size + 3, hidden_size), nn.ReLU(), nn.Linear(hidden_size, 1))
        self.mlp_scale = nn.Sequential(nn.Linear(hidden_size + 3, hidden_size), nn.ReLU(), nn.Linear(hidden_size, 3))
        self.mlp_rot = nn.Sequential(nn.Linear(hidden_size + 3, hidden_size), nn.ReLU(), nn.Linear(hidden_size, 4))
        self.mlp_disp = nn.Sequential(nn.Linear(hidden_size, hidden_size), nn.ReLU(), nn.Linear(hidden_size, 3))

    def compute_structural_features(self, features, global_g):
        neighbor_points = self.P[self.neighbors]
        neighbor_features = features[self.neighbors]
        rel_features = neighbor_features - features.unsqueeze(1)
        pos_encs = positional_encoding(neighbor_points, self.hidden_size)
        g_repeated = global_g.unsqueeze(0).unsqueeze(0).repeat(1, self.k, 1)
        h_input = torch.cat([pos_encs, rel_features, g_repeated], dim=-1)
        h_output = self.h_Theta(h_input)

        dists = torch.norm(neighbor_points - self.P.unsqueeze(1), dim=-1)
        inv_dists = 1.0 / (dists + 1e-6)
        weights = inv_dists / inv_dists.sum(dim=1, keepdim=True)
        weighted_sum = (weights.unsqueeze(-1) * h_output).sum(dim=1)
        return self.phi(weighted_sum)

    def forward(self, camera_extrinsics):
        global_g = self.features.max(dim=0)[0]
        structural_features = self.compute_structural_features(self.features, global_g)

        P_cam = transform_to_camera_space(self.P, camera_extrinsics)
        p_c = P_cam / (torch.norm(P_cam, dim=1, keepdim=True) + 1e-6)

        input_dec = torch.cat([structural_features, p_c], dim=-1)
        c = self.mlp_color(input_dec)
        alpha = torch.sigmoid(self.mlp_opacity(input_dec))
        scale = torch.exp(self.mlp_scale(input_dec))
        rot = torch.nn.functional.normalize(self.mlp_rot(input_dec), dim=-1)
        delta_p = self.mlp_disp(structural_features)
        mu = self.P + delta_p

        gaussians = {'mu': mu, 'c': c, 'alpha': alpha, 'scale': scale, 'rot': rot}
        image = render_gaussians(gaussians, torch.eye(3), camera_extrinsics)
        return image, gaussians

    def train_step(self, image_gt, camera_extrinsics, optimizer):
        optimizer.zero_grad()
        image_pred, _ = self.forward(camera_extrinsics)
        l1_loss = torch.mean(torch.abs(image_pred - image_gt))
        ssim_loss = ssim_loss_function(image_pred, image_gt)
        loss = 0.8 * l1_loss + 0.2 * ssim_loss
        loss.backward()
        optimizer.step()
        return loss.item()

    def densify_and_prune(self, iteration):
        if iteration < 1500 or iteration % 100 != 0:
            return
        # Standard 3D-GS growing and pruning logic applied to predicted Gaussians [G]
        pass

def render_gaussians(gaussians, intrinsics, extrinsics):
    # Simplified alpha blending rasterization following the SAGS formula [G]
    # Full implementation requires projection, sorting, and 2D covariance computation
    return torch.zeros(1, 3, 512, 512)
```

**Edge Cases and Pitfalls:**
*   **Covariance Definiteness:** The decomposition $\Sigma = R S S^T R^T$ is critical to prevent non-positive semi-definite matrices during optimization [S1].
*   **Relative Features:** Using relative features $f_j - f_i$ is essential for efficiency and expressivity [S1].
*   **Curvature Threshold:** The choice of threshold for low-curvature points $P_L$ determines which points undergo densification [S1].
*   **Global Feature:** The global feature $g$ is shared across all points and computed as the maximum of point features [S1].

## Operating characteristics
*   **Time Complexity:** Training runs for 30,000 iterations [S1].
*   **Memory Complexity:** SAGS achieves significant memory reduction. On benchmark datasets, SAGS uses 99 MB compared to 3D-GS's 414 MB. The lightweight SAGS-Lite version achieves up to 24× size reduction without compression, using only 43 MB [S1].
*   **Performance:** SAGS outperforms 3D-GS and Scaffold-GS in PSNR and LPIPS on Mip-NeRF360, Tanks&Temples, and Deep Blending datasets. It better captures high-frequency details and reduces floating artifacts [S1].
*   **Strengths:** Reduced storage, fewer artifacts, better structure preservation, high rendering quality [S1].
*   **Limitations:** Requires initial sparse point cloud from COLMAP [S1].

## Background
Structure-Aware 3D Gaussian Splatting (SAGS) was proposed by E. Ververas et al. to address the limitations of structure-agnostic optimization schemes in 3D Gaussian Splatting, which often result in floating points and artifacts due to independent Gaussian optimization [S1]. The method builds on the 3D-GS framework [15] and incorporates ideas from graph neural networks and curvature-aware densification [S1]. It was published in 2024 [S1].

## Sources
[S1] SAGS: Structure-Aware 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-72655-2_13, fetched 2026-10-29
