# RING-NeRF

Also called: Distance-Aware Coarse-To-Fine, RING-NeRF pipeline, Residual Implicit Neural Grids

## Overview
RING-NeRF is a neural field architecture that introduces two inductive biases: a continuous multi-scale representation of the scene and an invariance of the decoder’s latent space over spatial and scale domains [S1]. It solves the problem of rendering quality degradation at varying observation distances and resolutions by providing intrinsic continuous Level of Detail (LOD) without explicit LOD-specific supervision [S1]. Unlike sibling methods such as [AdaNeRF](6195-adanerf.md) or [Zip-NeRF](8875-multiplanenerf.md) which require explicit supervision for LOD or modify features based on scale, RING-NeRF encodes LOD exclusively in the grid hierarchy and uses a distance-aware forward mapping to adapt to observation distance, avoiding aliasing artifacts [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera pose (origin $\mathbf{o}$, direction $\mathbf{d}$), image resolution parameters (pixel size $c$ at distance 1), grid hierarchy parameters (base resolution $b$, growth factor $f$), and contraction function Jacobian $J$.
*   **Outputs:** Rendered pixel color $\hat{C}(\mathbf{r})$.

**Data Structures:**
*   **Unbounded Hierarchical 3D Grid:** A pyramid of $N$ grids. The resolution of grid $i$ is $f^i b$. Each grid stores latent features of size $D$ (typically 8).
*   **Decoder:** A small Multi-Layer Perceptron (MLP) and linear layers.

**Numbered Steps:**

1.  **Scene Representation (Grid Construction):**
    The scene is represented by a recursive multi-scale mapping function. The latent feature grid $G_N$ at level $N$ is defined recursively [S1]:
    $$
    G_0 \leftarrow \delta_0
    $$
    $$
    G_N \leftarrow S(G_{N-1}) + \delta_N
    $$
    where $\delta_i$ is the latent feature grid at level $i$, and $S$ is a subdivision scheme that increases the resolution of $G_{N-1}$ to the resolution of $G_N$ via tri-linear interpolation. The latent vector for a point at level $N$ is obtained by linearly interpolating the point in the cell for each level $k \le N$ and summing the results [S1]. This summation acts as a residual connection.

2.  **Continuous LOD:**
    To allow continuous LOD $L \in [N-1, N]$, a weighting factor $\alpha = L - (N-1)$ is applied [S1]:
    $$
    G_L \leftarrow S(G_{N-1}) + \alpha \delta_N
    $$

3.  **Distance-Aware Forward Mapping (LOD Selection):**
    For a sample at distance $d$ from the camera, the LOD $L$ is calculated by equating the cone volume in contracted space to the virtual grid cell volume [S1].
    *   Compute the cone volume in world space: $V = (dc)^3$, where $c$ is the pixel size at distance 1.
    *   Contract the volume using the Jacobian $J(p)$ of the contraction function at sample location $p$: $V_{contract} = V \det(J(p))$.
    *   Equate to the virtual grid cell volume $(\frac{1}{f^L b})^3$ and solve for $L$ [S1]:
        $$
        L = - \frac {\log \left (dcb \sqrt [3]{\operatorname {det}(J(p))}\right )}{\log (f)}
        $$
    *   Clamp $L$ to the maximum available grid level during training if using coarse-to-fine optimization.

4.  **Feature Extraction:**
    Interpolate the latent features from all grid levels $k \le L$ (or all levels if $L$ exceeds max grid, though typically clamped) and sum them to form the combined latent feature vector $\mathbf{f}_{combined}$ [S1].
    *   *Note:* Use summation, not concatenation, to maintain spatial and scale invariance of the decoder [S1].

5.  **Normalization:**
    Apply standard, non-learnable normalization to $\mathbf{f}_{combined}$ to improve convergence stability [S1].

6.  **Frequency Encoding:**
    Project the normalized feature into a higher dimension space using a Random Fourier Feature (RFF) mapping with a sinus filter [S1]:
    $$
    \mathbf{f}_{rff} = \sin(\mathbf{W} \mathbf{f}_{norm})
    $$
    where $\mathbf{W}$ is a learnable linear matrix (no bias) [S1].

7.  **Density/SDF and Color Feature Prediction:**
    Pass $\mathbf{f}_{rff}$ through a linear layer to produce a density (or SDF) value $\sigma$ and a single color feature vector $\mathbf{f}_{color}$ [S1].
    *   *SDF Mode:* If reconstructing SDF, convert $\sigma$ to density using a NEUS-like approach before volume rendering [S1].

8.  **View Direction Encoding:**
    Encode the view direction $\mathbf{d}$ using Spherical Harmonics (S.H.) [S1].

9.  **Radiance Prediction:**
    Concatenate $\mathbf{f}_{color}$ with the S.H. encoded direction. Feed this combined vector into a 3-layer MLP to predict the final radiance $\mathbf{c}$ [S1].

10. **Volume Rendering:**
    Compute the pixel color using the standard volume rendering integral [S1]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=0}^{N-1} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp(\sum_{j=0}^{i-1} \sigma_j \delta_j)$ and $\delta_i$ is the distance between samples.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.nn.functional as F

