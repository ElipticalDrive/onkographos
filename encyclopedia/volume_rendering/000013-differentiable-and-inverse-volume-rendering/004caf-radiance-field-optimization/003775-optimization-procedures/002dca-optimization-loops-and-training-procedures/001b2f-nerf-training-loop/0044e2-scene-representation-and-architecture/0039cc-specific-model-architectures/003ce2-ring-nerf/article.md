# RING-NeRF

Also called: Distance-Aware Coarse-To-Fine, RING-NeRF pipeline, Residual Implicit Neural Grids

## Overview
RING-NeRF is a neural field architecture that introduces two primary inductive biases: a continuous multi-scale scene representation and a decoder latent space invariant to spatial translations and scale changes [S1]. Unlike sibling methods such as [Zip-NeRF](8663-nerf-rpn.md) or [Mip-NeRF](8875-multiplanenerf.md) which require explicit supervision for Level of Detail (LOD) or modify features based on observation distance, RING-NeRF produces intrinsic continuous LOD without supervision, enabling robust generalization to novel, unseen observation distances [S1]. It distinguishes itself from [I-NGP](8953-pool-proposer-architecture.md) by using summation rather than concatenation for multi-resolution features, ensuring the decoder latent space remains invariant to LOD, and supports dynamic resolution extension during training [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera poses, image resolutions, field of view (FOV), and training images.
*   **Outputs:** Rendered pixel colors $\hat{C}(\mathbf{r})$ and optimized scene representation parameters (grid deviations $\delta_i$ and decoder weights).

**Data Structures:**
1.  **Unbounded Hierarchical 3D Grid:** A pyramid of $N$ grids. The resolution of the $i$-th grid is $f^i b$, where $b$ is the base resolution and $f$ is the growth factor.
    *   Each grid stores latent feature vectors of size $D_{feat}$ (typically 8) at each voxel.
    *   The grids are not independent; they form a recursive mapping function.
2.  **Decoder:** A small Multi-Layer Perceptron (MLP) and normalization layers.
    *   **Normalization Layer:** Standard, non-learnable normalization applied to the combined feature vector.
    *   **Random Fourier Feature (RFF) Layer:** A linear transformation followed by a sinusoidal activation $y = \sin(\mathbf{W}x)$, where $\mathbf{W}$ is a learnable matrix without bias. This projects the feature into a higher-dimensional space.
    *   **Density/Color Head:** A linear layer transforms the RFF output into a density (or SDF) value and a single color feature vector.
    *   **Color MLP:** The color feature is concatenated with the view direction encoded via Spherical Harmonics (S.H.). This combined vector is fed into a 3-layer MLP to predict the final radiance color $\mathbf{c}$.

**Numbered Steps:**

1.  **Scene Representation (Recursive Grid Definition):**
    The scene is represented by a mapping function defined recursively. Let $G_N$ be the mapping at level $N$.
    $$
    G_0 \leftarrow \delta_0
    $$
    $$
    G_N \leftarrow S(G_{N-1}) + \delta_N
    $$
    where $\delta_i$ is the latent feature grid at level $i$, and $S$ is a subdivision scheme that increases the resolution of $G_{N-1}$ to match the resolution of $G_N$ via tri-linear interpolation. The latent vector for a point $\mathbf{x}$ at level $N$ is obtained by interpolating linearly in the cell for each level $k \le N$ and summing the results. This summation acts as a residual connection [S1].

2.  **Continuous LOD Representation:**
    To allow continuous LOD $L \in [N-1, N]$, a weighting factor $\alpha = L - (N-1)$ is introduced:
    $$
    G_L \leftarrow S(G_{N-1}) + \alpha \delta_N
    $$
    This allows the model to represent details at fractional levels of detail between integer grid resolutions [S1].

3.  **Distance-Aware Forward Mapping (LOD Selection):**
    For a sample at distance $d$ from the camera, the appropriate LOD $L$ is determined by equating the volume of the cone cast pixel in contracted space to the volume of a virtual grid cell at LOD $L$.
    *   Let $c$ be the pixel size at distance 1 (derived from image resolution and FOV).
    *   Let $J(p)$ be the Jacobian of the scene contraction function at the sample's location $p$.
    *   The volume of the cone in contracted space is $V_{contract} = (dc)^3 \det(J(p))$.
    *   The volume of a cell in a virtual grid of LOD $L$ with base resolution $b$ and growth factor $f$ is $(\frac{1}{f^L b})^3$.
    *   Equating these volumes solves for $L$:
        $$
        L = - \frac {\log \left (dcb \sqrt [3]{\operatorname {det}(J(p))}\right )}{\log (f)}
        $$
    *   If scene contraction is disabled, $\det(J(p)) = 1$ [S1].

4.  **Feature Aggregation:**
    Using the calculated continuous LOD $L$, the latent feature $\mathbf{f}$ for a sample is computed as a weighted sum of the grid hierarchy up to level $L$. Specifically, for each level $k \le L$, the feature is interpolated from the grid $G_k$ and summed. This ensures the decoder latent space is invariant to position and scale, as LOD is exclusively encoded in the grid [S1].

5.  **Decoding:**
    *   The aggregated feature $\mathbf{f}$ is passed through the normalization layer.
    *   It is transformed by the RFF layer: $\mathbf{f}_{rff} = \sin(\mathbf{W} \mathbf{f}_{norm})$.
    *   A linear layer maps $\mathbf{f}_{rff}$ to density $\sigma$ (or SDF) and a color feature $\mathbf{c}_{feat}$.
    *   For SDF-based reconstruction, the SDF output is converted to density using a NEUS-like approach before volume rendering [S1].

6.  **View Direction Encoding:**
    The viewing direction $\mathbf{d}$ is encoded using Spherical Harmonics (S.H.) encoding. This encoded direction is concatenated with the color feature $\mathbf{c}_{feat}$ [S1].

7.  **Radiance Prediction:**
    The concatenated vector is fed into a 3-layer MLP to predict the final color $\mathbf{c}$ [S1].

8.  **Volume Rendering:**
    Rays are sampled at $N$ locations along the ray. The rendered color $\hat{C}(\mathbf{r})$ is computed via standard volume rendering:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=0}^{N-1} T_i (1 - \exp(-\sigma_i \delta_i)) c_i
    $$
    where $T_i = \exp(\sum_{j=0}^{i-1} \sigma_j \delta_j)$, $\sigma_i$ is density, $c_i$ is color, and $\delta_i$ is the distance between samples [S1].

