# Volume Radiance Network

Also called: VolumeRadiance

## Overview
The Volume Radiance Network is a neural rendering module that predicts RGB color from input features, view directions, and positions using a Multi-Layer Perceptron (MLP) with positional encoding applied to features and view direction, but NOT position. It builds on [MLPRender_Fea](2414-mlprender-fea.md). Unlike [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which injects global image context, or [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md) which uses linear projections, this method explicitly encodes view directions to model view-dependent appearance.

## How it works
**Inputs and Outputs:**
- Input: Feature tensor $\mathbf{F} \in \mathbb{R}^{N \times D_{feat}}$, view directions $\mathbf{d} \in \mathbb{R}^{N \times 3}$, and optional additional arguments.
- Output: Color tensor $\mathbf{C} \in \mathbb{R}^{N \times 3}$.

**Data Structures:**
The class `VolumeRadiance` maintains the following attributes:
- `n_dir_dims`: Integer, dimensionality of direction encoding (default 3) [S1, unofficial].
- `n_output_dims`: Integer, output dimensionality (default 3) [S1, unofficial].
- `encoding`: Direction encoder object, instantiated via `get_encoding` using `dir_encoding_config` [S1, unofficial].
- `network`: MLP object, instantiated via `get_mlp` using `mlp_network_config` [S1, unofficial].
- `n_input_dims`: Integer, calculated as `input_feature_dim` + output dimension of `encoding` [S1, unofficial].

**Parameters:**
- `input_feature_dim`: Dimension of the input feature vector [S1, unofficial].
- `dir_encoding_config`: Configuration dictionary for the direction encoder [S1, unofficial].
- `mlp_network_config`: Configuration dictionary for the MLP [S1, unofficial].
- `color_activation`: Optional activation function applied to the final color output [S1, unofficial].

**Algorithm:**
1. **Normalize Directions:** If input directions are in $(-1, 1)$, normalize them to $(0, 1)$ using:
   $$ \mathbf{d}_{norm} = \frac{\mathbf{d} + 1.}{2.} $$
   [S1, unofficial].
2. **Encode Directions:** Pass $\mathbf{d}_{norm}$ through the direction encoder. The encoder outputs a tensor of shape $(-1, n_{dir\_dims})$ [S1, unofficial].
3. **Construct Input:** Flatten the input features and concatenate them with the encoded directions and any additional arguments along the last dimension to form `network_inp` [S1, unofficial].
4. **MLP Forward:** Pass `network_inp` through `self.network` [S1, unofficial].
5. **Reshape:** Reshape the MLP output to match the input feature shape plus `n_output_dims` (i.e., $N \times 3$) [S1, unofficial].
6. **Activate Color:** If `color_activation` is defined in the configuration, apply it to the color output [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class VolumeRadiance(nn.Module):
    def __init__(self, input_feature_dim, dir_encoding_config, mlp_network_config, color_activation=None):
        super().__init__()
        self.n_dir_dims = 3  # Default from source [S1, unofficial]
        self.n_output_dims = 3  # Default from source [S1, unofficial]
        
        # Instantiate direction encoder
        # get_encoding is a helper defined in the broader context [S1, unofficial]
        self.encoding = get_encoding(dir_encoding_config)
        
        # Calculate input dimensions for MLP
        # The encoding output size depends on dir_encoding_config [S1, unofficial]
        encoded_dir_dim = self.encoding.output_dim 
        self.n_input_dims = input_feature_dim + encoded_dir_dim
        
        # Instantiate MLP
        # get_mlp is a helper defined in the broader context [S1, unofficial]
        self.network = get_mlp(mlp_network_config, input_dim=self.n_input_dims, output_dim=self.n_output_dims)
        
        self.color_activation = color_activation

    def forward(self, features, directions, **kwargs):
        # 1. Normalize directions from (-1, 1) to (0, 1) [S1, unofficial]
        # Assuming directions are in (-1, 1). If not, this step might be skipped or adapted.
        directions = (directions + 1.0) / 2.0
        
        # 2. Encode directions [S1, unofficial]
        # self.encoding expects directions of shape (..., 3)
        encoded_dirs = self.encoding(directions)
        
        # 3. Concatenate features and encoded directions [S1, unofficial]
        # features shape: (N, input_feature_dim)
        # encoded_dirs shape: (N, encoded_dir_dim)
        network_inp = torch.cat([features, encoded_dirs], dim=-1)
        
        # 4. Pass through MLP [S1, unofficial]
        # network_inp shape: (N, n_input_dims)
        # output shape: (N, n_output_dims)
        out = self.network(network_inp)
        
        # 5. Reshape to match input feature shape plus n_output_dims [S1, unofficial]
        # If features was (N, H, W, D_feat), out should be (N, H, W, 3)
        # Assuming flat input for simplicity as per "flatten features" note
        out = out.view(features.shape[:-1] + (self.n_output_dims,))
        
        # 6. Apply color activation if specified [S1, unofficial]
        if self.color_activation is not None:
            out = self.color_activation(out)
            
        return out
```

**Edge Cases and Pitfalls:**
- Direction normalization assumes input directions are in $(-1, 1)$. If inputs are unit vectors, this normalization is incorrect and should be omitted or replaced with standard normalization [S1, unofficial].
- The position $\mathbf{x}$ is explicitly NOT encoded by the direction encoder or the main network input in this specific variant, relying on the input features to carry spatial information [S1, unofficial].

## Subtopics

### High-Dimensional NeRF
High-Dimensional NeRF modifies the standard radiance network to output high-dimensional features $\mathbf{f}_k \in \mathbb{R}^{dim}$ instead of RGB color, enabling representation of semantics and materials [S2]. The MLP predicts $[\mathbf{f}_k; \sigma_k] = \text{MLP}(\gamma_x(\mathbf{x}_k), \gamma_d(\mathbf{d}_u); \Theta)$ [S2]. Ray feature aggregation is performed via volume rendering: $\mathbf{f}_{r_u} = \sum_k \tau_k (1 - \exp(-\sigma_k \delta_k)) \mathbf{f}_k$ [S2]. This allows supervision in both feature and color space [S2].

## Operating characteristics
**Time and Memory Complexity:**
Linear with respect to the number of samples $N$ and the MLP depth/width. Memory usage is dominated by the MLP parameters and the direction encoding lookup tables (if using positional encoding) [S1, unofficial].

**Strengths and Limitations:**
Explicitly models view-dependent effects through direction encoding. It is computationally efficient compared to full 5D field evaluations if position is handled by features [S1, unofficial]. Limitations include the assumption that position is adequately represented by input features, which may not hold for raw point clouds without prior processing.

**When to Use:**
Use when view-dependent appearance is critical and spatial information is provided via features (e.g., from a voxel grid or point cloud encoder) rather than raw coordinates [S1, unofficial].

## Background
The Volume Radiance Network is implemented in the `instant-nsr-pl` repository [S1, unofficial]. It is a component of neural rendering pipelines that decompose the radiance field prediction into feature processing and view-dependent color prediction.

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
