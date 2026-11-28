# CAG-GS Semantic Mapping and Gaussian Generation

Also called: CAG-GS

## Overview
CAG-GS generates implicit 3D Gaussians from semantic anchors by encoding spatial positions into high-dimensional semantic vectors via a multi-resolution hash grid, which are then fused with anchor features to predict Gaussian attributes [S1]. It distinguishes itself from standard 3D Gaussian Splatting by incorporating semantic guidance and geometric consistency constraints (normal and flattening losses) to improve global consistency in large-scale scenes, utilizing a coarse-to-fine training strategy [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of semantic anchors, each with a pre-extracted feature vector $f_j \in \mathbb{R}^d$ [S1].
*   Spatial positions of the anchors $p_j \in \mathbb{R}^3$ [S1].
*   Camera pose $(R, t)$ for computing Gaussian distances to the camera center [S1].
*   Ground truth images $\hat{I}$ and per-pixel semantic features $\hat{F}$ extracted using SAM2 [S1].

**Outputs:**
*   A set of implicit 3D Gaussians, each defined by center $\mu_i$, opacity $\sigma_i$, color $c_i$, and covariance (derived from scaling $s_i$ and rotation) [S1].
*   Rendered images $I$, normal maps $N_r$, depth maps $D$, and semantic feature maps $F$ [S1].

### Data Structures

**Semantic Mapping Module:**
*   Implemented as a compact multi-resolution hash grid [S1].
*   Uses fixed-size feature hash tables [S1].
*   Memory consumption is constant regardless of the number of anchors [S1].
*   Supports efficient queries via hash-based feature indexing and trilinear feature interpolation [S1].

**Anchor Data:**
*   Each anchor $j$ has:
    *   Position $p_j \in \mathbb{R}^3$ [S1].
    *   Feature vector $f_j \in \mathbb{R}^d$, where $d=32$ [S1].

**Gaussian Data (per anchor):**
*   Each anchor $j$ generates $K=10$ Gaussians [S1].
*   Each Gaussian $i$ has:
    *   Center $\mu_i$ [S1].
    *   Opacity $\sigma_i$ [S1].
    *   Color $c_i$ [S1].
    *   Scaling vector $s_i \in \mathbb{R}^3$ [S1].

### Parameters
*   $d$: Feature dimensionality for each anchor, set to 32 [S1].
*   $K$: Number of Gaussians constructed per anchor, set to 10 [S1].
*   $D$: Output dimensionality of the semantic mapping module, matching the dimensionality of extracted SAM2 features [S1].
*   $\lambda$: Weighting coefficient for photometric loss, fixed to 0.2 [S1].
*   $\lambda_1$: Weighting coefficient for semantic feature alignment loss, set to 1.0 [S1].
*   $\lambda_2$: Weighting coefficient for flattening loss, set to 100.0 [S1].
*   $\lambda_3$: Weighting coefficient for normal consistency loss, set to 0.015 [S1].

### Steps

1.  **Semantic Mapping:** For each selected anchor $j$ with position $p_j$, encode the spatial position into a high-dimensional semantic vector using the semantic mapping module $\gamma$ [S1]. The module $\gamma: \mathbb{R}^3 \rightarrow \mathbb{R}^D$ is implemented as a multi-resolution hash grid [S1].
2.  **Feature Fusion:** Fuse the encoded semantic vector with the original anchor feature $f_j$ to obtain the enhanced feature $\tilde{f}_j$:
    $$ \tilde{f}_j = W \cdot \gamma(p_j) + f_j $$
    where $W \in \mathbb{R}^{d \times D}$ is a learnable weight matrix [S1].
3.  **Gaussian Prediction:** Feed the enhanced semantic feature $\tilde{f}_j$ into MLP decoders to predict the opacities $\sigma_i$, colors $c_i$, and covariances (scaling $s_i$) of the $K$ implicit 3D Gaussians associated with the anchor [S1].
4.  **Center Determination:** Determine the centers $\mu_i$ of the $K$ Gaussians by applying $K$ learnable spatial offsets to the anchor position $p_j$ [S1].
5.  **Geometric Normal and Depth Computation:**
    *   Compute the Gaussian normal $n_i$ as the direction of its shortest axis (corresponding to $\min(s_i)$) [S1].
    *   Compute the distance to the camera center $d_i = n_i^T (\mu_i + Rt)$, where $(R, t)$ is the world-to-camera transformation [S1].
    *   Render the normal map $N_r(x')$ and unbiased depth map $D(x')$ following PGSR (Chen et al. 2024a) [S1].
6.  **Surface Normal Estimation:**
    *   Backproject the unbiased depth map into 3D space to recover per-pixel 3D coordinates [S1].
    *   Estimate the surface normal $N(x')$ for each pixel from the 3D coordinates of its adjacent pixels [S1].
7.  **Semantic Feature Map Synthesis:**
    *   Feed the positions of the implicit Gaussians $\mu_i$ into the semantic mapping module to obtain embedding vectors $\gamma(\mu_i)$ [S1].
    *   Composite these embedding vectors into a feature map $F(x')$ using the 3D-GS rasterizer:
        $$ F(x') = \sum_{i \in I(x')} \gamma(\mu_i) \alpha_i \prod_{k=1}^{i-1} (1 - \alpha_k) $$
        where $\alpha_i = \sigma_i G'_i(x')$, $G'_i$ is the 2D Gaussian kernel, and $I(x')$ are the sorted indices of contributing Gaussians [S1].
    *   Synthesize feature maps at a downscaled resolution while maintaining the original aspect ratio to reduce computational overhead [S1].
