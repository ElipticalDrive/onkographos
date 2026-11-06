# Instant-NGP

Also called: Instant Neural Graphics Primitives, Instant-NGP Implementation, VFR Implementation Details, hash encoding

## Overview
Instant-NGP is a neural radiance field (NeRF) technique that replaces conventional Fourier (positional) encoding with a multi-resolution hash encoding mechanism to enable rapid training and rendering on consumer hardware [S2]. It facilitates the capture of both overall shape and local textures, outperforming frequency-based methods in preserving sharpness, clarity, and fine details [S2]. Unlike PixelNeRF, which relies on CNN features, or vanilla NeRF, which uses positional encoding, Instant-NGP utilizes a learnable hash table to map spatial coordinates to feature vectors, allowing for efficient parallel processing via CUDA [S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x}$ (for density) and viewing direction $\mathbf{d}$ (for color).
*   **Output:** Volume density $\sigma$ and view-dependent color $\mathbf{c}$.

**Data Structures:**
1.  **Multi-Resolution Hash Grid (MHG):**
    *   Consists of $L$ levels (layers).
    *   Each level $l$ has a hash table mapping voxel positions to feature vectors.
    *   The number of feature channels per level is typically 2 [S1].
    *   If the number of voxels exceeds $2^{19}$, positions are hashed to a one-dimensional table [S1].
    *   The total number of hashing levels is typically 10 [S1].
    *   The final number of samplings on the radiance field is 32 [S1].
2.  **MLP Networks:**
    *   **Spatial MLP:** 2 layers, 256 hidden neurons [S1].
    *   **Directional MLP:** 4 layers, 256 hidden neurons [S1].
    *   **Bottleneck Feature Size:** 256 [S1].
    *   **Skip Connections:** The directional MLP has skip connections to the third layer [S2].
3.  **Density Mapping Layer:**
    *   For real-world 360 datasets: A linear transform from the feature vector to density [S1].
    *   For NeRF synthetic datasets: A tiny network with one hidden layer and 64 neurons [S1].
4.  **Sampling Structures:**
    *   **Occupancy Grid:** Used for efficient sampling on NeRF synthetic datasets [S1].
    *   **Hierarchical Importance Sampling:** Used for real-world 360 datasets, following Zip-NeRF, with two levels of importance sampling, each with 64 samples [S1].

**Parameters:**
*   **Hashmap Size:** 20 [S2].
*   **Activation Function:** GELU (chosen over ReLU for slightly better quality) [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2] or 0.01 [S1].
*   **Batch Size:** 1024 rays [S2].
*   **Training Epochs:** 30K [S2].

**Algorithm Steps:**
1.  **Query Features:** For a sample point $\mathbf{x}_i$, query the Multi-Resolution Hash Grid to obtain a feature vector $F(\mathbf{x}_i)$. This involves hashing the coordinates at each resolution level and trilinearly interpolating the features [S1, S2].
2.  **Compute Density:** Pass $F(\mathbf{x}_i)$ through the density mapping layer to derive the density $\sigma_i$ [S1].
3.  **Compute Color:**
    *   Concatenate the bottleneck feature (output of Spatial MLP) with the view direction $\mathbf{d}$.
    *   Pass the concatenated vector through the Directional MLP to obtain color $\mathbf{c}_i$ [S1].
