# Generalizable Implicit Reconstruction with SRDF

Also called: VolRecon, Volume Rendering of Signed Ray Distance Functions for Generalizable Multi-View Reconstruction

## Overview
VolRecon is a generalizable implicit reconstruction method that reconstructs scenes with fine details and low noise by combining local projection features aggregated from multi-view images with global shape priors from a coarse feature volume [S1]. It differs from its siblings, such as [SparseNeuS](11770-nerf-vpt.md) and [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md), by utilizing a Signed Ray Distance Function (SRDF) instead of a standard SDF, and by employing a ray transformer to aggregate features along the viewing ray, thereby capturing both local geometric cues and global context [S1]. This approach addresses the over-smoothing issues common in methods relying solely on global feature volumes [S1].

## How it works
**Inputs:** A set of $N$ source images $I = \{I_1, \dots, I_N\}$ where $I_i \in \mathbb{R}^{H \times W \times 3}$, and a reference view for which the reconstruction is performed.
**Outputs:** Rendered color image, depth map, and a signed distance field representation of the scene.

**Data Structures:**
1.  **Feature Maps:** $\{F_i\}_{i=1}^N \in \mathbb{R}^{H/4 \times W/4 \times C}$, extracted from source images.
2.  **Global Feature Volume:** $F_v \in \mathbb{R}^{K \times K \times K \times C}$, a 3D grid of voxels.
3.  **Ray Points:** A set of $M$ points $\{p(t) = o + tv \mid t \ge 0\}$ sampled along the ray from the reference pixel.

**Parameters:**
*   $N$: Number of source views.
*   $M$: Number of sampled points per ray.
*   $K$: Dimension of the global feature volume grid ($K^3$ voxels).
*   $C$: Channel dimension of feature maps.
*   $f_0$: Learnable aggregation token (vector of size $C$).

**Step 1: Feature Extraction**
For each source image $I_i$, extract feature maps $F_i$ using a Feature Pyramid Network (FPN) [S1].

**Step 2: Global Feature Volume Construction**
1.  Define the scene bounding volume and divide it into $K^3$ voxels.
2.  For each voxel center $c_j$, project it onto the feature map $F_i$ of each source view $i \in \{1, \dots, N\}$ using bilinear interpolation to obtain features $f_{i,j}$.
3.  Compute the mean $\mu_j$ and variance $\sigma_j$ of the $N$ features $\{f_{i,j}\}_{i=1}^N$.
4.  Concatenate $\mu_j$ and $\sigma_j$ to form the initial voxel feature.
5.  Pass the volume of voxel features through a 3D U-Net [38] to regularize and aggregate information, resulting in the global feature volume $F_v$ [S1].

**Step 3: Ray Sampling and Projection Feature Extraction**
For a pixel in the reference view, define the ray $p(t) = o + tv$. Sample $M$ points $\{p_m\}_{m=1}^M$ along the ray.
For each point $p_m$:
1.  Project $p_m$ onto the feature map $F_i$ of each source view $i$.
2.  Extract color $c_{i,m}$ and feature $f_{i,m}$ using bilinear interpolation [S1].