class RINGNeRF:
    def __init__(self, num_grids, base_resolution, growth_factor, feature_dim=8, 
                 rff_dim=256, mlp_hidden_dim=128, contraction_jacobian_func=None):
        """
        Initialize RING-NeRF.
        :param num_grids: Number of grid levels N.
        :param base_resolution: Resolution of the coarsest grid b.
        :param growth_factor: Growth factor f.
        :param feature_dim: Dimension of latent features in grids D.
        :param rff_dim: Dimension of RFF output.
        :param mlp_hidden_dim: Hidden dimension of the radiance MLP.
        :param contraction_jacobian_func: Function returning det(J(p)) for a point p.
        """
        self.num_grids = num_grids
        self.b = base_resolution
        self.f = growth_factor
        self.D = feature_dim
        self.rff_dim = rff_dim
        self.contraction_jacobian_func = contraction_jacobian_func
        
        # Grids: List of nn.Embedding or similar structure for each level
        # Resolution of level i is f^i * b
        self.grids = nn.ModuleList([
            nn.Embedding(int(f**i * b)**3, feature_dim) for i in range(num_grids)
        ])
        
        # Linear layer for RFF -> Density/Color
        self.linear_rff_to_feat = nn.Linear(rff_dim, feature_dim + 1) # 1 for density/SDF
        
        # RFF Matrix W (learnable)
        self.W = nn.Parameter(torch.randn(rff_dim, feature_dim))
        
        # Radiance MLP
        self.radiance_mlp = nn.Sequential(
            nn.Linear(feature_dim + 32, mlp_hidden_dim), # 32 from SH encoding
            nn.ReLU(),
            nn.Linear(mlp_hidden_dim, mlp_hidden_dim),
            nn.ReLU(),
            nn.Linear(mlp_hidden_dim, 3) # RGB
        )
        
        # Spherical Harmonics coefficients for view direction (order 4 -> 32 coeffs)
        # Simplified: using a fixed linear projection for SH if not implementing full SH
        self.sh_encoder = nn.Linear(3, 32) 

    def get_lod(self, distance, det_jacobian):
        """
        Calculate LOD L for a sample at given distance and Jacobian determinant.
        """
        # L = - log(d * c * b * det(J)^(1/3)) / log(f)
        # Note: c (pixel size at dist 1) is assumed to be 1.0 or passed in. 
        # Here we assume c=1 for simplicity or it's part of the distance scaling.
        # The formula in source: L = - log(dcb * det(J)^(1/3)) / log(f)
        # We need c. Let's assume c is passed or fixed.
        # For implementation, c is typically derived from image resolution and FOV.
        # We will assume c is a constant passed to the forward pass or set to 1.0 for normalized coords.
        # Let's assume c=1.0 for the formula derivation in the source context of contracted space.
        c = 1.0 
        numerator = math.log(distance * c * self.b * (det_jacobian ** (1/3)))
        denominator = math.log(self.f)
        return -numerator / denominator

    def interpolate_and_sum(self, point_coords, lod):
        """
        Interpolate features from all grids up to lod and sum them.
        point_coords: (N, 3) normalized coordinates in [-1, 1] or [0, 1] depending on grid setup.
        lod: (N,) float LOD values.
        """
        total_feature = torch.zeros_like(point_coords[:, :self.D])
        
        for i in range(self.num_grids):
            # Determine if this grid level contributes
            # If lod >= i, this grid contributes. 
            # If lod is continuous, we might weight it, but the source says:
            # "interpolating linearly the point in the cell for each level inferior or equal to N, and summing"
            # And for continuous LOD: G_L = S(G_{N-1}) + alpha * delta_N
            # This implies we sum all levels <= floor(lod) and partially the next?
            # Actually, the source says: "latent vector ... for a level of detail N is simply obtained by 
            # interpolating ... for each level inferior or equal to N, and summing".
            # For continuous L, we use the weighted residual connection.
            
            # Simplified implementation: Sum all grids up to floor(lod), and add partial for ceil(lod)
            # However, the source formula G_L = S(G_{N-1}) + alpha * delta_N suggests a recursive structure.
            # In practice, for a query point, we interpolate at each grid resolution and sum.
            
            # Let's implement the sum of interpolated features for all levels k <= L
            # If L is continuous, we interpolate at level floor(L) and ceil(L) and weight?
            # The source says: "G_L <- S(G_{N-1}) + alpha * delta_N".
            # This means the feature at LOD L is the feature at LOD N-1 (upsampled) plus alpha times the residual at N.
            # So we sum: Feature(0) + (Feature(1) - Feature(0)_upsampled) + ... 
            # Which simplifies to sum of interpolated residuals?
            # The source explicitly says: "interpolating linearly the point in the cell for each level inferior or equal to N, and summing the results."
            # This refers to the delta_i grids.
            
            # So, for a continuous LOD L:
            # Sum features from grids 0 to floor(L).
            # Add alpha * feature from grid ceil(L).
            
            level_idx = int(lod[i])
            alpha = lod[i] - level_idx
            
            # Interpolate at level_idx
            # Assuming grid indices are mapped from coords. 
            # This requires a trilinear interpolation function.
            feat_i = self.interpolate_grid(self.grids[level_idx], point_coords, resolution=int(self.f**level_idx * self.b))
            total_feature += feat_i
            
            if level_idx + 1 < self.num_grids and alpha > 0:
                feat_next = self.interpolate_grid(self.grids[level_idx + 1], point_coords, resolution=int(self.f**(level_idx + 1) * self.b))
                total_feature += alpha * feat_next
                
        return total_feature

    def interpolate_grid(self, grid, coords, resolution):
        # Placeholder for trilinear interpolation from grid embeddings
        # In a real implementation, this maps coords to grid indices and interpolates
        pass

    def forward(self, rays_o, rays_d, image_res, fov):
        """
        rays_o: (N, 3) ray origins
        rays_d: (N, 3) ray directions
        image_res: (H, W)
        fov: float
        """
        # 1. Sample points along rays
        # Standard NeRF sampling: N samples per ray
        N_samples = 128
        t_vals = torch.linspace(0., 1., N_samples)
        z_vals = 0.1 * t_vals + 0.9 * (t_vals ** 2) # Example stratified sampling
        # Expand to (N_rays, N_samples, 3)
        points = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(2)
        
        # 2. Compute LOD for each point
        # Need pixel size c at distance 1. 
        # c = tan(fov/2) * 2 / image_res (approx)
        # Let's assume c is computed.
        c = 1.0 # Placeholder, should be calculated from fov and res
        
        # Compute det(J(p)) for contraction
        # Assuming a standard contraction function like in Mip-NeRF 360
        # det(J) is computed analytically.
        det_j = self.contraction_jacobian_func(points)
        
        distances = torch.norm(points - rays_o.unsqueeze(1), dim=2)
        lod = self.get_lod(distances, det_j)
        
        # 3. Extract Features
        features = self.interpolate_and_sum(points, lod)
        
        # 4. Normalize
        features = F.normalize(features, dim=-1)
        
        # 5. RFF
        rff = torch.sin(features @ self.W.T) # W is (rff_dim, D) or (D, rff_dim)? Source: y=sin(Wx). W is linear matrix.
        # If W is (D, rff_dim), then features @ W.T
        
        # 6. Linear to Density/Color Feature
        # Linear layer transforms RFF output into density and one color feature
        linear_out = self.linear_rff_to_feat(rff)
        density = linear_out[:, :1]
        color_feat = linear_out[:, 1:]
        
        # 7. View Direction Encoding
        sh_feat = self.sh_encoder(rays_d)
        
        # 8. Radiance MLP
        concat = torch.cat([color_feat, sh_feat], dim=-1)
        radiance = self.radiance_mlp(concat)
        
        # 9. Volume Rendering
        # Convert density to alpha
        # delta_i is distance between samples
        deltas = z_vals[:, 1:] - z_vals[:, :-1]
        deltas = torch.cat([deltas, torch.tensor([1e10]).to(deltas.device)], dim=-1)
        
        alpha = 1.0 - torch.exp(-torch.relu(density) * deltas)
        transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[:, :1]), 1.0 - alpha + 1e-10], dim=-1), dim=-1)[:, :-1]
        
        weights = alpha * transmittance
        color = torch.sum(weights.unsqueeze(-1) * radiance, dim=1)
        
        return color
