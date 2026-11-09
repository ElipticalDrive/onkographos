# RING-NeRF

Also called: Distance-Aware Coarse-To-Fine, RING-NeRF pipeline, Residual Implicit Neural Grids

## Overview
RING-NeRF is a neural field architecture that introduces two primary inductive biases: a continuous multi-scale scene representation and a decoder latent space invariant to spatial translations and scale changes [S1]. Unlike sibling methods such as [Zip-NeRF](8663-zip-nerf.md) or [PyNeRF](8663-pynerf.md) which require explicit LOD supervision, RING-NeRF produces intrinsic continuous Level of Detail (LOD) without specific supervision, enabling robust generalization to novel observation distances [S1]. It distinguishes itself from [I-NGP](8585-specific-model-architectures.md)-based architectures by using summation rather than concatenation for multi-resolution features, ensuring the decoder latent space remains independent of the grid hierarchy size [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera pose (position and viewing direction $\mathbf{d}$), image pixel coordinates.
*   **Outputs:** Rendered pixel color $C(\mathbf{r})$.

**Data Structures:**
1.  **Unbounded Hierarchical 3D Grid:** A pyramid of $N$ grids $G_0, \dots, G_{N-1}$.
    *   $G_0$ is a base grid with resolution $b$.
    *   $G_i$ has resolution $f^i b$, where $f$ is the growth factor.
    *   Each grid stores latent feature vectors of size $D_{feat}$ (typically 8).
    *   The grids are stored as hash tables or dense arrays depending on implementation, but logically form a hierarchy where $G_N$ refines $G_{N-1}$.
2.  **Decoder MLP:** A small Multi-Layer Perceptron.
    *   Input dimension: $D_{feat} + D_{SH}$, where $D_{SH}$ is the dimension of Spherical Harmonics encoding.
    *   Hidden layers: Typically 3 layers.
    *   Output: Density $\sigma$ and color $\mathbf{c}$.

**Numbered Steps:**

1.  **Ray Casting and Sampling:**
    For a given pixel and camera pose, cast a ray $\mathbf{r}$. Sample $N_{samples}$ points $\mathbf{x}_i$ along the ray. The number of samples is typically 128 [S1].

2.  **Distance-Aware Forward Mapping (LOD Calculation):**
    For each sample point $\mathbf{x}_i$ at distance $d$ from the camera:
    a. Compute the Jacobian $J(\mathbf{x}_i)$ of the scene contraction function at $\mathbf{x}_i$. If no contraction is used, $J$ is the identity matrix.
    b. Calculate the Level of Detail $L_i$ using the formula:
       $$
       L_i = - \frac {\log \left (d \cdot c \cdot b \cdot \sqrt [3]{\operatorname {det}(J(\mathbf{x}_i))}\right )}{\log (f)}
       $$
       where:
       *   $d$ is the distance from the camera to the sample.
       *   $c$ is the pixel size at distance 1 (derived from image resolution and FOV).
       *   $b$ is the base grid resolution.
       *   $f$ is the grid growth factor.
       *   $J(\mathbf{x}_i)$ is the Jacobian of the contraction function.
    c. Clamp $L_i$ to the range $[0, N-1]$ or the current maximal LOD $l$ if using coarse-to-fine optimization.

3.  **Feature Aggregation (Residual Implicit Neural Grids):**
    Compute the latent feature vector $\mathbf{v}_i$ for sample $\mathbf{x}_i$ by summing interpolated features from all grid levels $k \le L_i$:
    $$
    \mathbf{v}_i = \sum_{k=0}^{\lfloor L_i \rfloor} \text{Interpolate}(G_k, \mathbf{x}_i)
    $$
    *   $\text{Interpolate}(G_k, \mathbf{x}_i)$ performs tri-linear interpolation of the latent features in grid $G_k$ at position $\mathbf{x}_i$.
    *   Note: The sources specify that using summation is critical; concatenation leads to decreased performance in generalization [S1].
    *   If $L_i$ is non-integer, the contribution of the finest level is weighted by $\alpha = L_i - \lfloor L_i \rfloor$. However, the primary implementation described sums integer levels up to the clamped LOD. The continuous LOD formula $G_L \leftarrow S(G_{N-1}) + \alpha \delta_N$ implies the feature is a weighted residual sum. In practice, the implementation sums the interpolated values from grids $0$ to $\lfloor L_i \rfloor$.

4.  **Normalization:**
    Apply standard, non-learnable normalization (e.g., LayerNorm or InstanceNorm) to the aggregated feature vector $\mathbf{v}_i$ to improve convergence stability [S1].

5.  **Feature Expansion (RFF):**
    Project the normalized feature $\mathbf{v}_i$ into a higher-dimensional space using a Random Fourier Feature (RFF) mapping:
    $$
    \mathbf{z}_i = \sin(\mathbf{W} \mathbf{v}_i)
    $$
    where $\mathbf{W}$ is a learnable linear matrix (without bias) acting as a frequency filter [S1].

6.  **Direction Encoding:**
    Encode the viewing direction $\mathbf{d}$ using Spherical Harmonics (S.H.) encoding to get $\mathbf{e}_{SH}(\mathbf{d})$ [S1].

7.  **Decoding:**
    Concatenate the expanded feature $\mathbf{z}_i$ and the direction encoding $\mathbf{e}_{SH}(\mathbf{d})$:
    $$
    \mathbf{input}_{MLP} = [\mathbf{z}_i; \mathbf{e}_{SH}(\mathbf{d})]
    $$
    Pass $\mathbf{input}_{MLP}$ through the 3-layer MLP to predict density $\sigma_i$ and color $\mathbf{c}_i$ [S1].
    *   *Note:* If performing SDF-based reconstruction, convert the SDF output to density using a NEUS-like approach before volume rendering [S1].

8.  **Volume Rendering:**
    Compute the pixel color $C(\mathbf{r})$ using the standard differentiable volume rendering equation:
    $$
    C(\mathbf{r}) = \sum_{i=1}^{N_{samples}} T_i \alpha_i \mathbf{c}_i
    $$
    where:
    $$
    \alpha_i = 1 - \exp(-\sigma_i \delta_i)
    $$
    $$
    T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
    $$
    and $\delta_i$ is the distance between adjacent samples [S1].

9.  **Loss Calculation:**
    Minimize the Mean Squared Error (MSE) loss between the rendered color $\hat{C}(\mathbf{r})$ and the ground truth pixel color $C(\mathbf{r})$:
    $$
    \mathcal{L} = ||\hat{C}(\mathbf{r}) - C(\mathbf{r})||_2
    $$
    For SDF reconstruction, an additional Eikonal loss is optimized [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn

class RINGNeRF:
    def __init__(self, base_res=16, growth_factor=2, num_levels=5, feat_dim=8, 
                 sh_degree=3, hidden_dim=64, contraction_func=None):
        """
        Initialize RING-NeRF.
        :param base_res: Resolution of the coarsest grid (b).
        :param growth_factor: Factor by which resolution increases per level (f).
        :param num_levels: Number of grid levels (N).
        :param feat_dim: Dimension of latent features in each grid.
        :param sh_degree: Degree of Spherical Harmonics.
        :param hidden_dim: Hidden dimension of the decoder MLP.
        :param contraction_func: Function to contract space (e.g., standard NeRF contraction).
        """
        self.base_res = base_res
        self.growth_factor = growth_factor
        self.num_levels = num_levels
        self.feat_dim = feat_dim
        self.contraction_func = contraction_func
        
        # Initialize hierarchical grids
        # Each grid stores features of shape [resolution, resolution, resolution, feat_dim]
        # In practice, these are often hash grids, but for clarity we assume dense or accessible structure
        self.grids = nn.ModuleList([
            nn.Embedding(base_res**3, feat_dim) for _ in range(num_levels)
        ])
        
        # RFF Matrix W
        self.rff_matrix = nn.Parameter(torch.randn(feat_dim, feat_dim))
        
        # Spherical Harmonics Encoding (simplified placeholder for SH)
        self.sh_encoder = SphericalHarmonicsEncoder(sh_degree)
        
        # Decoder MLP
        # Input: feat_dim (from RFF) + SH_dim
        sh_dim = (sh_degree + 1)**2 * 3
        self.decoder = nn.Sequential(
            nn.Linear(feat_dim + sh_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 4) # 1 for sigma, 3 for color
        )
        
        # Normalization layer
        self.norm = nn.LayerNorm(feat_dim)

    def get_grid_resolution(self, level):
        return int(self.base_res * (self.growth_factor ** level))

    def interpolate_grid(self, grid, points, level):
        """
        Tri-linear interpolation of features from a grid.
        :param grid: The grid embedding or tensor.
        :param points: Coordinates normalized to [0, 1] or grid-specific range.
        :param level: Grid level index.
        :return: Interpolated features.
        """
        # Implementation detail: Map points to grid coordinates
        res = self.get_grid_resolution(level)
        # Normalize points to [0, res-1]
        grid_coords = points * (res - 1)
        
        # Perform tri-linear interpolation
        # This is a placeholder for the actual interpolation logic
        # which depends on whether grids are dense tensors or hash tables.
        # For dense grids:
        features = torch.nn.functional.grid_sample(
            grid.weight.view(1, 1, res, res, res, self.feat_dim).permute(0, 5, 1, 2, 3, 4),
            grid_coords.unsqueeze(0).unsqueeze(0).unsqueeze(0),
            mode='trilinear',
            padding_mode='zeros',
            align_corners=True
        ).squeeze()
        return features

    def compute_lod(self, distance, contraction_jacobian_det):
        """
        Compute the Level of Detail (LOD) for a sample.
        :param distance: Distance from camera to sample.
        :param contraction_jacobian_det: Determinant of the Jacobian of the contraction function.
        :return: LOD value L.
        """
        # c: pixel size at distance 1. 
        # This depends on image resolution and FOV. 
        # Assuming c is passed or computed externally.
        # For this implementation, we assume c is a constant or passed in.
        # Let's assume c is available in the context or passed as an argument.
        # Since it's not a class attribute, we'll pass it in the forward pass or assume it's known.
        # Here, we'll assume c is passed to the forward method.
        pass 

    def forward(self, points, directions, pixel_size_at_1, max_lod=None):
        """
        Forward pass for a batch of points.
        :param points: [B, 3] 3D coordinates.
        :param directions: [B, 3] Viewing directions.
        :param pixel_size_at_1: Scalar c.
        :param max_lod: Optional clamp for coarse-to-fine training.
        :return: Density and Color.
        """
        # 1. Compute LOD
        if self.contraction_func:
            # Compute Jacobian determinant at points
            # This requires the specific contraction function implementation
            # Assuming a helper function exists
            jacobian_det = compute_jacobian_det(points, self.contraction_func)
        else:
            jacobian_det = 1.0
            
        lod = -math.log(points.norm(dim=1) * pixel_size_at_1 * self.base_res * jacobian_det**(1/3)) / math.log(self.growth_factor)
        
        if max_lod is not None:
            lod = torch.clamp(lod, max=max_lod)
        lod = torch.clamp(lod, min=0, max=self.num_levels - 1)
        
        # 2. Feature Aggregation
        aggregated_features = torch.zeros(points.shape[0], self.feat_dim, device=points.device)
        
        for level in range(self.num_levels):
            if level <= lod:
                # Interpolate features from this level
                # Note: lod is continuous, but the summation is over integer levels <= floor(lod)
                # The continuous aspect is handled by the LOD selection logic in the paper's description
                # of "weighted sum", but the pseudocode in the paper implies summing up to floor(L).
                # We will sum features from all levels k <= floor(lod).
                if level <= int(lod):
                    features = self.interpolate_grid(self.grids[level], points, level)
                    aggregated_features += features
                    
        # 3. Normalization
        normalized_features = self.norm(aggregated_features)
        
        # 4. RFF Expansion
        rff_features = torch.sin(normalized_features @ self.rff_matrix.T)
        
        # 5. Direction Encoding
        sh_features = self.sh_encoder(directions)
        
        # 6. Concatenation and Decoding
        input_vec = torch.cat([rff_features, sh_features], dim=1)
        output = self.decoder(input_vec)
        
        sigma = output[:, 0]
        color = output[:, 1:]
        
        return sigma, color

class SphericalHarmonicsEncoder(nn.Module):
    def __init__(self, degree):
        super().__init__()
        self.degree = degree
        
    def forward(self, directions):
        # Simplified SH encoding
        # Real implementation involves calculating SH basis functions
        # For the purpose of this article, we assume a standard SH encoder
        # is available or implemented separately.
        # This is a placeholder.
        return directions # Placeholder for actual SH encoding
```

**Parameters:**
*   $b$: Base grid resolution (e.g., 16).
*   $f$: Growth factor (e.g., 2).
*   $N$: Number of grid levels (e.g., 5 or 9).
*   $D_{feat}$: Feature size (e.g., 8).
*   $l_0$: Initial maximal LOD for coarse-to-fine training.
*   $n_{ctf}$: Hyperparameter for the speed of the coarse-to-fine scheduler.
*   $N_{samples}$: Number of samples per ray (e.g., 128).
*   MLP Hidden Dimension: 64 (for few-view setups), typically higher for full-resolution NVS.

**Edge Cases and Pitfalls:**
*   **Concatenation vs. Summation:** Using concatenation for multi-resolution features instead of summation leads to decreased performance in generalization to novel unseen observation distances [S1].
*   **Decoder Latent Space Invariance:** The decoder must not be conditioned on position or scale. Concatenating positional encoding with grid features breaks this invariance [S1].
*   **Unobserved Samples:** During novel view rendering, discard samples not observed by any training view to remove noise [S1].
*   **Scene Contraction:** If using scene contraction, the Jacobian determinant must be correctly computed for the LOD formula. The sources note that disabling scene contraction and distortion loss is an option for certain datasets [S1].

## Operating characteristics
*   **Time Complexity:** Training time is significantly faster than many dedicated architectures. On the 360 Dataset Mono-Scale setup, RING-NeRF achieves 28.09 PSNR in 0.45 hours on one Nvidia-A100 GPU, compared to 21.69 hours for Mip-NeRF 360 [S1].
*   **Memory Complexity:** The hierarchical grid structure allows for dynamic resolution increase. The '3+1+1 levels' configuration (adding grids to a pre-trained model) results in the same quality as a '5 levels' configuration but with potentially better memory efficiency during incremental training [S1].
*   **Scaling:** The architecture supports resolution-adaptive reconstruction. The maximal LOD can be increased continuously during training using the scheduler $l = (l_0 + n_{ctf}^n)$ [S1].
*   **Strengths:**
    *   Intrinsic continuous LOD without explicit supervision.
    *   Robust to varying observation distances (anti-aliasing).
    *   Efficient training times.
    *   Extensible resolution (dynamic grid addition).
*   **Limitations:**
    *   Requires careful tuning of the LOD calculation parameters ($c, b, f$).
    *   Performance depends on the correct implementation of the distance-aware forward mapping.
    *   May require disabling scene contraction for certain datasets to avoid artifacts [S1].

## Background
RING-NeRF was proposed by D. Petit et al. in the paper "RING-NeRF: Rethinking Inductive Biases for Versatile and Efficient Neural Fields" [S1]. The work addresses limitations of NeRF-based reconstruction regarding adaptability to varying observation distances and reconstruction extensibility. It builds on the idea of injecting generic priors (inductive biases) into the NeRF architecture, specifically a continuous multi-scale representation and decoder latent space invariance. The method is designed to be simple, versatile, and efficient, competing with state-of-the-art solutions on tasks like anti-aliasing and few-view reconstruction.

## Sources
[S1] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
