# Binary Radiance Fields

Also called: BRF, BiRF, Binarized Radiance Fields

## Overview
Binary Radiance Fields (BiRF) is a storage-efficient radiance field representation that constrains feature grid parameters to binary values $\{-1, +1\}$ to drastically reduce memory footprint while maintaining reconstruction quality [S2]. Unlike post-processing compression methods such as PlenOctrees or Re:NeRF, which require additional optimization steps and are bounded by pre-trained model performance, BiRF performs binarization during training using a straight-through estimator, allowing for end-to-end optimization without post-optimization [S2]. It distinguishes itself from other grid-based methods by combining binary feature encoding with a 2D-3D hybrid multi-resolution architecture, where 3D voxels capture main components and 2D planes capture fine details, effectively mitigating hash collisions common in high-resolution 3D grids [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed images with known camera parameters; a 3D coordinate $x = (x, y, z)$ and viewing direction $\mathbf{d} = (\theta, \phi)$ for ray marching.
*   **Output:** Volume density $\sigma$ and color $\mathbf{c} = (r, g, b)$ for the query point.

**Data Structures:**
The model uses a hybrid multi-resolution feature grid consisting of one 3D voxel grid and three orthogonal 2D plane grids, all implemented using hash encoding [S2].
1.  **3D Feature Grid ($\Phi_{\theta_{xyz}}$):** Stores binary parameters. It has $L$ resolution levels, ranging from $16^3$ to $1024^3$. Each level contains up to $T_{3D}$ feature vectors.
2.  **2D Feature Grids ($\Phi_{\theta_{xy}}, \Phi_{\theta_{xz}}, \Phi_{\theta_{yz}}$):** Store binary parameters for the XY, XZ, and YZ planes respectively. Each has $M$ resolution levels, ranging from $64^2$ to $512^2$. Each level contains up to $T_{2D}$ feature vectors.
3.  **Parameter Constraint:** All grid parameters $\theta$ are constrained to $\{-1, +1\}$ via binarization [S2].

**Binarization and Gradient Flow:**
Real-valued parameters $\theta$ are transformed to binary parameters $\theta'$ using the sign function:
$$
\theta' = \text{sign}(\theta) = \begin{cases} +1 & \text{if } \theta \ge 0 \\ -1 & \text{otherwise} \end{cases}
$$
Since the derivative of the sign function is zero almost everywhere, gradients are backpropagated using the Straight-Through Estimator (STE) [S2]:
$$
\frac{\partial L}{\partial \theta} = \frac{\partial L}{\partial \theta'} \cdot \mathbb{I}(|\theta| \le 1)
$$
This strategy ensures gradient flow while constraining the scale of $\theta$ to $\{-1, 1\}$, preventing divergence [S2].

**Feature Encoding:**
For a query 3D coordinate $x$, features are extracted from the hybrid grids:
1.  **3D Encoding:** Tri-linearly interpolate binary parameters across $L$ levels and concatenate:
    $$
    f_{xyz} = \{ \text{interp}(x, \text{sign}(\Phi^l_{\theta_{xyz}})) \}_{l=1}^L
    $$
2.  **2D Encoding:** Project $x$ onto the three orthogonal planes: $x_{xy}=(x,y)$, $x_{xz}=(x,z)$, $x_{yz}=(y,z)$. Bi-linearly interpolate binary parameters across $M$ levels for each plane and concatenate:
    $$
    f_{xy} = \{ \text{interp}(x_{xy}, \text{sign}(\Phi^m_{\theta_{xy}})) \}_{m=1}^M
    $$
    Repeat for $f_{xz}$ and $f_{yz}$ [S2].
3.  **Concatenation:** Combine all features:
    $$
    f = \{ f_{xyz}, f_{xy}, f_{xz}, f_{yz} \}
    $$

**Network Architecture:**
1.  **Positional Encoding:** Apply sinusoidal positional encoding $\gamma(x)$ to the input coordinate $x$ [S2].
2.  **Density MLP:** Feed $\gamma(x)$ and $f$ into a shallow MLP with one hidden layer (128 channels, ReLU) to predict density $\sigma$ and embedded feature $e$:
    $$
    (\sigma, e) = \text{MLP}_{\text{density}}(\gamma(x), f)
    $$
3.  **Color MLP:** Feed $e$ and directional encoding $\mathbf{d}$ (using spherical harmonics) into a shallow MLP with two hidden layers (128 channels, ReLU) to predict color $\mathbf{c}$:
    $$
    \mathbf{c} = \text{MLP}_{\text{color}}(e, \mathbf{d})
    $$

**Loss Function:**
The total loss combines reconstruction loss and sparsity regularization:
$$
L = L_{\text{recon}} + \lambda_{\text{sparsity}} L_{\text{sparsity}}
$$
*   **Reconstruction Loss:** Mean squared error between rendered color $\hat{C}(r)$ and ground truth $C(r)$ for sampled rays $r$:
    $$
    L_{\text{recon}} = \sum_{r \in \mathcal{R}} \| \hat{C}(r) - C(r) \|^2_2
    $$
*   **Sparsity Loss:** Cauchy loss to encourage sparse density, accelerating rendering by allowing empty space skipping via an occupancy grid:
    $$
    L_{\text{sparsity}} = \sum_{i,k} \log(1 + 2\sigma(r_i(t_k))^2)
    $$
    where $\lambda_{\text{sparsity}} = 2.0 \times 10^{-5}$ [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BinaryRadianceField:
    def __init__(self, T_3D, T_2D, F, L=10, M=4):
        """
        Initialize BiRF model.
        :param T_3D: Max feature vectors per level in 3D grid (e.g., 2^17 for small, 2^19 for base)
        :param T_2D: Max feature vectors per level in 2D grid (e.g., 2^15 for small, 2^17 for base)
        :param F: Feature dimension per level (e.g., 2, 4, 8)
        :param L: Number of 3D resolution levels
        :param M: Number of 2D resolution levels
        """
        super().__init__()
        self.F = F
        self.L = L
        self.M = M
        
        # Initialize real-valued parameters for hash grids
        # Note: In practice, these are managed by a hash table implementation like Instant-NGP
        # Here we abstract the grid storage as a list of tensors for clarity
        self.grid_3d = nn.ParameterList([
            nn.Parameter(torch.randn(T_3D, F)) for _ in range(L)
        ])
        self.grid_2d_xy = nn.ParameterList([
            nn.Parameter(torch.randn(T_2D, F)) for _ in range(M)
        ])
        self.grid_2d_xz = nn.ParameterList([
            nn.Parameter(torch.randn(T_2D, F)) for _ in range(M)
        ])
        self.grid_2d_yz = nn.ParameterList([
            nn.Parameter(torch.randn(T_2D, F)) for _ in range(M)
        ])
        
        # Density MLP: Input dim = L*F + M*3*F (concatenated features) + pos_enc_dim
        # Simplified: Assume pos_enc_dim is handled separately or included in input dim
        input_dim_density = L * F + 3 * M * F + 6 # 6 for basic pos enc (sin/cos of x,y,z)
        self.mlp_density = nn.Sequential(
            nn.Linear(input_dim_density, 128),
            nn.ReLU(),
            nn.Linear(128, 1 + F) # Output: 1 density, F embedded features
        )
        
        # Color MLP: Input dim = F + 6 (directional encoding)
        self.mlp_color = nn.Sequential(
            nn.Linear(F + 6, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, 3) # Output: RGB
        )

    def sign_ste(self, x):
        """Straight-Through Estimator for sign function."""
        return torch.sign(x)

    def get_binary_features(self, grid_params, coords, interp_func):
        """
        Retrieve and interpolate binary features from a grid.
        :param grid_params: List of parameters for each level
        :param coords: Coordinates to query
        :param interp_func: Interpolation function (trilinear or bilinear)
        :return: Concatenated feature vector
        """
        features = []
        for level_params in grid_params:
            # Binarize parameters
            binary_params = self.sign_ste(level_params)
            # Interpolate
            feat = interp_func(binary_params, coords)
            features.append(feat)
        return torch.cat(features, dim=-1)

    def forward(self, x, d):
        """
        Forward pass.
        :param x: 3D coordinates (B, 3)
        :param d: Viewing directions (B, 3)
        :return: density (B, 1), color (B, 3)
        """
        # 1. Positional Encoding for x
        # Assuming standard NeRF positional encoding
        gamma_x = positional_encoding(x, num_frequencies=3) # Returns (B, 6)
        
        # 2. Feature Encoding
        # 3D Grid: Trilinear interpolation
        f_xyz = self.get_binary_features(self.grid_3d, x, trilinear_interp)
        
        # 2D Grids: Bilinear interpolation on projections
        x_xy = x[:, :2]
        x_xz = x[:, [0, 2]]
        x_yz = x[:, [1, 2]]
        
        f_xy = self.get_binary_features(self.grid_2d_xy, x_xy, bilinear_interp)
        f_xz = self.get_binary_features(self.grid_2d_xz, x_xz, bilinear_interp)
        f_yz = self.get_binary_features(self.grid_2d_yz, x_yz, bilinear_interp)
        
        # Concatenate features
        f = torch.cat([f_xyz, f_xy, f_xz, f_yz], dim=-1)
        
        # 3. Density Prediction
        density_input = torch.cat([gamma_x, f], dim=-1)
        density_output = self.mlp_density(density_input)
        sigma = density_output[:, :1]
        e = density_output[:, 1:]
        
        # 4. Color Prediction
        # Directional encoding (e.g., spherical harmonics or simple pos enc)
        gamma_d = directional_encoding(d) # Returns (B, 6)
        color_input = torch.cat([e, gamma_d], dim=-1)
        color = self.mlp_color(color_input)
        
        return sigma, color

    def loss(self, sigma, color, gt_color, gt_sigma=None):
        """
        Compute total loss.
        :param sigma: Predicted density
        :param color: Predicted color
        :param gt_color: Ground truth color
        :param gt_sigma: Ground truth density (optional, for sparsity loss if available)
        :return: Total loss scalar
        """
        # Reconstruction Loss
        l_recon = F.mse_loss(color, gt_color)
        
        # Sparsity Loss (Cauchy loss on predicted density)
        # L_sparsity = sum(log(1 + 2 * sigma^2))
        l_sparsity = torch.log(1 + 2 * sigma**2).mean()
        
        lambda_sparsity = 2.0e-5
        total_loss = l_recon + lambda_sparsity * l_sparsity
        
        return total_loss
```

## Operating characteristics
*   **Storage Efficiency:** BiRF achieves extremely compact storage. The small model variant (Ours-S2) requires less than 0.5 MB, significantly outperforming baselines like TensoRF-CP (3.7 MB) and Instant-NGP (24.5 MB) on the "mic" scene while maintaining comparable or better PSNR [S2].
*   **Performance:** On Synthetic-NeRF scenes, BiRF achieves a PSNR of 32.03 dB; on Synthetic-NSVF, 34.48 dB; and on Tanks and Temples, 28.20 dB [S2].
*   **Training Time:** Training takes approximately 6, 9, and 14 minutes for Synthetic-NeRF scenes using Ours-B2 to Ours-B8 models on a single NVIDIA RTX A6000 GPU [S2].
*   **Strengths:** Dramatic reduction in storage overhead compared to real-valued floating-point grids; no post-optimization required; hybrid 2D-3D structure mitigates hash collisions in 3D grids [S2].
*   **Limitations:** Binary quantization may limit the expressiveness of the feature space compared to full-precision models, though the hybrid architecture compensates for this [S2]. Hash collisions in 3D grids at high resolutions can still impact performance if not supplemented by 2D grids [S2].
*   **When to Use:** Ideal for applications requiring extreme storage efficiency and fast deployment, such as mobile AR/VR or web-based 3D viewing, where model size is a critical constraint [S2].
*   **When Not to Use:** When maximum possible reconstruction quality is required regardless of storage size, full-precision NeRF or Instant-NGP may be preferable [S2].

## Background
Binary Radiance Fields were proposed by Seungjoo Shin and Jaesik Park from POSTECH and Seoul National University, published in arXiv:2306.07581v2 on December 4, 2023 [S2]. The method builds upon the Instant-NGP architecture, utilizing its multi-resolution hash encoding and occupancy grid for efficient ray sampling [S2]. It addresses the storage burden imposed by implicit feature embeddings in modern NeRF variants by introducing binarization of feature embeddings to $\{-1, +1\}$ using a sign function and backpropagating through a straight-through estimator [S1]. The approach was implemented based on Instant-NGP using the occupancy grid from NerfAcc and trained on NVIDIA RTX A6000 GPUs [S2].

## Sources
[S1] How Far can we Compress Instant-NGP-Based NeRF?, https://doi.org/10.1109/cvpr52733.2024.01921, fetched 2026-10-26
[S2] Binary Radiance Fields, https://doi.org/10.52202/075280-2438, fetched 2026-10-26
