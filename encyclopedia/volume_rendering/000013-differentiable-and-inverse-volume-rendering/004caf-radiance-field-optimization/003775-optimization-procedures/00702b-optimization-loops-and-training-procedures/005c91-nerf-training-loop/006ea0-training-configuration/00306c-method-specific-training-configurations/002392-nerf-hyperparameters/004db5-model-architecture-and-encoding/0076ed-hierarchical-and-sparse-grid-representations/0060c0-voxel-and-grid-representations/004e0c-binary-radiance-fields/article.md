# Binary Radiance Fields

Also called: BRF, BiRF, Binarized Radiance Fields

## Overview
Binary Radiance Fields (BiRF) is a storage-efficient radiance field representation that constrains learnable feature grid parameters to binary values $\{-1, +1\}$, reducing model size by orders of magnitude compared to real-valued floating-point grids [S2]. It distinguishes itself from post-processing compression methods (e.g., PlenOctrees, Re:NeRF) by performing binarization directly during training via a straight-through estimator, avoiding additional optimization steps and pre-trained model performance bounds [S2]. It further employs a 2D-3D hybrid multi-resolution architecture to mitigate hash collisions inherent in 3D grids while maintaining high reconstruction quality [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images (poses and RGB), 3D coordinates $\mathbf{x}$, viewing directions $\mathbf{d}$.
*   **Outputs:** Volume density $\sigma$, color $\mathbf{c}$.

**Data Structures:**
The representation consists of four multi-resolution feature grids implemented using hash encoding [S2]:
1.  **3D Voxel Grid ($\Phi_{\theta_{xyz}}$):** $L$ levels with resolutions ranging from 16 to 1024. Each level stores up to $T_{3D}$ feature vectors.
2.  **2D Plane Grids ($\Phi_{\theta_{xy}}, \Phi_{\theta_{xz}}, \Phi_{\theta_{yz}}$):** $M$ levels with resolutions ranging from 64 to 512. Each level stores up to $T_{2D}$ feature vectors.
3.  **Feature Values:** All grid parameters are constrained to $\{-1, +1\}$, stored as 1-bit values [S2].
4.  **MLPs:**
    *   Density MLP: 1 hidden layer, 128 channels, ReLU activation.
    *   Color MLP: 2 hidden layers, 128 channels, ReLU activation.

**Parameters:**
*   $L$: Number of 3D grid levels.
*   $M$: Number of 2D grid levels.
*   $T_{3D}$: Max feature vectors per 3D level.
*   $T_{2D}$: Max feature vectors per 2D level.
*   $F$: Feature dimension per level. Variants use $F \in \{2, 4, 8\}$ [S2].
*   $\lambda_{\text{sparsity}}$: Sparsity loss weight, typically $2.0 \times 10^{-5}$ [S2].
*   Model Variants:
    *   Small (Ours-S): $\log_2(T_{3D}) = 17, \log_2(T_{2D}) = 15$.
    *   Base (Ours-B): $\log_2(T_{3D}) = 19, \log_2(T_{2D}) = 17$ [S2].

**Numbered Steps:**

1.  **Binarization (Training):**
    Maintain real-valued parameters $\theta$. During the forward pass, binarize them using the sign function:
    $$ \theta' = \text{sign}(\theta) = \begin{cases} +1 & \text{if } \theta \ge 0 \\ -1 & \text{otherwise} \end{cases} $$
    During backpropagation, use the Straight-Through Estimator (STE) to pass gradients:
    $$ \frac{\partial L}{\partial \theta} = \frac{\partial L}{\partial \theta'} \cdot \mathbb{I}(|\theta| \le 1) $$
    This constrains $\theta$ to $\{-1, 1\}$ and prevents scale divergence [S2].

2.  **3D Feature Encoding:**
    For input coordinate $\mathbf{x}$, compute features from the 3D grid by linearly interpolating the binary parameters across $L$ resolution levels and concatenating them:
    $$ \mathbf{f}_{xyz} = \left\{ \text{interp}(\mathbf{x}, \text{sign}(\Phi^l_{\theta_{xyz}})) \right\}_{l=1}^L $$
    where $\text{interp}$ denotes tri-linear interpolation [S2].

3.  **2D Feature Encoding:**
    Project $\mathbf{x}=(x,y,z)$ to 2D coordinates: $\mathbf{x}_{xy}=(x,y)$, $\mathbf{x}_{xz}=(x,z)$, $\mathbf{x}_{yz}=(y,z)$. For each axis, interpolate binary parameters across $M$ resolution levels and concatenate:
    $$ \mathbf{f}_{xy} = \left\{ \text{interp}(\mathbf{x}_{xy}, \text{sign}(\Phi^m_{\theta_{xy}})) \right\}_{m=1}^M $$
    Repeat for $\mathbf{f}_{xz}$ and $\mathbf{f}_{yz}$ using bi-linear interpolation [S2].

4.  **Feature Concatenation:**
    Combine all encoded features:
    $$ \mathbf{f} = \{ \mathbf{f}_{xyz}, \mathbf{f}_{xy}, \mathbf{f}_{xz}, \mathbf{f}_{yz} \} $$

5.  **Density Prediction:**
    Apply positional encoding $\gamma(\mathbf{x})$ to the input coordinate. Feed $\gamma(\mathbf{x})$ and $\mathbf{f}$ into the density MLP:
    $$ (\sigma, \mathbf{e}) = \text{MLP}_{\text{density}}(\gamma(\mathbf{x}), \mathbf{f}) $$
    where $\sigma$ is volume density and $\mathbf{e}$ is the embedded feature vector [S2].

6.  **Color Prediction:**
    Encode the viewing direction $\mathbf{d}$ using spherical harmonics $\mathbf{d}_{\text{enc}}$. Feed $\mathbf{e}$ and $\mathbf{d}_{\text{enc}}$ into the color MLP:
    $$ \mathbf{c} = \text{MLP}_{\text{color}}(\mathbf{e}, \mathbf{d}_{\text{enc}}) $$

7.  **Loss Calculation:**
    Compute reconstruction loss $L_{\text{recon}}$ (MSE between rendered and ground truth colors) and sparsity loss $L_{\text{sparsity}}$:
    $$ L_{\text{sparsity}} = \sum_{i,k} \log(1 + 2\sigma(\mathbf{r}_i(t_k))^2) $$
    Total loss:
    $$ L = L_{\text{recon}} + \lambda_{\text{sparsity}} L_{\text{sparsity}} $$
    Optimize using Adam [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BinaryRadianceField:
    def __init__(self, L=4, M=4, T_3D=2**17, T_2D=2**15, F=2, hash_size=2**19):
        """
        Initialize BiRF.
        L: Number of 3D grid levels.
        M: Number of 2D grid levels.
        T_3D: Max feature vectors per 3D level.
        T_2D: Max feature vectors per 2D level.
        F: Feature dimension per level.
        hash_size: Hash table size for Instant-NGP style encoding.
        """
        self.L = L
        self.M = M
        self.F = F
        self.hash_size = hash_size
        
        # Initialize hash tables for 3D grid (L levels)
        # Each level has resolution 16^3 to 1024^3, mapped to hash table
        self.grid_3d = nn.ModuleList([
            nn.Embedding(hash_size, F) for _ in range(L)
        ])
        
        # Initialize hash tables for 2D grids (3 planes * M levels)
        # Each level has resolution 64^2 to 512^2
        self.grid_2d_xy = nn.ModuleList([
            nn.Embedding(hash_size, F) for _ in range(M)
        ])
        self.grid_2d_xz = nn.ModuleList([
            nn.Embedding(hash_size, F) for _ in range(M)
        ])
        self.grid_2d_yz = nn.ModuleList([
            nn.Embedding(hash_size, F) for _ in range(M)
        ])
        
        # Initialize MLPs
        # Density MLP: Input dim = L*F + M*3*F, Hidden=128, Output=1+128 (sigma + e)
        input_dim_density = L * F + 3 * M * F
        self.mlp_density = nn.Sequential(
            nn.Linear(input_dim_density + 10, 128), # +10 for positional encoding of x
            nn.ReLU(),
            nn.Linear(128, 129) # 1 for sigma, 128 for e
        )
        
        # Color MLP: Input dim = 128 + 27 (SH), Hidden=128, Output=3
        self.mlp_color = nn.Sequential(
            nn.Linear(128 + 27, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, 3)
        )
        
        # Initialize weights
        self._init_weights()

    def _init_weights(self):
        for layer in self.grid_3d + self.grid_2d_xy + self.grid_2d_xz + self.grid_2d_yz:
            nn.init.uniform_(layer.weight, -1, 1)
        for m in self.mlp_density.modules():
            if isinstance(m, nn.Linear):
                nn.init.kaiming_normal_(m.weight, mode='fan_in', nonlinearity='relu')
        for m in self.mlp_color.modules():
            if isinstance(m, nn.Linear):
                nn.init.kaiming_normal_(m.weight, mode='fan_in', nonlinearity='relu')

    def binarize(self, x):
        """
        Binarize real-valued parameters to {-1, 1} using STE.
        """
        # Forward: sign function
        # Backward: identity if |x| <= 1, else 0
        return (x.sign() * (x.abs() <= 1).float()).detach() + x - x.detach()

    def get_hash_indices(self, coords, resolution):
        """
        Compute hash indices for coordinates at a given resolution.
        Simplified hash function based on Instant-NGP.
        """
        # coords: (N, 3)
        # resolution: int
        # Returns: (N,) indices
        # Note: In practice, this involves bit manipulation and hashing.
        # Here we assume a placeholder hash function that maps coords to indices.
        # For implementation, one would use the specific hash function from Instant-NGP.
        # We simulate it here by returning random indices for demonstration of structure.
        # In real code, use torch.ops.nvrtc or similar for efficient hashing.
        N = coords.shape[0]
        # Placeholder: In reality, this is a deterministic hash based on coords and resolution
        return torch.randint(0, self.hash_size, (N,))

    def query_3d(self, x, levels):
        """
        Query 3D grid features.
        x: (N, 3)
        levels: list of resolution levels [16, 32, ..., 1024]
        Returns: (N, L*F)
        """
        features = []
        for l, res in enumerate(levels):
            # Get hash indices for this resolution
            indices = self.get_hash_indices(x, res)
            # Get embeddings
            emb = self.grid_3d[l](indices) # (N, F)
            # Interpolate? Instant-NGP uses hash lookup directly, but BiRF uses interpolation.
            # BiRF paper says "linearly interpolating binary parameters".
            # However, hash encoding typically skips explicit interpolation by hashing neighbors.
            # The paper states: "all feature grids are implemented using a hash encoding [20]".
            # And: "tri-linearly interpolating binary parameters".
            # This implies the hash encoding is used to store the grid, but interpolation is performed on the grid values.
            # For a pure hash implementation (Instant-NGP), interpolation is over the hash table entries of the 8 neighbors.
            # We will implement the interpolation logic assuming we have the grid values.
            # Since hash encoding maps coords to indices, we need the 8 neighbors.
            # For simplicity in this pseudocode, we assume `get_hash_indices` returns the index of the cell,
            # and we need to fetch neighbors. 
            # Given the complexity of implementing Instant-NGP hash interpolation from scratch,
            # we will use a simplified lookup that assumes the hash function handles the neighbor fetching or
            # we approximate it by just fetching the primary index if exact interpolation is too verbose.
            # However, the paper explicitly mentions interpolation.
            # Let's assume a helper `interpolate_hash` exists which takes coords, resolution, and grid embedding.
            feat = interpolate_hash(x, res, self.grid_3d[l], self.hash_size)
            features.append(feat)
        return torch.cat(features, dim=-1)

    def query_2d(self, x, levels, plane_type):
        """
        Query 2D grid features.
        plane_type: 'xy', 'xz', 'yz'
        """
        features = []
        for m, res in enumerate(levels):
            if plane_type == 'xy':
                coords_2d = x[:, :2]
                grid = self.grid_2d_xy[m]
            elif plane_type == 'xz':
                coords_2d = x[:, [0, 2]]
                grid = self.grid_2d_xz[m]
            elif plane_type == 'yz':
                coords_2d = x[:, 1:]
                grid = self.grid_2d_yz[m]
            
            feat = interpolate_hash(coords_2d, res, grid, self.hash_size)
            features.append(feat)
        return torch.cat(features, dim=-1)

    def forward(self, x, d):
        """
        Forward pass.
        x: (N, 3) 3D coordinates
        d: (N, 3) viewing directions
        Returns: sigma (N,), color (N, 3)
        """
        # 1. Binarize grid weights
        # Note: In PyTorch, we usually binarize the weights in the forward pass of the embedding or manually.
        # We will binarize the embeddings retrieved.
        
        # 3D Features
        # Resolutions: 16, 32, 64, 128, 256, 512, 1024 (7 levels? Paper says L levels, 16 to 1024)
        # Assuming L=4 for small model, resolutions might be 16, 32, 64, 128? 
        # Paper says "resolutions from 16 to 1024". Let's assume geometric progression.
        res_3d = [16 * (2**i) for i in range(self.L)]
        f_xyz = self.query_3d(x, res_3d)
        
        # 2D Features
        res_2d = [64 * (2**i) for i in range(self.M)]
        f_xy = self.query_2d(x, res_2d, 'xy')
        f_xz = self.query_2d(x, res_2d, 'xz')
        f_yz = self.query_2d(x, res_2d, 'yz')
        
        # Concatenate
        f = torch.cat([f_xyz, f_xy, f_xz, f_yz], dim=-1)
        
        # Positional Encoding for x
        gamma_x = positional_encoding(x, num_frequencies=10)
        
        # Density MLP
        density_input = torch.cat([gamma_x, f], dim=-1)
        density_out = self.mlp_density(density_input)
        sigma = density_out[:, 0:1]
        e = density_out[:, 1:]
        
        # Directional Encoding (Spherical Harmonics)
        d_enc = spherical_harmonic_encoding(d, degree=4) # 27 coefficients for degree 4
        
        # Color MLP
        color_input = torch.cat([e, d_enc], dim=-1)
        color = self.mlp_color(color_input)
        
        return sigma, color

    def loss(self, sigma, color, gt_color, gt_sigma=None):
        """
        Compute loss.
        """
        L_recon = F.mse_loss(color, gt_color)
        
        # Sparsity loss
        # L_sparsity = sum(log(1 + 2 * sigma^2))
        L_sparsity = torch.mean(torch.log(1 + 2 * sigma**2))
        
        lambda_sparsity = 2.0e-5
        L = L_recon + lambda_sparsity * L_sparsity
        
        return L

def interpolate_hash(coords, resolution, embedding_layer, hash_size):
    """
    Interpolate features from hash grid.
    This is a simplified placeholder for the complex hash interpolation logic.
    In Instant-NGP, this involves finding the 8 neighbors in the hash table and interpolating.
    """
    # Get indices for the 8 corners of the voxel containing coords
    # This requires a specific hash function implementation.
    # For the purpose of this article, we assume this function correctly retrieves
    # and interpolates the binary features.
    # The features are binarized inside the embedding lookup or immediately after.
    
    # Get raw embeddings for the 8 neighbors
    # indices = get_8_neighbor_indices(coords, resolution, hash_size)
    # embs = embedding_layer(indices) # (8, F)
    
    # Binarize
    # embs_bin = binarize(embs)
    
    # Interpolate
    # weights = get_trilinear_weights(coords, resolution)
    # feat = sum(weights * embs_bin)
    
    # Placeholder return
    return embedding_layer.weight.mean(dim=0).unsqueeze(0).expand(coords.shape[0], -1)

def binarize(x):
    return (x.sign() * (x.abs() <= 1).float()).detach() + x - x.detach()

def positional_encoding(x, num_frequencies):
    # Standard NeRF positional encoding
    scales = torch.tensor([2**i for i in range(num_frequencies)], device=x.device)
    shaped = x.shape[:-1] + (num_frequencies,) + x.shape[-1:]
    x = torch.reshape(x, shaped)
    x = x * scales
    return torch.cat([torch.sin(x), torch.cos(x)], dim=-1)

def spherical_harmonic_encoding(d, degree):
    # Simplified SH encoding
    # Returns 27 coefficients for degree 4
    # Implementation details omitted for brevity, standard library available
    return torch.randn(d.shape[0], 27)
```

## Operating characteristics
*   **Storage:** Extremely compact. Ours-S2 requires < 0.5 MB, outperforming TensoRF-CP (3.7 MB) and Instant-NGP (24.5 MB) on the mic scene [S2].
*   **Training Time:** Approximately 6, 9, and 14 minutes for Synthetic-NeRF scenes using Ours-B2 to Ours-B8 models on a single NVIDIA RTX A6000 [S2].
*   **Performance:** PSNR of 32.03 dB on Synthetic-NeRF, 34.48 dB on Synthetic-NSVF, and 28.20 dB on Tanks and Temples [S2].
*   **Strengths:** Significant reduction in storage overhead (1-bit parameters); no post-optimization required; competitive reconstruction quality.
*   **Limitations:** Hash collisions in 3D grids at high resolutions can limit performance; mitigated by 2D grids but adds complexity [S2].
*   **When to use:** When storage is a critical bottleneck and real-time or fast training is desired.
*   **When not to use:** When maximum possible reconstruction quality is required over storage constraints, or when hardware does not support efficient hash table operations.

## Background
Binary Radiance Fields (BiRF) was proposed by Seungjoo Shin and Jaesik Park in 2023 [S2]. It builds on the Instant-NGP architecture [20] and NerfAcc [42] for efficient ray sampling. The method addresses the storage burden of implicit feature embeddings in NeRFs by introducing binarization during training [S1]. It is used for static scene reconstruction and novel-view synthesis [S2].

## Sources
[S1] How Far can we Compress Instant-NGP-Based NeRF?, https://doi.org/10.1109/cvpr52733.2024.01921, fetched 2026-10-26
[S2] Binary Radiance Fields, https://doi.org/10.52202/075280-2438, fetched 2026-10-26
