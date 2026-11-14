# HashNeRF Parameters

Also called: HashNeRF Hyperparameters

## Overview
HashNeRF parameters define the configuration for the multi-resolution hash grid encoding and the associated training loop used in the original HashNeRF implementation [S1, unofficial]. This method replaces positional encoding with a learnable sparse hash table to map 3D coordinates to feature vectors, enabling fast training and inference [S1, unofficial]. It differs from sibling methods like Depth-Guided NeRF and CLIPNeRF by relying on explicit geometric hashing rather than depth priors or semantic constraints, prioritizing raw reconstruction speed and memory efficiency over semantic understanding or depth accuracy [S1, unofficial][11916-depth-guided-nerf-training-configuration.md][12073-clipnerf-training-configuration.md].

## How it works
The method configures a neural network that predicts density ($\sigma$) and color ($c$) using a hash-based input encoding. The core component is the `HashGrid` encoder, which maps 3D positions to a concatenated feature vector from multiple resolution levels.

**Inputs and Outputs:**
*   **Input:** 3D point position $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\mathbf{d} \in \mathbb{R}^3$.
*   **Output:** Feature vector for $\mathbf{x}$ (concatenated across levels) and final pixel color $c$.

**Data Structures:**
1.  **Hash Grid Pyramid:** A list of $L$ hash tables.
    *   Each table $i$ has a resolution $R_i = \lfloor B \cdot b^i \rfloor$, where $B$ is the base resolution and $b$ is the growth factor.
    *   Each table maps hashed voxel indices to feature vectors of size $F$.
    *   Hashmap size is defined by `log2_hashmap_size` ($H$), where total entries = $2^H$.
2.  **Embedding Parameters:** Stored in `embedding_params` when `args.i_embed==1` [S1, unofficial].

**Parameters:**
*   `finest_res`: Finest resolution for hashed embedding. Default: 512 [S1, unofficial].
*   `log2_hashmap_size`: Log2 of hashmap size per level. Default: 19 [S1, unofficial].
*   `raw_noise_std`: Standard deviation of noise added to regularize $\sigma_a$ output. Recommended: $1.0$ [S1, unofficial].
*   `sparse-loss-weight`: Learning rate/weight for sparse loss. Default: $10^{-10}$ [S1, unofficial].
*   `tv-loss-weight`: Learning rate/weight for Total Variation loss. Default: $10^{-6}$ [S1, unofficial].
*   `render_factor`: Downsampling factor for rendering. Set to 4 or 8 for fast preview [S1, unofficial].
*   `precrop_iters`: Number of steps to train on central crops. Default: 0 [S1, unofficial].
*   `precrop_frac`: Fraction of image taken for central crops. Default: 0.5 [S1, unofficial].
*   `dataset_type`: Options: `llff`, `blender`, `deepvoxels`. Default: `llff` [S1, unofficial].
*   `shape`: Options: `armchair`, `cube`, `greek`, `vase`. Default: `greek` [S1, unofficial].
*   `white_bkgd`: Boolean to render synthetic data on a white background [S1, unofficial].
*   `half_res`: Boolean to load Blender synthetic data at 400x400 instead of 800x800 [S1, unofficial].
*   `scannet_sceneID`: Scene ID to load from ScanNet. Default: `scene0000_00` [S1, unofficial].
*   `factor`: Downsample factor for LLFF images. Default: 8 [S1, unofficial].
*   `no_ndc`: Boolean to disable Normalized Device Coordinates (set for non-forward facing scenes) [S1, unofficial].
*   `lindisp`: Boolean to sample linearly in disparity rather than depth [S1, unofficial].
*   `spherify`: Boolean to set for spherical 360 scenes [S1, unofficial].
*   `llffhold`: Take every $1/N$ images as LLFF test set. Default: 8 [S1, unofficial].
*   `i_print`: Frequency of console printout and metric logging. Default: 100 [S1, unofficial].
*   `i_img`: Frequency of TensorBoard image logging. Default: 500 [S1, unofficial].
*   `i_weights`: Frequency of weight checkpoint saving. Default: 10000 [S1, unofficial].
*   `i_testset`: Frequency of testset saving. Default: 1000 [S1, unofficial].
*   `i_video`: Frequency of render_poses video saving. Default: 5000 [S1, unofficial].
*   `testskip`: Load $1/N$ images from test/val sets. Default: 8 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize Grids:** Create $L$ hash tables with resolutions $R_i$. For the original HashNeRF, $L$ is determined by `finest_res` and growth factor. For RING-NeRF (which uses HashNeRF parameters), $L=10$, growth factor $b=2$, resolutions 16 to 8192, feature size $F=8$, and $H=21$ ($128^3$) [S2].
2.  **Encode Position:** For a query point $\mathbf{x}$:
    a. For each level $i$:
       i. Calculate resolution $R_i = \lfloor B \cdot b^i \rfloor$.
       ii. Determine voxel bounds and hash indices for $\mathbf{x}$ at resolution $R_i$.
       iii. Lookup features from the hash table using hashed indices.
       iv. Perform trilinear interpolation of features based on $\mathbf{x}$'s position within the voxel.
    b. Concatenate interpolated features from all levels.