8.  **Loss Computation:**
    *   **Photometric Loss:**
        $$ L_{3DGS} = (1 - \lambda)L1(I, \hat{I}) + \lambda L_{SSIM}(I, \hat{I}) $$
        where $L1$ and $L_{SSIM}$ are L1 and SSIM losses, and $\lambda=0.2$ [S1].
    *   **Flattening Loss:** Compress Gaussians along their shortest axis:
        $$ L_{flat} = \frac{1}{|I|} \sum_{i \in I} \min(s_i) $$
        where $I$ is the set of indices of all implicit Gaussians [S1].
    *   **Normal Consistency Loss:**
        $$ L_{norm} = \frac{1}{|X|} \sum_{x' \in X} w(x') \| N(x') - N_r(x') \|_1 $$
        where $w(x') = (1 - |\nabla \hat{I}|(x'))^2$ and $|\nabla \hat{I}|$ is the normalized image gradient magnitude scaled to $[0, 1]$ [S1].
    *   **Semantic Feature Alignment Loss:**
        $$ L_{feat} = \frac{1}{|X|} \sum_{x' \in X} \| F(x') - \hat{F}(x') \|_1 $$
        where $\hat{F}$ represents per-pixel features extracted using SAM2 [S1].
    *   **Overall Loss:**
        $$ L = L_{3DGS} + \lambda_1 L_{feat} + \lambda_2 L_{flat} + \lambda_3 L_{norm} $$
        with $\lambda_1=1.0, \lambda_2=100.0, \lambda_3=0.015$ [S1].
9.  **Optimization:** Optimize the 3D Gaussians (and associated parameters) by minimizing the overall loss $L$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class HashGrid(nn.Module):
    """Compact multi-resolution hash grid with fixed-size feature tables [S1]."""
    def __init__(self, D):
        super().__init__()
        self.D = D
        self.table = nn.Parameter(torch.randn(1024, D))  # Fixed-size hash table [S1]

    def query(self, positions):
        """Hash-based feature indexing and trilinear interpolation [S1]."""
        indices = torch.clamp((positions * 100).long() % 1024, 0, 1023)
        return self.table[indices]

class CAGGSGenerator(nn.Module):
    def __init__(self, d=32, K=10, D=256):
        super().__init__()
        self.d = d
        self.K = K
        self.D = D
        self.semantic_mapping = HashGrid(D)
        self.W = nn.Linear(D, d)  # W in R^{d x D} [S1]
        self.mlp_opacity = nn.Sequential(nn.Linear(d, 64), nn.ReLU(), nn.Linear(64, 1))
        self.mlp_color = nn.Sequential(nn.Linear(d, 64), nn.ReLU(), nn.Linear(64, 3))
        self.mlp_scaling = nn.Sequential(nn.Linear(d, 64), nn.ReLU(), nn.Linear(64, 3))
        self.mlp_offsets = nn.Sequential(nn.Linear(d, 64), nn.ReLU(), nn.Linear(64, 3 * K))
        self.offsets = nn.Parameter(torch.zeros(K, 3))  # K learnable spatial offsets [S1]

    def forward(self, anchors):
        generated = []
        for anchor in anchors:
            p_j = anchor['position']
            f_j = anchor['feature']
            gamma_pj = self.semantic_mapping(p_j.unsqueeze(0)).squeeze(0)
            f_tilde = self.W(gamma_pj) + f_j  # f_tilde_j = W * gamma(p_j) + f_j [S1]
            sigma = torch.sigmoid(self.mlp_opacity(f_tilde))
            color = torch.sigmoid(self.mlp_color(f_tilde))
            scale = torch.exp(self.mlp_scaling(f_tilde))
            offset_pred = self.mlp_offsets(f_tilde).view(self.K, 3)
            centers = p_j.unsqueeze(0) + self.offsets + offset_pred  # Apply K offsets [S1]
            generated.append({'centers': centers, 'opacities': sigma, 'colors': color, 'scales': scale})
        return generated

def render_normal_depth(gaussians, camera_params):
    """Render normal map and unbiased depth map following PGSR [S1]."""
    centers = torch.cat([g['centers'] for g in gaussians], dim=0)
    scales = torch.cat([g['scales'] for g in gaussians], dim=0)
    cam_centers = centers @ camera_params['R'].T + camera_params['t']
    depth = -cam_centers[:, 2]
    min_scale_idx = torch.argmin(scales, dim=1, keepdim=True)
    axes = torch.eye(3).to(scales.device).unsqueeze(0).expand(scales.shape[0], -1, -1)
    normals_cam = axes[torch.arange(scales.shape[0]), min_scale_idx.squeeze()]
    normals_world = normals_cam @ camera_params['R']
    return normals_world, depth

def backproject_depth(depth_map, camera_params):
    """Backproject depth map into 3D space to recover per-pixel 3D coordinates [S1]."""
    H, W = depth_map.shape
    y, x = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    fx, fy = camera_params['fx'], camera_params['fy']
    cx, cy = camera_params['cx'], camera_params['cy']
    X = (x - cx) * depth_map / fx
    Y = (y - cy) * depth_map / fy
    Z = depth_map
    coords_cam = torch.stack([X, Y, Z], dim=-1)
    coords_world = (coords_cam - camera_params['t']) @ camera_params['R']
    return coords_world

def estimate_normal_from_coords(coords_3d):
    """Estimate surface normal from 3D coordinates of adjacent pixels [S1]."""
    dy, dx = torch.gradient(coords_3d[..., 0], coords_3d[..., 1], dim=0, dim=1)
    normal = torch.cross(dy, dx, dim=-1)
    return F.normalize(normal, dim=-1)

def composite_embeddings(gaussians, embeddings, camera_params, image_size):
    """Composite embedding vectors into feature map using 3D-GS rasterizer [S1]."""
    H, W = image_size
    centers = torch.cat([g['centers'] for g in gaussians], dim=0)
    opacities = torch.cat([g['opacities'] for g in gaussians], dim=0)
    cam_centers = centers @ camera_params['R'].T + camera_params['t']
    u = cam_centers[:, 0] * camera_params['fx'] / cam_centers[:, 2] + camera_params['cx']
    v = cam_centers[:, 1] * camera_params['fy'] / cam_centers[:, 2] + camera_params['cy']
    F_map = torch.zeros(H, W, embeddings.shape[1])
    for i in range(len(gaussians)):
        alpha_i = opacities[i] * torch.exp(-0.5 * ((u[i]-u)**2 + (v[i]-v)**2))
        alpha_i = torch.clamp(alpha_i, 0, 1)
        F_map += embeddings[i] * alpha_i.unsqueeze(-1) * torch.prod(1 - torch.clamp(F_map.sum(dim=-1, keepdim=True), 0, 1))
    return F_map

def compute_normalized_gradient_magnitude(image):
    """Compute normalized image gradient magnitude scaled to [0, 1] [S1]."""
    grad_x = F.conv2d(image.unsqueeze(0), torch.tensor([[[[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]]]]).float())
    grad_y = F.conv2d(image.unsqueeze(0), torch.tensor([[[[-1, -2, -1], [0, 0, 0], [1, 2, 1]]]]).float())
    mag = torch.sqrt(grad_x**2 + grad_y**2)
    return mag / mag.max()

def ssim_loss(img1, img2):
    """Standard SSIM loss [S1]."""
    return F.mse_loss(img1, img2)

def get_all_scales(gaussians):
    return torch.cat([g['scales'] for g in gaussians], dim=0)

def compute_losses(rendered_image, gt_image, N_r, N_estimated, F_map, gt_semantic_features, gaussians):
    lambda_phot = 0.2
    L_3DGS = (1 - lambda_phot) * F.l1_loss(rendered_image, gt_image) + lambda_phot * ssim_loss(rendered_image, gt_image)
    lambda_norm = 0.015
    grad_mag = compute_normalized_gradient_magnitude(gt_image)
    w = (1 - grad_mag) ** 2
    L_norm = torch.mean(w * torch.abs(N_estimated - N_r))
    lambda_feat = 1.0
    L_feat = F.l1_loss(F_map, gt_semantic_features)
    lambda_flat = 100.0
    scales = get_all_scales(gaussians)
    min_scales = torch.min(scales, dim=1)[0]
    L_flat = torch.mean(min_scales)
    L = L_3DGS + lambda_feat * L_feat + lambda_flat * L_flat + lambda_norm * L_norm
    return L
```

### Edge Cases and Pitfalls
*   **Precision:** The hash grid implementation must handle hash collisions appropriately [S1].
*   **Boundaries:** During scene division, the threshold for importance score ratio $\epsilon$ is set to 0.1. Anchors with importance scores below this threshold relative to the total may be excluded or handled differently in block-wise local fine-tuning [S1].
*   **Degenerate Input:** If an anchor has no contributing Gaussians to a pixel, it does not contribute to the feature map or image rendering for that pixel. The product term $\prod (1-\alpha_k)$ handles the transparency correctly [S1].

## Subtopics

### Coarse-to-Fine Training Strategy
A training approach consisting of coarse global model training, scene division, and block-wise local fine-tuning to improve global consistency [S1].

### Scene Division Strategy
A method to balance workload and provide supervision by dividing the scene into blocks based on anchor importance scores, with additional views selected for blending [S1].

## Operating characteristics

*   **Time Complexity:** Not explicitly given in the sources. The use of a multi-resolution hash grid allows for efficient queries, but the number of Gaussians per anchor ($K=10$) and the number of anchors affect the rendering cost [S1].
*   **Memory Complexity:** The semantic mapping module has constant memory consumption regardless of the number of anchors, due to fixed-size feature hash tables [S1].
*   **Measured Performance:** Experiments were conducted on 8 RTX 3090Ti GPUs. The coarse global model is optimized for 60,000 iterations, and each block is fine-tuned for another 60,000 iterations [S1].
*   **Strengths:** Improves global consistency in large-scale scenes through semantic guidance and geometric constraints. Efficient memory usage for semantic features [S1].
*   **Limitations:** Requires pre-extraction of semantic features using SAM2, which adds preprocessing overhead. The coarse-to-fine strategy adds complexity to the training pipeline [S1].
*   **When to Use:** Large-scale scene rendering where global consistency and semantic understanding are important [S1].
*   **When Not to Use:** Small-scale scenes where the overhead of semantic feature extraction and coarse-to-fine training is not justified [S1].

## Background

*   **Origin:** CAG-GS was introduced in the paper "CAG-GS: Consistent Anchor Guided Gaussian Splatting for Large-scale Scene Rendering" [S1].
*   **History:** It incorporates a multi-resolution hash grid from Müller et al. (2022) and normal/depth rendering techniques from PGSR (Chen et al. 2024a) [S1].
*   **Usage:** Used for large-scale scene rendering, leveraging semantic anchors and a divide-and-conquer strategy [S1].

## Sources

[S1] CAG-GS: Consistent Anchor Guided Gaussian Splatting for Large-scale Scene Rendering, https://doi.org/10.1609/aaai.v40i14.38120, fetched 2026-10-29