```

**Parameters:**
*   $c$: Pixel size at distance 1. Depends on image resolution and camera FOV [S1].
*   $J$: Jacobian of the contraction function at sample location $p$ [S1].
*   $f$: Growth factor for grid hierarchy resolution (e.g., 2) [S1].
*   $l_0$: Initial number of grids used in coarse-to-fine optimization [S1].
*   $n_{ctf}$: Hyperparameter for the speed of the coarse-to-fine process [S1].
*   Feature size: 8 [S1].
*   MLP hidden dimension: 64 (few-view setup) or 128 (NVS) [S1].
*   Number of samples per ray: 128 [S1].
*   Hashgrid configuration: Pyramid of 9 grids with growth factor 2 (in some experiments) [S1].

**Edge Cases and Pitfalls:**
*   **Concatenation vs Summation:** Using concatenation instead of summation for multi-resolution features leads to decreased performance in generalization to novel unseen observation distances [S1].
*   **Decoder Latent Space:** Unlike I-NGP-based architectures, RING-NeRF keeps supervising all dimensions of the decoder latent space even when grids are zeroed out in coarse-to-fine training, preventing global refinement of decoder weights when new grids are activated [S1].
*   **Unobserved Samples:** During novel view rendering, discard samples not observed by any training view to remove noise [S1].

## Operating characteristics
*   **Time Complexity:** Training time is significantly faster than many dedicated architectures. On the 360 Dataset Mono-Scale setup, RING-NeRF achieves 28.09 PSNR with a training time of 0.45 hours on one Nvidia-A100 GPU [S1].
*   **Memory Complexity:** The unbounded hierarchical grid allows dynamic resolution increase. The '3+1+1 levels' configuration results in the same rendering quality as the '5 levels' configuration, demonstrating efficient memory usage by only activating necessary grids [S1].
*   **Performance:**
    *   360 Dataset Mono-Scale, Full Res: PSNR 28.09, SSIM 0.799, LPIPS 0.157 [S1].
    *   360 Dataset Mono-Scale, 1/2 Res: PSNR 27.18, SSIM 0.804, LPIPS 0.138 [S1].
    *   360 Dataset Mono-Scale, 1/4 Res: PSNR 25.82, SSIM 0.786, LPIPS 0.142 [S1].
    *   360 Dataset Mono-Scale, 1/8 Res: PSNR 24.38, SSIM 0.737, LPIPS 0.167 [S1].
    *   With Distance-Aware Coarse-To-Fine: PSNR 28.26, SSIM 0.803, LPIPS 0.155 [S1].
*   **Strengths:** Intrinsic continuous LOD without explicit supervision; robust to varying observation distances; dynamic resolution extensibility; fast training [S1].
*   **Limitations:** Requires careful tuning of the contraction function and LOD calculation; performance may degrade if the grid hierarchy is not sufficiently deep for very high-resolution novel views [S1].
*   **When to use:** When anti-aliasing across multiple resolutions is required, or when the observation distance varies significantly between training and novel views [S1].
*   **When not to use:** If the scene is bounded and observation distances are fixed, simpler methods like [AdaNeRF](6195-adanerf.md) might suffice.

## Background
RING-NeRF was proposed by D. Petit et al. in the paper "RING-NeRF: Rethinking Inductive Biases for Versatile and Efficient Neural Fields" [S1]. It addresses limitations of vanilla NeRF and I-NGP regarding adaptability to various scene natures, robustness to varying observation distances, and reconstruction extensibility [S1]. It builds on the concept of implicit neural representations and grid-based acceleration [S1].

## Sources
[S1] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