4.  **Render:** Use volume rendering integration (e.g., standard NeRF rendering equation) to combine $\sigma_i$ and $\mathbf{c}_i$ along the ray [S1].
5.  **Train:** Optimize the hash table entries and MLP weights using the Adam optimizer [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class InstantNGP(nn.Module):
    def __init__(self, num_levels=10, num_features_per_level=2, hash_size=2**19, 
                 hidden_dim=256, num_direction_layers=4, num_spatial_layers=2):
        super().__init__()
        # Multi-resolution hash grid
        # In practice, this is a CUDA kernel or specialized tensor lookup
        # Here we simulate the feature query
        self.num_levels = num_levels
        self.num_features_per_level = num_features_per_level
        
        # Spatial MLP: 2 layers, 256 hidden neurons
        self.spatial_mlp = nn.Sequential(
            nn.Linear(num_features_per_level * num_levels, hidden_dim),
            nn.GELU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.GELU()
        )
        
        # Directional MLP: 4 layers, 256 hidden neurons, skip to 3rd layer
        self.directional_mlp = nn.Sequential(
            nn.Linear(hidden_dim + 3, hidden_dim), # +3 for direction
            nn.GELU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.GELU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.GELU(),
            nn.Linear(hidden_dim, 3) # Output color
        )
        
        # Density mapping layer
        # For synthetic: 1 hidden layer, 64 neurons
        # For real-world: Linear
        self.density_mapping = nn.Sequential(
            nn.Linear(hidden_dim, 64),
            nn.GELU(),
            nn.Linear(64, 1)
        )
        
        # Initialize hash table entries (learnable parameters)
        # Size: (num_levels, hash_size, num_features_per_level)
        self.hash_table = nn.Parameter(torch.randn(num_levels, hash_size, num_features_per_level))

    def query_hash_grid(self, x):
        """
        Query the multi-resolution hash grid for coordinates x.
        x: (N, 3) tensor of coordinates.
        Returns: (N, num_levels * num_features_per_level) feature vector.
        """
        # Note: Actual implementation uses CUDA for hashing and trilinear interpolation.
        # This is a simplified placeholder for the logic.
        features = []
        for l in range(self.num_levels):
            # Hash coordinates to table indices
            # In real implementation, this involves scaling x to grid resolution l
            # and using a hash function to find the index in self.hash_table[l]
            indices = self._hash_coordinates(x, l)
            # Gather features
            level_features = self.hash_table[l][indices]
            features.append(level_features)
        return torch.cat(features, dim=-1)

    def _hash_coordinates(self, x, level):
        # Placeholder for actual hashing logic
        # Returns indices into the hash table for the given level
        return torch.randint(0, self.hash_table.shape[1], (x.shape[0], 3))

    def forward(self, x, d):
        """
        x: (N, 3) spatial coordinates
        d: (N, 3) viewing directions
        """
        # 1. Query features
        features = self.query_hash_grid(x)
        
        # 2. Spatial MLP
        spatial_out = self.spatial_mlp(features)
        
        # 3. Density
        density = self.density_mapping(spatial_out).squeeze(-1)
        
        # 4. Color
        # Concatenate spatial features and direction
        # Skip connection to 3rd layer of directional MLP
        # Note: The skip connection is typically implemented by adding the spatial output 
        # to the input of the 3rd layer or concatenating. 
        # Based on [S2], skip connections to the third layer.
        # Simplified: Concatenate at input
        dir_input = torch.cat([spatial_out, d], dim=-1)
        color = self.directional_mlp(dir_input)
        
        return density, color
```

**Edge Cases and Pitfalls:**
*   **Soft Tissues:** NeRF techniques are sensitive to tissue type. Soft tissues have lower contrast and homogeneous textures, leading to reconstructions that are artifact-free but sometimes appear less detailed or slightly blurred compared to hard tissues [S2].
*   **Precision:** The hash table size and number of levels must be balanced to avoid collisions and ensure sufficient granularity [S2].

## Subtopics

### Zip-NeRF
A hierarchical importance sampling method used in Instant-NGP for real-world datasets, using two levels of importance sampling with 64 samples each [S1].

### Occupancy Grid
A spatial data structure used for efficient sampling in NeRF synthetic datasets, reducing the number of samples needed [S1].

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than vanilla NeRF due to the parallel nature of hash table lookups and the small MLP size [S2].
*   **Memory Complexity:** The hash table size is configurable. A hashmap size of 20 is used in medical imaging applications [S2].
*   **Measured Performance:**
    *   **Chest Dataset:** 3D SSIM of 0.943 (vs. 0.885 for vanilla NeRF), LPIPS of 0.090 (vs. 0.164), GMSD of $3.9 \times 10^{-6}$ (vs. $2.7 \times 10^{-5}$) [S2]. Rendering time: 2.591 s (vs. 5.493 s for PixelNeRF) [S2].
    *   **Feet Dataset:** 3D SSIM margin of 0.185 over FDK. Rendering time: 1.016 s [S2].
    *   **Abdominal Dataset:** Surpassed FDK by 8.3 in PSNR [S2].
    *   **Jaw Dataset:** GMSD of $3.7 \times 10^{-6}$ (vs. $3.3 \times 10^{-5}$ for FDK) [S2].
*   **Strengths:** Fast training and rendering, high fidelity for hard tissues and fine details, efficient memory usage compared to dense voxel grids [S2][S3, unofficial].
*   **Limitations:** Performance degrades on soft tissues with low contrast and homogeneous textures [S2].

## Background
Instant-NGP was introduced by Müller et al. in 2022 as part of the SIGGRAPH conference [S3, unofficial]. It was designed to address the slow training times of traditional NeRFs by introducing a multi-resolution hash encoding scheme. The method has been adapted for various applications, including medical image reconstruction, where it has shown superior performance compared to traditional methods like FDK and other NeRF variants [S2].

## Sources
[S1] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://doi.org/10.52202/075280-2853, fetched 2026-10-26
[S2] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