9.  **Optimization (Continuous Coarse-To-Fine):**
    *   The model minimizes the Mean Squared Error (MSE) loss: $||\hat{C}(\mathbf{r}) - C(\mathbf{r})||_2$.
    *   To improve convergence, a continuous coarse-to-fine strategy is used. During cone casting, samples' LOD is clamped up to a maximal LOD $l$. Grids of level $l$ and above are set to zero and not optimized.
    *   The maximal LOD $l$ increases continuously with the epoch $n$:
        $$
        l = l_0 + n_{ctf}^n
        $$
        where $l_0$ is the initial number of grids and $n_{ctf}$ is a hyperparameter controlling the speed of the process [S1].
    *   This approach keeps all dimensions of the decoder latent space supervised, unlike concatenation-based methods where zeroed grids leave dimensions unsupervised [S1].

10. **Dynamic Resolution Extension:**
    The grid hierarchy can be extended dynamically. For example, after training a 3-level configuration, a new grid of higher resolution can be added. The new grid is trained while the previous grids and the decoder are frozen. This allows unbounded LOD without retraining the decoder [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.nn.functional as F

class RINGNeRF:
    def __init__(self, base_res=16, growth_factor=2, num_levels=5, feat_dim=8, 
                 num_sh_bases=4, mlp_hidden_dim=64):
        """
        Initialize RING-NeRF.
        :param base_res: Resolution of the coarsest grid (b).
        :param growth_factor: Factor by which resolution increases per level (f).
        :param num_levels: Number of grid levels (N).
        :param feat_dim: Dimension of latent features in grids.
        :param num_sh_bases: Number of spherical harmonic bases for view direction.
        :param mlp_hidden_dim: Hidden dimension for the color MLP.
        """
        self.base_res = base_res
        self.growth_factor = growth_factor
        self.num_levels = num_levels
        self.feat_dim = feat_dim
        self.num_sh_bases = num_sh_bases
        
        # Initialize hierarchical grids
        # Grid i has resolution (base_res * growth_factor^i)
        self.grids = nn.ModuleList()
        for i in range(num_levels):
            res = base_res * (growth_factor ** i)
            # Grid stores features. Shape: [res, res, res, feat_dim]
            # Initialized to zeros or small random values
            grid = nn.Parameter(torch.zeros(res, res, res, feat_dim))
            self.grids.append(grid)
            
        # Decoder components
        # RFF Layer: Linear + Sin
        self.rff_weight = nn.Parameter(torch.randn(feat_dim, feat_dim)) # W in sin(Wx)
        
        # Linear layer to map RFF output to density and color feature
        self.head_linear = nn.Linear(feat_dim, 2) # 1 for density, 1 for color feature dim
        
        # Color MLP
        # Input: color_feature_dim + sh_encoded_dim
        # Output: 3 (RGB)
        sh_encoded_dim = num_sh_bases * 2 # sin and cos components per base
        color_input_dim = 1 + sh_encoded_dim # 1 color feature + SH
        
        self.color_mlp = nn.Sequential(
            nn.Linear(color_input_dim, mlp_hidden_dim),
            nn.ReLU(),
            nn.Linear(mlp_hidden_dim, mlp_hidden_dim),
            nn.ReLU(),
            nn.Linear(mlp_hidden_dim, 3)
        )
        
        # Normalization layer (standard, non-learnable)
        self.norm = nn.LayerNorm(feat_dim)

    def get_grid_resolution(self, level):
        return self.base_res * (self.growth_factor ** level)

    def interpolate_grid(self, grid, coords):
        """
        Trilinear interpolation of a grid at given coordinates.
        :param grid: [R, R, R, D]
        :param coords: [N, 3] normalized to [0, 1]
        :return: [N, D]
        """
        # Scale coords to grid dimensions
        R = grid.shape[0]
        scaled_coords = coords * (R - 1)
        # Use grid_sample or manual interpolation
        # Assuming coords are in [0, 1] range relative to grid size
        # Convert to [-1, 1] for grid_sample
        norm_coords = 2.0 * scaled_coords / (R - 1) - 1.0
        # grid_sample expects [N, 1, 1, 3] for 3D
        norm_coords = norm_coords.unsqueeze(1).unsqueeze(1)
        
        # grid_sample input: [N, C, H, W, D] -> [N, 1, R, R, R, D]
        grid_expanded = grid.unsqueeze(0).unsqueeze(0)
        
        # Interpolate
        # Note: grid_sample uses normalized coordinates in [-1, 1]
        # We need to handle the coordinate system carefully.
        # Here we assume standard PyTorch grid_sample behavior.
        # For simplicity, we assume a helper function or standard grid_sample usage.
        # In practice, one might use F.grid_sample with align_corners=False.
        
        # Reshape grid for grid_sample: [1, D, R, R, R]
        grid_for_sample = grid.permute(3, 0, 1, 2).unsqueeze(0)
        
        # grid_sample output: [1, D, 1, 1, 1] -> [D]
        # We need to process N points.
        # This is a simplified representation. In implementation, batch processing is used.
        interpolated = F.grid_sample(grid_for_sample, norm_coords, mode='trilinear', align_corners=False)
        return interpolated.squeeze(-1).squeeze(-1).squeeze(-1).permute(1, 0)

    def compute_feature(self, coords, lod):
        """
        Compute the aggregated feature for a point given its continuous LOD.
        :param coords: [N, 3] normalized coordinates in [0, 1]
        :param lod: [N] continuous LOD values
        :return: [N, feat_dim]
        """
        total_feature = torch.zeros_like(coords[:, :1].repeat(1, self.feat_dim))
        
        for level in range(self.num_levels):
            # Determine if this level contributes
            # Level k contributes if k <= lod
            # If lod is continuous, we interpolate between levels
            
            # For simplicity, we sum contributions from all levels k <= floor(lod)
            # and partial contribution from floor(lod)+1 if needed.
            # The paper describes: "interpolating linearly the point in the cell for each level inferior or equal to N, and summing the results."
            # And for continuous LOD: G_L = S(G_{N-1}) + alpha * delta_N
            
            # Implementation of continuous LOD aggregation:
            # For each sample, we determine which grids to use.
            # A simpler approach matching the "summing" description:
            # Sum features from all grids k where k <= lod.
            
            # However, the recursive definition G_N = S(G_{N-1}) + delta_N implies
            # that the feature at level N includes the upsampled feature from N-1.
            # So, G_L effectively sums delta_0 + delta_1 + ... + delta_L (with interpolation).
            
            # Let's implement the sum of interpolated features from all levels k <= L
            # where L is the continuous LOD.
            
            # If lod < level, contribution is 0? 
            # The paper says: "interpolating ... for each level inferior or equal to N".
            # And "continuous LOD ... weighting factor alpha".
            
            # Let's assume we sum features from all levels k such that k <= lod.
            # If lod is 1.3, we sum level 0 and level 1.
            # Level 1 contribution is weighted by alpha = 1.3 - 1 = 0.3?
            # Or does the recursive structure handle it?
            # G_1.3 = S(G_0) + 0.3 * delta_1.
            # G_0 = delta_0.
            # So Feature = delta_0 + 0.3 * delta_1.
            
            # Generalizing:
            # Feature = sum_{k=0}^{floor(L)-1} delta_k + (L - floor(L)) * delta_{floor(L)} ?
            # No, the recursion is G_N = S(G_{N-1}) + delta_N.
            # So G_1 = S(G_0) + delta_1 = S(delta_0) + delta_1.
            # G_1.3 = S(G_0) + 0.3 * delta_1 = S(delta_0) + 0.3 * delta_1.
            # This implies that the feature is the sum of the upsampled coarser grids plus the weighted finer grid.
            
            # To implement this efficiently:
            # We can compute the feature for the integer level floor(L) and then add the fractional part.
            
            pass 
            
        # Simplified implementation for the article:
        # Sum features from all levels k <= floor(lod), upsampled to the finest resolution if necessary?
        # No, the features are summed in the latent space.
        # The paper says: "interpolating linearly the point in the cell for each level inferior or equal to N, and summing the results."
        # This suggests we interpolate each grid at the point's location and sum them.
        # But the grids have different resolutions.
        # The recursive definition G_N = S(G_{N-1}) + delta_N suggests that the "value" at level N is the upsampled value from N-1 plus the residual delta_N.
        # So, to get the feature at LOD L, we sum the residuals delta_k for k <= L, with appropriate weighting for the fractional part.
        
        # Let's implement the sum of residuals.
        # For a sample with LOD L:
        # Feature = sum_{k=0}^{floor(L)-1} interpolate(delta_k, coords) + (L - floor(L)) * interpolate(delta_{floor(L)}, coords)
        # Wait, the recursion is G_N = S(G_{N-1}) + delta_N.
        # So G_0 = delta_0.
        # G_1 = S(G_0) + delta_1.
        # G_1.3 = S(G_0) + 0.3 * delta_1.
        # So the feature is S(G_0) + 0.3 * delta_1.
        # S(G_0) is the upsampled delta_0.
        # So Feature = interpolate_upsampled(delta_0) + 0.3 * interpolate(delta_1).
        
        # This requires upsampling coarser grids to the finest resolution before summing?
        # Or summing in the latent space? The latent space dimension is constant.
        # The interpolation is spatial.
        # So, for each level k <= floor(L), we interpolate delta_k at the point's location.
        # But delta_k is at a lower resolution.
        # The term S(G_{N-1}) implies that the feature from level N-1 is upsampled to the resolution of level N.
        # So, effectively, we are summing the interpolated features from all levels k <= L,
        # where the feature from level k is interpolated at the point's location.
        # But wait, if we just interpolate delta_k, we are not accounting for the S(G_{k-1}) part?
        # The recursive definition builds the feature.
        # G_L is the feature.
        # G_L = S(G_{L-1}) + alpha * delta_L.
        # Expanding this:
        # G_L = S(S(...S(G_0)...)) + ... + alpha * delta_L.
        # This means the feature is the sum of the upsampled residuals.
        # So, Feature = sum_{k=0}^{floor(L)-1} interpolate_upsampled_to_L(delta_k) + (L - floor(L)) * interpolate_upsampled_to_L(delta_{floor(L)})?
        # No, the interpolation is done at the point's location.
        # The "upsampling" S is a spatial operation.
        # So, for a point p, the feature from level k is the value of the grid G_k at p.
        # But G_k is defined recursively.
        # G_k(p) = G_{k-1}(p) + delta_k(p) ? No, S(G_{k-1}) is upsampled.
        # So G_k(p) = interpolate_upsampled(G_{k-1}, p) + delta_k(p).
        # This implies G_k(p) = sum_{j=0}^k delta_j(p) (with upsampling for j < k).
        # So, to compute G_L(p), we sum the interpolated values of delta_j for all j <= L.
        # The interpolation for delta_j is done at the resolution of level L?
        # No, the interpolation is just spatial interpolation of the grid value.
        # The "upsampling" S is implicit in the recursive definition.
        # So, Feature(p) = sum_{j=0}^{floor(L)-1} interpolate(delta_j, p) + (L - floor(L)) * interpolate(delta_{floor(L)}, p).
        # Where interpolate(delta_j, p) is the trilinear interpolation of grid delta_j at point p.
        # This seems to be the correct interpretation of "summing the results" of interpolation.
        
        for k in range(self.num_levels):
            if k < lod:
                # Interpolate delta_k at coords
                feat_k = self.interpolate_grid(self.grids[k], coords)
                total_feature += feat_k
            elif k == math.floor(lod):
                alpha = lod - math.floor(lod)
                feat_k = self.interpolate_grid(self.grids[k], coords)
                total_feature += alpha * feat_k
            else:
                break
                
        return total_feature

    def encode_view_direction(self, dirs):
        """
        Encode view direction using Spherical Harmonics.
        :param dirs: [N, 3]
        :return: [N, sh_encoded_dim]
        """
        # Standard SH encoding
        # For simplicity, we use a basic sin/cos encoding if SH library is not available
        # Or assume a helper function
        # Here we assume a standard SH encoding function
        # sh_encoded = spherical_harmonics(dirs, self.num_sh_bases)
        # For implementation, we can use a simple positional encoding for direction
        # But the paper specifies S.H.
        # We will assume a placeholder for SH encoding
        sh_encoded = torch.zeros(dirs.shape[0], self.num_sh_bases * 2)
        # In practice, use a library or implement SH basis functions
        return sh_encoded

    def forward(self, coords, lod, dirs):
        """
        Forward pass.
        :param coords: [N, 3] normalized coordinates
        :param lod: [N] continuous LOD values
        :param dirs: [N, 3] view directions
        :return: density, color
        """
        # 1. Compute feature
        feature = self.compute_feature(coords, lod)
        
        # 2. Normalize
        feature_norm = self.norm(feature)
        
        # 3. RFF
        rff_out = torch.sin(feature_norm @ self.rff_weight.T)
        
        # 4. Head Linear
        head_out = self.head_linear(rff_out)
        density = head_out[:, 0]
        color_feat = head_out[:, 1:]
        
        # 5. View Direction Encoding
        sh_encoded = self.encode_view_direction(dirs)
        
        # 6. Concatenate and MLP
        color_input = torch.cat([color_feat, sh_encoded], dim=-1)
        color = self.color_mlp(color_input)
        
        return density, color

    def render_ray(self, ray_origins, ray_dirs, near, far, num_samples, lod_values):
        """
        Render a single ray.
        """
        # Sample points along the ray
        t_vals = torch.linspace(0., 1., num_samples)
        z_vals = near * (1. - t_vals) + far * t_vals
        points = ray_origins.unsqueeze(0) + ray_dirs.unsqueeze(0) * z_vals.unsqueeze(1)
        
        # Compute density and color
        density, color = self.forward(points, lod_values, ray_dirs)
        
        # Volume Rendering
        # delta = z_vals[i+1] - z_vals[i]
        deltas = z_vals[1:] - z_vals[:-1]
        deltas = torch.cat([deltas, torch.tensor([1e10]).to(deltas.device)], dim=0)
        
        alpha = 1.0 - torch.exp(-density * deltas)
        transmittance = torch.cumprod(torch.exp(-density * deltas), dim=0)
        transmittance = torch.cat([torch.ones_like(transmittance[:, :1]), 1.0 - transmittance], dim=1)
        
        weights = alpha * transmittance
        rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=0)
        
        return rendered_color