**Step 4: View Transformer**
Aggregate multi-view features for each point $p_m$ into a single projection feature $f_{p,m}$.
1.  Initialize a learnable aggregation token $f_0$.
2.  Input $\{f_0, \{f_{i,m}\}_{i=1}^N\}$ into a View Transformer. The transformer uses self-attention with linear attention [19].
3.  Do not use positional encoding, as the order of source views is not assumed [S1].
4.  Compute:
    $$ f_{p,m}, \{f'_{i,m}\}_{i=1}^N = \text{ViewTrans}(f_0, \{f_{i,m}\}_{i=1}^N) $$
    where $f_{p,m}$ is the projection feature and $f'_{i,m}$ are updated multi-view features [S1].

**Step 5: Volume Feature Interpolation**
For each point $p_m$, interpolate the feature from the global feature volume $F_v$ at the location of $p_m$ to obtain the volume feature $f_{v,m}$ [S1].

**Step 6: Ray Transformer and SRDF Prediction**
1.  Concatenate the features of all $M$ points along the ray. For each point $m$, the input to the ray transformer is the concatenation of its projection feature $f_{p,m}$ and volume feature $f_{v,m}$.
2.  Apply a Ray Transformer to allow each point to attend to features of other points on the ray.
3.  The Ray Transformer outputs the SRDF value $\tilde{d}_\Omega(p_m, v)$ for each point $p_m$ along the ray direction $v$ [S1].

**Step 7: Volume Rendering**
Model the density function using the SRDF values and render the color and depth map using the volume rendering technique from NeuS [49].

**Formulas:**
The Signed Ray Distance Function (SRDF) $\tilde{d}_\Omega(p, v)$ is defined as the signed distance to the nearest surface point $p^* \in M$ along the ray direction $v$ ($||v||_2 = 1$) [S1]:
$$ \tilde{d}_\Omega(p, v) = (-1)^{1_\Omega(p)} \min_{p^* \in M, \frac{p^*-p}{||p^*-p||} = v} ||p - p^*||_2 $$
where $1_\Omega(p) = 1$ if $p \in \Omega$ (inside) and $0$ if $p \notin \Omega$ (outside) [S1].

The standard Signed Distance Function (SDF) $d_\Omega(p)$ is the minimum absolute SRDF over all directions $v$ [S1]:
$$ d_\Omega(p) = (-1)^{1_\Omega(p)} \min_v (|\tilde{d}_\Omega(p, v)|) $$
Which simplifies to the standard SDF definition [S1]:
$$ d_\Omega(p) = (-1)^{1_\Omega(p)} \min_{p^* \in M} ||p - p^*||_2 $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VolRecon:
    def __init__(self, K, C, N, M):
        """
        K: Grid dimension for global feature volume
        C: Channel dimension
        N: Number of source views
        M: Number of sampled points per ray
        """
        self.K = K
        self.C = C
        self.N = N
        self.M = M
        
        # Global Feature Volume
        self.F_v = nn.Parameter(torch.randn(K, K, K, C))
        
        # Learnable aggregation token for View Transformer
        self.f_0 = nn.Parameter(torch.randn(C))
        
        # Feature Pyramid Network (FPN) for source views
        self.fpn = FeaturePyramidNetwork()
        
        # 3D U-Net for global feature volume regularization
        self.global_volume_net = UNet3D()
        
        # View Transformer (Self-attention with linear attention)
        self.view_transformer = ViewTransformer(C)
        
        # Ray Transformer
        self.ray_transformer = RayTransformer(C)
        
        # MLP to predict SRDF from ray transformer output
        self.srdf_mlp = nn.Sequential(
            nn.Linear(C, C),
            nn.ReLU(),
            nn.Linear(C, 1)
        )

    def extract_features(self, images):
        """
        images: Tensor of shape (N, 3, H, W)
        Returns: List of feature maps {F_i} of shape (N, C, H/4, W/4)
        """
        return [self.fpn(img) for img in images]

    def build_global_volume(self, feature_maps):
        """
        feature_maps: List of N tensors, each (C, H/4, W/4)
        Returns: Global feature volume F_v of shape (K, K, K, C)
        """
        # 1. Project voxel centers to feature maps
        # Assume voxel_centers is precomputed tensor of shape (K^3, 3) normalized to [0,1]
        # and view_projection_params contains camera intrinsics/extrinsics
        voxel_features = []
        for k in range(self.K**3):
            center = voxel_centers[k] # (3,)
            voxel_feat = []
            for i in range(self.N):
                F_i = feature_maps[i]
                # Project center to view i
                uv = project_to_uv(center, view_projection_params[i])
                # Bilinear interpolation
                feat = bilinear_interpolate(F_i, uv) # (C,)
                voxel_feat.append(feat)
            
            voxel_feat = torch.stack(voxel_feat, dim=0) # (N, C)
            mean_feat = torch.mean(voxel_feat, dim=0)
            var_feat = torch.var(voxel_feat, dim=0)
            concat_feat = torch.cat([mean_feat, var_feat], dim=0) # (2C,)
            voxel_features.append(concat_feat)
            
        voxel_features = torch.stack(voxel_features, dim=0).view(self.K, self.K, self.K, 2*self.C)
        
        # 2. Regularize with 3D U-Net
        # Note: The source says mean/var are concatenated, but U-Net input might need adjustment.
        # Assuming U-Net handles the channel dimension or we project back to C.
        # For implementation clarity, we assume the U-Net takes the concatenated features.
        F_v_raw = self.global_volume_net(voxel_features)
        
        # The source states F_v is the result. We store it.
        # In practice, F_v might be a grid of size KxKxKxC.
        # The interpolation step later assumes F_v is C channels.
        # If U-Net outputs 2C, we might need a projection layer. 
        # Based on "concatenated as the voxel features... use 3D U-Net to regularize", 
        # we assume the output is the final F_v.
        return F_v_raw

    def get_projection_feature(self, point, feature_maps):
        """
        point: (3,) 3D coordinate
        feature_maps: List of N tensors (C, H/4, W/4)
        Returns: Projection feature f_p (C,)
        """
        multi_view_feats = []
        for i in range(self.N):
            F_i = feature_maps[i]
            uv = project_to_uv(point, view_projection_params[i])
            feat = bilinear_interpolate(F_i, uv)
            multi_view_feats.append(feat)
            
        multi_view_feats = torch.stack(multi_view_feats, dim=0) # (N, C)
        
        # View Transformer
        # Input: f_0 (C,) and multi_view_feats (N, C)
        # The transformer expects a sequence. f_0 is the aggregation token.
        # Input sequence: [f_0, f_1, ..., f_N] -> (N+1, C)
        input_seq = torch.cat([self.f_0.unsqueeze(0), multi_view_feats], dim=0)
        
        # Output: f_p (C,) and updated features
        # The source says: f_p, {f'_i} = ViewTrans(f_0, {f_i})
        # We assume the first token of the output is f_p
        out_seq = self.view_transformer(input_seq)
        f_p = out_seq[0]
        
        return f_p

    def get_volume_feature(self, point, F_v):
        """
        point: (3,) 3D coordinate
        F_v: (K, K, K, C)
        Returns: Volume feature f_v (C,)
        """
        # Trilinear interpolation
        f_v = trilinear_interpolate(F_v, point)
        return f_v

    def predict_srdf(self, ray_points, feature_maps, F_v):
        """
        ray_points: (M, 3)
        feature_maps: List of N tensors
        F_v: (K, K, K, C)
        Returns: SRDF values (M,)
        """
        M = ray_points.shape[0]
        concatenated_features = []
        
        for m in range(M):
            p = ray_points[m]
            
            # Get projection feature
            f_p = self.get_projection_feature(p, feature_maps)
            
            # Get volume feature
            f_v = self.get_volume_feature(p, F_v)
            
            # Concatenate
            f_concat = torch.cat([f_p, f_v], dim=0) # (2C,)
            concatenated_features.append(f_concat)
            
        concatenated_features = torch.stack(concatenated_features, dim=0) # (M, 2C)
        
        # Ray Transformer
        # The source says "concatenated features of all M points... compute SRDF values"
        # We assume the Ray Transformer takes the sequence of features and outputs features for each point
        ray_out = self.ray_transformer(concatenated_features) # (M, C)
        
        # MLP to predict SRDF
        srdf_values = self.srdf_mlp(ray_out).squeeze(-1) # (M,)
        
        return srdf_values

    def render(self, ray_points, srdf_values, colors):
        """
        ray_points: (M, 3)
        srdf_values: (M,)
        colors: (M, 3) colors of the points
        Returns: Rendered color and depth
        """
        # Volume rendering based on NeuS [49]
        # Convert SRDF to density and transmittance
        # NeuS uses SDF. SRDF is used similarly.
        # Density sigma is derived from the derivative of SRDF or a sigmoid of SRDF
        # Standard NeuS: sigma = sigmoid(SDF / alpha) or similar.
        # For SRDF, the sign indicates inside/outside.
        
        # Approximation of density from SRDF
        # Let's assume a standard conversion: sigma = max(0, -d/dt(SRDF)) or similar
        # Since we have discrete points, we use the SRDF values directly.
        # NeuS paper: sigma = sigmoid(SDF / alpha) is not quite right for SDF.
        # NeuS uses: alpha = 1 - exp(-sigma * delta_t)
        # And sigma is predicted by an MLP from SDF.
        # Here, SRDF is predicted. We can use a similar mapping.
        
        # Let's assume a simple mapping for implementation:
        # sigma = max(0, -srdf) ? No, SRDF can be positive.
        # The source says "Model the density function with SRDF".
        # In NeuS, SDF is used to determine surface location.
        # We will use the standard volume rendering equation with alpha derived from SRDF.
        
        # Calculate distances between points
        deltas = torch.diff(ray_points, dim=0, prepend=ray_points[0:1]) # (M, 3)
        dists = torch.norm(deltas, dim=1) # (M,)
        
        # Convert SRDF to alpha
        # Using a sigmoid-like function centered at 0
        # alpha = 1 - exp(-max(0, -srdf) * dist) ? 
        # Or simply: sigma = -d(SRDF)/dt. 
        # For discrete points, we can use the SRDF value to modulate alpha.
        # A common approach for SDF rendering: 
        # alpha = 1 - exp(-sigma * delta)
        # sigma = sigmoid(SDF / beta) * scale ?
        # Let's stick to the NeuS formulation adapted for SRDF.
        # NeuS: sigma = sigmoid(SDF / alpha) is incorrect. 
        # NeuS: sigma = max(0, -d(SDF)/dt) is not used directly.
        # NeuS uses an MLP to predict sigma from SDF.
        # Here, we predict SRDF. We can use an MLP to predict sigma from SRDF.
        
        sigma = torch.sigmoid(-srdf_values) * 10.0 # Example scaling
        
        alphas = 1.0 - torch.exp(-sigma * dists)
        transmittances = torch.cumprod(1.0 - alphas + 1e-10, dim=0)
        transmittances = torch.cat([torch.ones(1, device=alphas.device), transmittances[:-1]], dim=0)
        
        weights = alphas * transmittances
        color = torch.sum(weights.unsqueeze(-1) * colors, dim=0)
        depth = torch.sum(weights * ray_points[:, 2], dim=0) # Assuming z is depth
        
        return color, depth

def bilinear_interpolate(feature_map, uv):
    """
    feature_map: (C, H, W)
    uv: (2,) normalized coordinates [0, 1]
    Returns: (C,)
    """
    # Normalize uv to [-1, 1] for grid_sample
    uv_norm = 2 * uv - 1
    uv_norm = uv_norm.unsqueeze(0).unsqueeze(0).unsqueeze(0) # (1, 1, 1, 2)
    feature_map = feature_map.unsqueeze(0).unsqueeze(0) # (1, 1, C, H, W)
    
    # grid_sample expects (N, H_out, W_out, 2)
    # We need to reshape
    uv_grid = uv_norm.permute(0, 3, 1, 2) # (1, 2, 1, 1)
    
    # Actually, grid_sample input is (N, H, W, 2) for the grid
    # Let's use a simpler manual interpolation or assume a helper
    # For the sake of the article, we assume a standard bilinear interp function
    pass

def trilinear_interpolate(volume, point):
    """
    volume: (K, K, K, C)
    point: (3,) normalized coordinates [0, 1]
    Returns: (C,)
    """
    pass

def project_to_uv(point, camera_params):
    """
    point: (3,)
    camera_params: dict with K, R, T
    Returns: uv (2,) normalized
    """
    pass
```

## Operating characteristics
*   **Time Complexity:** Rendering is computationally expensive. It takes approximately 30 seconds to render an image and depth map at 800x600 resolution [S1]. This is typical for volume rendering-based methods [S1].
*   **Memory Complexity:** The global feature volume scales with $K^3$. Increasing $K$ improves resolution but increases memory consumption significantly [S1].
*   **Performance:** On the DTU dataset, VolRecon outperforms SparseNeuS by about 30% in sparse view reconstruction (N=3) and 22% in full view reconstruction [S1]. It achieves comparable accuracy to MVSNet in full view reconstruction and performs better in depth evaluation [S1]. Novel View Synthesis on DTU (4 source views) yields PSNR 15.37, MSE 0.04, SSIM 0.56 [S1].
*   **Strengths:** Combines local projection features and global volume features to reconstruct fine details with less noise than methods using only global volumes (e.g., SparseNeuS) [S1]. Good generalization to large-scale scenes (ETH3D) without fine-tuning [S1].
*   **Limitations:** Low rendering efficiency [S1]. Not suitable for very large-scale scenes due to the fixed resolution of the global feature volume [S1].

## Background
VolRecon was introduced in 2023 by researchers from the IVRL lab [S1]. It builds upon the concept of generalizable neural implicit reconstruction, aiming to overcome the lack of generalization in per-scene optimization methods like NeRF [S1]. It specifically addresses the limitations of SparseNeuS, which uses only global feature volumes and an MLP, leading to over-smoothed surfaces [S1]. VolRecon incorporates a view transformer and a ray transformer to better capture local and global information [S1].

## Sources
[S1] VolRecon: Volume Rendering of Signed Ray Distance Functions for Generalizable Multi-View Reconstruction, https://doi.org/10.1109/cvpr52729.2023.01601, fetched 2026-10-26