3.  **Network Forward Pass:**
    a. Concatenate encoded $\mathbf{x}$ features and encoded $\mathbf{d}$ (if using spherical harmonics or similar).
    b. Pass through MLP to predict density $\sigma$ and color $c$.
    c. Add noise to $\sigma$ output: $\sigma' = \sigma + \epsilon$, where $\epsilon \sim \mathcal{N}(0, \text{raw\_noise\_std})$ [S1, unofficial].
4.  **Volume Rendering:** Integrate $\sigma'$ and $c$ along rays using standard differentiable volume rendering equation to produce pixel color.
5.  **Loss Calculation:** Compute loss including photometric error, sparse loss (weighted by `sparse-loss-weight`), and TV loss (weighted by `tv-loss-weight`) [S1, unofficial].
6.  **Optimization:** Update parameters using RAdam optimizer.
    *   Embedding parameters: `eps=1e-15` [S1, unofficial].
    *   General parameters: Initial LR $10^{-2}$, cosine decay to $10^{-3}$ [S2].
7.  **Training Loop:** Run for 25k iterations [S2]. Batch size: $2^{16}$ rays [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class HashGridEncoder(nn.Module):
    def __init__(self, finest_res=512, log2_hashmap_size=19, base_res=16, growth_factor=2, num_levels=10, feature_dim=8):
        super().__init__()
        self.finest_res = finest_res
        self.log2_hashmap_size = log2_hashmap_size
        self.base_res = base_res
        self.growth_factor = growth_factor
        self.num_levels = num_levels
        self.feature_dim = feature_dim
        
        # Initialize hash tables
        self.embeddings = nn.ModuleList([
            nn.Embedding(2**log2_hashmap_size, feature_dim) 
            for _ in range(num_levels)
        ])
        
        # Initialize weights to small values
        for emb in self.embeddings:
            nn.init.uniform_(emb.weight, a=-1e-4, b=1e-4)

    def get_voxel_vertices(self, x, resolution, bounding_box):
        # Normalize x to [0, 1] based on bounding_box
        x_norm = (x - bounding_box.min()) / (bounding_box.max() - bounding_box.min())
        # Calculate voxel coordinates
        voxel_coords = x_norm * resolution
        voxel_min = torch.floor(voxel_coords).long()
        voxel_max = voxel_min + 1
        # Clamp to resolution
        voxel_min = torch.clamp(voxel_min, 0, resolution - 1)
        voxel_max = torch.clamp(voxel_max, 0, resolution - 1)
        return voxel_min, voxel_max, x_norm

    def hash(self, voxel_coords, resolution):
        # Simple hash function for demonstration
        # In practice, use a specific hash function like the one in HashNeRF
        x, y, z = voxel_coords.unbind(-1)
        p = 2654435761
        h = (x * p) ^ (y * p) ^ (z * p)
        return h % (2**self.log2_hashmap_size)

    def trilinear_interp(self, x_norm, voxel_min, voxel_max, features):
        # Calculate interpolation weights
        d = x_norm - voxel_min.float()
        w0 = (1 - d)
        w1 = d
        
        # Get features for 8 corners
        # This is a simplified version; actual implementation requires handling 8 corners
        # and their respective hashes
        pass

    def forward(self, x):
        x_embedded_all = []
        for i in range(self.num_levels):
            resolution = int(torch.floor(self.base_res * self.growth_factor**i))
            voxel_min, voxel_max, x_norm = self.get_voxel_vertices(x, resolution, self.bounding_box)
            
            # Hash indices
            hashed_indices = self.hash(voxel_min, resolution)
            
            # Lookup features
            voxel_embedds = self.embeddings[i](hashed_indices)
            
            # Interpolate
            x_embedded = self.trilinear_interp(x_norm, voxel_min, voxel_max, voxel_embedds)
            x_embedded_all.append(x_embedded)
            
        return torch.cat(x_embedded_all, dim=-1)

class HashNeRFModel(nn.Module):
    def __init__(self, raw_noise_std=1.0):
        super().__init__()
        self.encoder = HashGridEncoder()
        self.raw_noise_std = raw_noise_std
        # MLP for density and color
        self.mlp = nn.Sequential(
            nn.Linear(64, 64), # Assuming 8 levels * 8 features
            nn.ReLU(),
            nn.Linear(64, 4), # density + 3 color
        )

    def forward(self, x, d):
        # Encode position
        x_feat = self.encoder(x)
        # Encode direction (simplified, assuming spherical harmonics or similar)
        d_feat = ... # Direction encoding
        # Concatenate
        input_feat = torch.cat([x_feat, d_feat], dim=-1)
        # MLP
        output = self.mlp(input_feat)
        sigma = output[..., 0]
        color = output[..., 1:]
        # Add noise to sigma
        sigma = sigma + torch.randn_like(sigma) * self.raw_noise_std
        return sigma, color
```

## Subtopics

### Hashed Embedding Table Initialization
When `args.i_embed==1`, the model uses a hashed embedding table stored in `embedding_params` [S1, unofficial]. The optimizer uses RAdam with a specific epsilon of $10^{-15}$ for these embedding parameters [S1, unofficial]. If `args.i_embed==1`, the model uses `NeRFSmall` with fixed dimensions (hidden_dim=64, geo_feat_dim=15); otherwise, it uses the standard `NeRF` model with configurable depth/width [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is fast due to the efficient hash lookup and small MLP. RING-NeRF (using HashNeRF parameters) trains in 25k iterations [S2].
*   **Memory Complexity:** Memory usage is determined by the hash table size ($2^H$ entries per level) and feature dimension. With $H=19$ and $F=8$, memory is manageable on modern GPUs [S1, unofficial][S2].
*   **Performance:** Fast preview with `render_factor` 4 or 8 [S1, unofficial]. High fidelity with sufficient iterations and resolution [S2].
*   **Strengths:** Fast training, low memory footprint, easy to implement.
*   **Limitations:** Hash collisions can occur, potentially affecting quality. Less robust to out-of-distribution inputs compared to continuous encodings.
*   **When to use:** When training speed and memory efficiency are priorities, and the scene is bounded.
*   **When not to use:** For unbounded scenes or when high-frequency details are critical and collisions are a concern.

## Background
HashNeRF was introduced by Müller et al. in "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding" (2022) [S1, unofficial]. It popularized the use of multi-resolution hash grids for neural radiance fields, enabling real-time rendering and fast training. The parameters described here are from the unofficial PyTorch implementation by Yash Bhalgat [S1, unofficial]. RING-NeRF also utilizes similar hash grid configurations [S2].

## Sources
[S1] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S2] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
