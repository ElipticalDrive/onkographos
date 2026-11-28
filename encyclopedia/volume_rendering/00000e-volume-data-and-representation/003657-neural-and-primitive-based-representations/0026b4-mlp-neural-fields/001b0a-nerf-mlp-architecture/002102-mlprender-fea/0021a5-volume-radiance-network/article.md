# Volume Radiance Network

Also called: VolumeRadiance

## Overview
The Volume Radiance Network is a neural rendering module that predicts color from input features and view directions using a Multi-Layer Perceptron (MLP) [S1, unofficial]. It builds on [MLPRender_Fea](2414-mlprender-fea.md) by applying positional encoding to features and view directions, but not to positions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Input features, view directions, and optional additional arguments.
- Output: Predicted color or feature values.

**Data Structures:**
The method is implemented as a class `VolumeRadiance` with attributes: `n_dir_dims` (from config, default 3), `n_output_dims` (3), `encoding` (instantiated from `dir_encoding_config`), `n_input_dims` (sum of `input_feature_dim` and encoding output dimensions), and `network` (instantiated from `mlp_network_config`) [S1, unofficial].

**Parameters:**
- `input_feature_dim`: Dimension of input features.
- `dir_encoding_config`: Configuration for direction encoding.
- `mlp_network_config`: Configuration for the MLP network.
- `color_activation`: Activation function for color output.

**Steps:**
1. Normalize directions from $(-1, 1)$ to $(0, 1)$ using $\mathbf{D}_{norm} = (\mathbf{D} + 1.0) / 2.0$ [S1, unofficial].
2. Encode directions using `self.encoding`, producing output of shape $(-1, n\_dir\_dims)$ [S1, unofficial].
3. Concatenate flattened features, encoded directions, and any additional arguments along the last dimension to form `network_inp` [S1, unofficial].
4. Pass `network_inp` through `self.network` [S1, unofficial].
5. Reshape the output to match the input feature shape plus `n_output_dims` [S1, unofficial].
6. If `color_activation` is in the config, apply the activation function to the color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class PositionalEncoding(nn.Module):
    def __init__(self, out_dim):
        super().__init__()
        self.out_dim = out_dim
    def forward(self, x):
        return x.view(x.size(0), -1)

class MLP(nn.Module):
    def __init__(self, in_dim, out_dim, hidden_dim=256, num_layers=8):
        super().__init__()
        layers = []
        for _ in range(num_layers - 1):
            layers.append(nn.Linear(in_dim, hidden_dim))
            layers.append(nn.ReLU())
            in_dim = hidden_dim
        layers.append(nn.Linear(hidden_dim, out_dim))
        self.net = nn.Sequential(*layers)
    def forward(self, x):
        return self.net(x)

class VolumeRadiance(nn.Module):
    def __init__(self, input_feature_dim, dir_encoding_config, mlp_network_config, color_activation=None):
        super().__init__()
        self.n_dir_dims = dir_encoding_config.get('n_dir_dims', 3)
        self.n_output_dims = 3
        self.encoding = PositionalEncoding(dir_encoding_config.get('out_dim', self.n_dir_dims))
        self.n_input_dims = input_feature_dim + self.encoding.out_dim
        self.network = MLP(mlp_network_config.get('in_dim', self.n_input_dims), self.n_output_dims)
        self.color_activation = color_activation

    def forward(self, features, directions, **kwargs):
        directions_norm = (directions + 1.0) / 2.0
        directions_enc = self.encoding(directions_norm)
        features_flat = features.view(features.size(0), -1)
        network_inp = torch.cat([features_flat, directions_enc], dim=-1)
        output = self.network(network_inp)
        if self.color_activation is not None:
            output = self.color_activation(output)
        return output
```

## Subtopics

### [Disco NSVF Models](11063-disco-nsvf-models.md)
Disco NSVF models represent 3D scenes using a disentangled radiance field, registered as `disco_nsvf` and `multi_disco_nsvf`. The `disco_nsvf` variant uses `compressed_light_dim=64` and `nerf3_architecture`, while `multi_disco_nsvf` employs `multi_sparsevoxel_encoder` with `inputs_to_density="pos:10, context:0:256"` and `inputs_to_texture="feat:0:256, pos:10, normal:4, ray:4, context:0:256"`. They differ from their sibling [SDF NSVF Models](11064-sdf-nsvf-models.md) by utilizing a disentangled radiance field representation rather than a unified SDF-based approach.

### [Neural Irradiance Volume](14514-neural-irradiance-volume.md)
Neural Irradiance Volume (NIV) is a real-time rendering technique that compresses the 5D indirect diffuse irradiance field of a static scene into a compact neural model, enabling the shading of unseen dynamic objects without runtime ray tracing or denoising. Unlike traditional grid-based probe methods which suffer from aliasing and cubic memory scaling, NIV maps position and direction directly to irradiance values, improving quality by at least 10x at the same memory budget. It differs from other neural rendering methods by learning a pre-integrated volumetric quantity (irradiance) rather than raw incident radiance, allowing noise-free shading at ~1 ms per frame on consumer hardware.

### High-Dimensional NeRF
High-Dimensional NeRF is a modification of NeRF that maps 3D points to high-dimensional features $\mathbf{f}_k \in \mathbb{R}^{dim}$ instead of RGB color, allowing representation of semantics and materials [S2]. The MLP prediction is defined as:
$$ [\mathbf{f}_k; \sigma_k] = \text{MLP}(\gamma_x(\mathbf{x}_k), \gamma_d(\mathbf{d}_u); \Theta) $$
where $\mathbf{f}_k$ is the high-dimensional feature of point $\mathbf{x}_k$ [S2]. Ray feature aggregation is performed via volume rendering:
$$ \mathbf{f}_{r_u} = \sum_k \tau_k (1 - \exp(-\sigma_k \delta_k)) \mathbf{f}_k $$
where $\mathbf{f}_{r_u}$ is the aggregated ray feature [S2]. This allows the network to encode richer information such as semantics and materials, supervised in both feature and color space [S2].

### Indirect Illumination MLP
Indirect Illumination MLP is an MLP $I(\mathbf{x})$ that outputs Spherical Gaussian (SG) parameters $\boldsymbol{\Gamma} \in \mathbb{R}^{24 \times 7}$ for any input 3D location $\mathbf{x}$, representing indirect incoming illumination as a mixture of SGs to avoid Monte-Carlo integration [S3]. The indirect illumination MLP is supervised by drawing samples from the outgoing radiance field $\mathcal{R}$ and forcing the incoming radiance to reproduce the corresponding outgoing radiance [S3]. During rendering of indirect illumination, spatially varying indirect illumination is queried from MLP $I$; visibility is not required [S3]. The incoming radiance is determined by querying the SG function at the desired surface point and direction: $L_i(\hat{\mathbf{x}}, \boldsymbol{\omega}_i) = G(\boldsymbol{\omega}_i; I(\hat{\mathbf{x}}))$ [S3]. The output dimension of the indirect illumination MLP is the SG parameters $\boldsymbol{\Gamma} \in \mathbb{R}^{24 \times 7}$ [S3].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
[S3] Modeling Indirect Illumination for Inverse Rendering, https://doi.org/10.1109/cvpr52688.2022.01809, fetched 2026-10-26
