# Volume Radiance Network

Also called: VolumeRadiance

## Overview
The Volume Radiance Network is a neural rendering module that predicts RGB color from input features, view directions, and positions using a Multi-Layer Perceptron (MLP) with positional encoding applied to features and view direction, but NOT position. It serves as the color prediction head in neural implicit representations, building on [MLPRender_Fea](2414-mlprender-fea.md). Unlike its sibling [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which injects global scene context, or [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md) which uses linear projections, this method employs a non-linear MLP to map concatenated feature and encoded direction vectors to color space.

## How it works
**Inputs and Outputs:**
- Input: A batch of input features $\mathbf{F} \in \mathbb{R}^{N \times D_{feat}}$, view directions $\mathbf{D} \in \mathbb{R}^{N \times 3}$, and optional additional arguments.
- Output: Predicted RGB colors $\mathbf{C} \in \mathbb{R}^{N \times 3}$.

**Data Structures:**
The implementation is encapsulated in a class `VolumeRadiance` with the following attributes:
- `n_dir_dims`: Integer, dimensionality of direction encoding (default 3).
- `n_output_dims`: Integer, dimensionality of output (default 3 for RGB).
- `encoding`: An encoding object (e.g., positional encoder) instantiated via `get_encoding` from `dir_encoding_config`.
- `n_input_dims`: Integer, total input dimension to the MLP, calculated as $D_{feat} + D_{enc}$, where $D_{enc}$ is the output dimension of `encoding`.
- `network`: An MLP object instantiated via `get_mlp` from `mlp_network_config`.

**Parameters:**
- `input_feature_dim`: Dimension of the input feature vector.
- `dir_encoding_config`: Configuration dictionary for the direction encoder.
- `mlp_network_config`: Configuration dictionary for the MLP (layers, hidden size, activation).
- `color_activation`: Optional activation function applied to the final color output.

**Steps:**
1. **Direction Normalization**: Normalize view directions from the range $(-1, 1)$ to $(0, 1)$ using the formula:
   $$ \mathbf{D}_{norm} = \frac{\mathbf{D} + 1.0}{2.0} $$
   [S1, unofficial]

2. **Direction Encoding**: Apply the positional encoding to the normalized directions. Let $\gamma(\cdot)$ denote the encoding function. The encoded directions are:
   $$ \mathbf{D}_{enc} = \gamma(\mathbf{D}_{norm}) $$
   The shape of $\mathbf{D}_{enc}$ is $(N, D_{enc})$. [S1, unofficial]

3. **Input Concatenation**: Flatten the input features (if not already flat) and concatenate them with the encoded directions and any additional arguments along the last dimension to form the network input tensor $\mathbf{X}_{inp}$:
   $$ \mathbf{X}_{inp} = \text{concat}(\mathbf{F}_{flat}, \mathbf{D}_{enc}, \dots) $$
   The shape of $\mathbf{X}_{inp}$ is $(N, n\_input\_dims)$. [S1, unofficial]

4. **MLP Forward Pass**: Pass $\mathbf{X}_{inp}$ through the MLP network:
   $$ \mathbf{Y} = \text{MLP}(\mathbf{X}_{inp}) $$
   The output $\mathbf{Y}$ has shape $(N, n\_output\_dims)$. [S1, unofficial]

5. **Reshape**: Reshape $\mathbf{Y}$ to match the expected output shape, typically $(N, 3)$ for RGB. [S1, unofficial]

6. **Color Activation**: If `color_activation` is specified in the configuration, apply it to the output:
   $$ \mathbf{C} = \text{color\_activation}(\mathbf{Y}) $$
   If not specified, $\mathbf{C} = \mathbf{Y}$. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class VolumeRadiance(nn.Module):
    def __init__(self, input_feature_dim, dir_encoding_config, mlp_network_config, color_activation=None):
        super().__init__()
        self.n_dir_dims = dir_encoding_config.get('n_dir_dims', 3)
        self.n_output_dims = 3
        # Instantiate encoder based on config
        self.encoding = get_encoding(dir_encoding_config) 
        # Calculate input dimensions for MLP
        self.n_input_dims = input_feature_dim + self.encoding.out_dim
        # Instantiate MLP based on config
        self.network = get_mlp(mlp_network_config, self.n_input_dims, self.n_output_dims)
        self.color_activation = color_activation

    def forward(self, features, directions, **kwargs):
        # Step 1: Normalize directions from (-1, 1) to (0, 1)
        # directions shape: (N, 3)
        directions_norm = (directions + 1.0) / 2.0
        
        # Step 2: Encode directions
        # encoding output shape: (N, D_enc)
        directions_enc = self.encoding(directions_norm)
        
        # Step 3: Concatenate features and encoded directions
        # features shape: (N, D_feat)
        # Ensure features are flattened if necessary
        features_flat = features.view(features.size(0), -1)
        
        # Concatenate along last dimension
        network_inp = torch.cat([features_flat, directions_enc], dim=-1)
        
        # Step 4: Pass through MLP
        # network output shape: (N, 3)
        output = self.network(network_inp)
        
        # Step 5: Reshape if necessary (already 3D in this case)
        # output is (N, 3)
        
        # Step 6: Apply color activation if specified
        if self.color_activation is not None:
            output = self.color_activation(output)
            
        return output
```
[G] `get_encoding` and `get_mlp` are assumed to be helper functions that instantiate the specific encoder (e.g., positional encoding) and MLP architecture defined in the respective config dictionaries. The MLP typically consists of linear layers with ReLU activations, ending with a linear layer to produce the output dimension.

## Subtopics

### [Disco NSVF Models](11063-disco-nsvf-models.md)
Disco NSVF models represent 3D scenes using a disentangled radiance field, registered as `disco_nsvf` and `multi_disco_nsvf`. They differ from their sibling [SDF NSVF Models](11064-sdf-nsvf-models.md) by utilizing a disentangled radiance field representation rather than a unified SDF-based approach.

### [Neural Irradiance Volume](14514-neural-irradiance-volume.md)
Neural Irradiance Volume (NIV) is a real-time rendering technique that compresses the 5D indirect diffuse irradiance field of a static scene into a compact neural model, enabling the shading of unseen dynamic objects without runtime ray tracing or denoising. Unlike traditional grid-based probe methods which suffer from aliasing and cubic memory scaling, NIV maps position and direction directly to irradiance values, improving quality by at least 10x at the same memory budget. It differs from other neural rendering methods by learning a pre-integrated volumetric quantity (irradiance) rather than raw incident radiance, allowing noise-free shading at ~1 ms per frame on consumer hardware.

### High-Dimensional NeRF
High-Dimensional NeRF is a modification of NeRF that maps 3D points to high-dimensional features $\mathbf{f}_k \in \mathbb{R}^{dim}$ instead of RGB color, allowing representation of semantics and materials [S2]. The MLP prediction is defined as:
$$ [\mathbf{f}_k; \sigma_k] = \text{MLP}(\gamma_x(\mathbf{x}_k), \gamma_d(\mathbf{d}_u); \Theta) $$
where $\mathbf{f}_k$ is the high-dimensional feature of point $\mathbf{x}_k$ [S2]. Ray feature aggregation is performed via volume rendering:
$$ \mathbf{f}_{r_u} = \sum_k \tau_k (1 - \exp(-\sigma_k \delta_k)) \mathbf{f}_k $$
where $\mathbf{f}_{r_u}$ is the aggregated ray feature [S2]. This allows the network to encode richer information such as semantics and materials, supervised in both feature and color space [S2].

## Operating characteristics
**Time and Memory Complexity:**
The time complexity is dominated by the MLP forward pass, which is linear with respect to the number of samples $N$ and the MLP size. The memory complexity is determined by the MLP parameters and the encoding storage.

**Strengths and Limitations:**
The method is flexible and can model complex view-dependent effects through the direction encoding. However, it relies on the MLP's capacity to approximate the radiance function, which may require significant depth and width for high-fidelity results. The lack of positional encoding for position (as per the parent definition) implies that spatial smoothness is handled by the input features or the MLP's inherent inductive bias, which may limit its ability to capture high-frequency spatial details compared to methods that encode position directly.

**When to Use:**
Use this method when the scene representation is feature-based (e.g., from a 3D CNN or a separate geometry network) and view-dependent shading is required. It is suitable for applications where the color prediction needs to be decoupled from the geometry representation.

## Background
The Volume Radiance Network is a component of neural implicit representations, specifically within the framework of neural radiance fields (NeRF) and its variants. It was introduced as part of the instant-nsr-pl implementation, which provides a unified platform for neural surface reconstruction and radiance field learning [S1, unofficial]. The design follows the general principle of using MLPs to predict radiance from spatial and directional inputs, a concept popularized by NeRF [S2].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