```

**Parameters:**
*   $c$: Pixel size at distance 1, calculated from image resolution and camera FOV [S1].
*   $J$: Jacobian of the contraction function at the sample's location $p$ [S1].
*   $f$: Growth factor for the grid hierarchy resolution (e.g., 2) [S1].
*   $l_0$: Initial number of grids used in coarse-to-fine optimization [S1].
*   $n_{ctf}$: Hyperparameter describing the speed of the coarse-to-fine process [S1].
*   Number of samples per ray: 128 [S1].
*   Hashgrid configuration: Pyramid of 9 grids with a growth factor of 2 (in some experiments) [S1].
*   Feature size: 8 [S1].
*   MLP hidden dimension: 64 for both density and color in few-view setups [S1].
*   No appearance embedding is used [S1].

**Edge Cases and Pitfalls:**
*   **Concatenation vs. Summation:** Using concatenation instead of summation for multi-resolution features leads to decreased performance in generalization to novel unseen observation distances [S1].
*   **Decoder Latent Space:** Unlike I-NGP-based architectures, RING-NeRF keeps supervising all dimensions of the decoder latent space during coarse-to-fine optimization, preventing unsupervised dimensions when grids are zeroed [S1].
*   **Unseen Areas:** During novel view rendering, samples not observed by any training view should be discarded to remove unnecessary noise [S1].
*   **Scene Contraction:** Scene contraction and distortion loss mechanisms can be disabled depending on the scene type (e.g., object-centric scenes) [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The hierarchical grid structure allows for efficient memory usage compared to dense grids. The dynamic resolution extension allows optimizing memory footprint by only training necessary levels. Training time is significantly reduced compared to vanilla NeRF (0.45h on Nvidia-A100 for 360 Dataset Mono-Scale setup vs 12.65h for NeRF) [S1].
*   **Measured Performance:** On the 360 Dataset Mono-Scale setup at Full Res, RING-NeRF achieves PSNR 28.09, SSIM 0.799, LPIPS 0.157 [S1]. At 1/2 Res, PSNR 27.18, SSIM 0.804, LPIPS 0.138. At 1/4 Res, PSNR 25.82, SSIM 0.786, LPIPS 0.142. At 1/8 Res, PSNR 24.38, SSIM 0.737, LPIPS 0.167 [S1]. With Distance-Aware Coarse-To-Fine, it achieves 28.26 PSNR, 0.803 SSIM, 0.155 LPIPS [S1].
*   **Scaling:** The architecture supports dynamic growth of the grid pyramid size, allowing unbounded LOD. Adding new grids does not require retraining the decoder or previous grids, enabling adaptive resolution [S1].
*   **Strengths:** Intrinsic continuous LOD without supervision, robust to varying observation distances, efficient training, dynamic resolution extension, and invariant decoder latent space [S1].
*   **Limitations:** Requires careful tuning of the coarse-to-fine scheduler parameters ($l_0, n_{ctf}$). Performance may degrade if scene contraction is not handled correctly for unbounded scenes [S1].
*   **When to use:** Suitable for tasks requiring anti-aliasing, few-view reconstruction, and SDF reconstruction without scene-specific initialization. Ideal for scenarios with varying observation distances [S1].
*   **When not to use:** If explicit LOD supervision is preferred or if the scene has very limited complexity where a single grid resolution suffices, the overhead of the hierarchical structure may not be justified [S1].

## Background
RING-NeRF was proposed by D. Petit et al. in the paper "RING-NeRF: Rethinking Inductive Biases for Versatile and Efficient Neural Fields" [S1]. The work addresses limitations of existing NeRF architectures regarding adaptability to varying observation distances, robustness to limited viewpoints, and extensibility of resolution. It builds on the idea of injecting generic priors (inductive biases) into the NeRF architecture, specifically a continuous multi-scale representation and decoder latent space invariance [S1]. The architecture is designed to be simple, versatile, and efficient, competing with state-of-the-art dedicated solutions on multiple tasks [S1].

## Sources
[S1] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
