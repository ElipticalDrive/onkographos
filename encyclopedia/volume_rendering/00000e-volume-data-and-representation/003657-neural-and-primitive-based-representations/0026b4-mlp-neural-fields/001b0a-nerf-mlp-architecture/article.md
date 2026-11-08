# NeRF MLP Architecture

Also called: NeRF MLP, NeRF Network, NeRF Architecture, NeRF Model Architecture, NeRF Network Architecture, Neural Radiance Field MLP, Neural Radiance Field Network

## Overview
The NeRF MLP Architecture is a multi-layer perceptron (MLP) that implicitly represents a 3D scene as a continuous volumetric function, mapping 3D spatial coordinates and viewing directions to volume density and view-dependent emitted color [S2]. It solves the problem of view synthesis by optimizing a neural network to minimize the error between rendered images and observed input images, enabling the generation of photorealistic novel views from sparse input views [S2]. What sets it apart from its siblings, such as [Scene Representation Networks (SRN)](1150-scene-representation-networks-srn.md) and [SIREN Neural Field](1250-siren-neural-field.md), is its specific two-branch design that enforces multiview consistency by predicting density as a function of position only, while predicting color as a function of both position and viewing direction, combined with the use of positional encoding to overcome spectral bias [S2][S10, unofficial].

## How it works
The architecture defines a scene function $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\mathbf{c}, \sigma)$, where $\mathbf{x} = (x, y, z)$ is the 3D spatial location, $\mathbf{d}$ is the 3D Cartesian unit vector representing the viewing direction (derived from spherical coordinates $(\theta, \phi)$), $\mathbf{c} = (r, g, b)$ is the emitted color, and $\sigma$ is the volume density [S2]. The network is trained by minimizing the squared error between rendered and ground truth pixel colors [S17].

### Inputs and Outputs
*   **Inputs**: A batch of 3D positions $\mathbf{x} \in \mathbb{R}^{N \times 3}$ and viewing directions $\mathbf{d} \in \mathbb{R}^{N \times 3}$.
*   **Outputs**: A batch of volume densities $\sigma \in \mathbb{R}^{N \times 1}$ and RGB colors $\mathbf{c} \in \mathbb{R}^{N \times 3}$.

### Data Structures and Layout
The network consists of two main components: a position MLP (density backbone) and a color head (view-dependent branch).

1.  **Positional Encoding**: Inputs are expanded using positional encoding $\gamma(\cdot)$. For an input coordinate $v$, the encoding is:
    $$ \gamma(v) = [\cos(2^0 \pi v), \sin(2^0 \pi v), \dots, \cos(2^{L-1} \pi v), \sin(2^{L-1} \pi v)] $$
    The original input $v$ is often concatenated. For position $\mathbf{x}$, $L_{pos}=10$, resulting in an input dimension $D_{pos} = 3 \times (1 + 2L_{pos}) = 63$ [S10, unofficial][S14, unofficial]. For direction $\mathbf{d}$, $L_{dir}=4$ (or 6 in some implementations), resulting in $D_{dir} = 3 \times (1 + 2L_{dir}) = 27$ (or 39) [S10, unofficial][S14, unofficial].

2.  **Position MLP (Density Backbone)**:
    *   **Layers**: 8 fully-connected (linear) layers.
    *   **Width**: 256 hidden units per layer [S2].
    *   **Activation**: ReLU for all layers except the final output layer of the backbone [S2].
    *   **Skip Connections**: The original positional encoding $\gamma(\mathbf{x})$ is concatenated to the hidden state at layer index 4 (0-indexed, i.e., after the 5th linear layer) [S2][S10, unofficial].
    *   **Output**: The final layer of the backbone outputs a vector of size 257. The first element is processed to produce $\sigma$, and the remaining 256 elements form the feature vector $\mathbf{f}$ [S2][S10, unofficial].

3.  **Color Head (View-Dependent Branch)**:
    *   **Input**: Concatenation of the feature vector $\mathbf{f}$ (256 dims) and the directional encoding $\gamma(\mathbf{d})$ [S2].
    *   **Layers**: 1 fully-connected layer with 128 hidden units [S2].
    *   **Activation**: ReLU for the hidden layer, Sigmoid for the final output [S2].
    *   **Output**: 3 units representing RGB color [S2].

4.  **Density Output Processing**:
    *   The raw output of the backbone's density head is processed to ensure non-negativity. The original paper uses a softplus activation shifted by -1: $\sigma = \text{softplus}(h - 1)$, where $h$ is the linear output [S8, unofficial]. Some implementations use ReLU: $\sigma = \text{ReLU}(h)$ [S11, unofficial].

### Parameters
*   **Hidden Layers (Position)**: 8 layers, width 256 [S2].
*   **Hidden Layers (Color)**: 1 layer, width 128 [S2].
*   **Positional Encoding Depth**: $L_{pos}=10$ for position, $L_{dir}=4$ for direction [S10, unofficial].
*   **Skip Connection Index**: 4 (after the 5th layer) [S2].
*   **Weight Initialization**: Weights initialized from Uniform$(-1/\sqrt{\text{out\_features}}, 1/\sqrt{\text{out\_features}})$, biases to 0 [S14, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PositionalEncoding(nn.Module):
    def __init__(self, num_frequencies):
        super().__init__()
        self.num_frequencies = num_frequencies

    def forward(self, x):
        # x shape: (..., 3)
        # Generate frequencies: 2^0, 2^1, ..., 2^(L-1)
        freq_bands = 2.0 ** torch.arange(self.num_frequencies, device=x.device).float()
        # Reshape for broadcasting: (..., 3, 1) * (1, 1, num_frequencies)
        freq_bands = freq_bands.view(1, 1, -1)
        
        # Expand x to (..., 3, num_frequencies)
        x_expanded = x.unsqueeze(-1) * freq_bands
        
        # Apply sin and cos
        s = torch.sin(x_expanded)
        c = torch.cos(x_expanded)
        
        # Concatenate sin/cos and original x
        # Output shape: (..., 3 * 2 * num_frequencies + 3)
        return torch.cat([x, s, c], dim=-1)

class NeRFMLP(nn.Module):
    def __init__(self, pos_freqs=10, dir_freqs=4, hidden_dim=256, color_hidden_dim=128):
        super().__init__()
        
        # Positional Encoders
        self.pos_enc = PositionalEncoding(pos_freqs)
        self.dir_enc = PositionalEncoding(dir_freqs)
        
        # Input dimensions after encoding
        # 3 original + 2 * num_frequencies * 3
        self.pos_input_dim = 3 + 2 * pos_freqs * 3
        self.dir_input_dim = 3 + 2 * dir_freqs * 3
        
        # Position MLP (Density Backbone)
        # Layer 0: pos_input_dim -> hidden_dim
        self.fc1 = nn.Linear(self.pos_input_dim, hidden_dim)
        # Layers 1-3: hidden_dim -> hidden_dim
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.fc3 = nn.Linear(hidden_dim, hidden_dim)
        self.fc4 = nn.Linear(hidden_dim, hidden_dim)
        # Layer 4 (index 4, 5th layer): hidden_dim + pos_input_dim -> hidden_dim (Skip connection)
        self.fc5 = nn.Linear(hidden_dim + self.pos_input_dim, hidden_dim)
        # Layers 5-7: hidden_dim -> hidden_dim
        self.fc6 = nn.Linear(hidden_dim, hidden_dim)
        self.fc7 = nn.Linear(hidden_dim, hidden_dim)
        self.fc8 = nn.Linear(hidden_dim, hidden_dim)
        
        # Output heads from the last hidden layer
        # Density head: 1 output
        self.sigma_head = nn.Linear(hidden_dim, 1)
        # Feature projection: hidden_dim outputs
        self.feature_proj = nn.Linear(hidden_dim, hidden_dim)
        
        # Color Head
        # Input: hidden_dim (features) + dir_input_dim (direction encoding)
        self.color_fc1 = nn.Linear(hidden_dim + self.dir_input_dim, color_hidden_dim)
        self.color_head = nn.Linear(color_hidden_dim, 3)

    def forward(self, positions, directions):
        """
        positions: (N, 3)
        directions: (N, 3)
        """
        # 1. Encode inputs
        pos_encoded = self.pos_enc(positions)
        dir_encoded = self.dir_enc(directions)
        
        # 2. Position MLP
        x = pos_encoded
        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))
        x = F.relu(self.fc3(x))
        x = F.relu(self.fc4(x))
        
        # Skip connection at layer index 4 (after 5th linear layer)
        x = torch.cat([x, pos_encoded], dim=-1)
        x = F.relu(self.fc5(x))
        
        x = F.relu(self.fc6(x))
        x = F.relu(self.fc7(x))
        x = F.relu(self.fc8(x))
        
        # 3. Density and Feature Extraction
        # Density: softplus(h - 1) for sparser initialization [S8, unofficial]
        raw_sigma = self.sigma_head(x)
        sigma = F.softplus(raw_sigma - 1)
        
        # Feature vector
        features = self.feature_proj(x)
        
        # 4. Color Head
        # Concatenate features and direction encoding
        color_input = torch.cat([features, dir_encoded], dim=-1)
        color_hidden = F.relu(self.color_fc1(color_input))
        rgb = torch.sigmoid(self.color_head(color_hidden))
        
        return sigma, rgb
```

### Edge Cases and Pitfalls
*   **Spectral Bias**: Standard MLPs struggle with high-frequency details. Positional encoding is critical; removing it results in oversmoothed renderings [S2].
*   **View Dependence**: Removing the viewing direction input prevents the model from representing specular reflections and non-Lambertian effects [S2].
*   **Density Non-Negativity**: Volume density must be non-negative. Using ReLU or shifted softplus is required [S2][S8, unofficial].
*   **Color Range**: RGB values must be in $[0, 1]$. Sigmoid activation is used [S2].
*   **Implementation Discrepancy**: The original paper supplement mentions 4 hidden layers in the view-dependent branch, but the official code and experiments used only 1 hidden layer [S5, unofficial][S15, unofficial].

## Subtopics

### [Unbounded inward-facing scene parameterization](957-unbounded-inward-facing-scene-parameterization.md)
Parameterizes unbounded 360-degree scenes by mapping coordinates to a bounded domain, enabling the use of voxel grids.

### [Position-Only NeRF Field](1253-position-only-nerf-field.md)
Omits viewing direction input to reduce model size, assuming isotropic appearance.

### [Tiny NeRF](1275-tiny-nerf.md)
Compresses voxel-based NeRFs using DCT and pruning.

### [MLPRender_Fea](2414-mlprender-fea.md)
Predicts RGB from latent features and viewing directions using an MLP.

### [View-Dependent MLP](2444-view-dependent-mlp.md)
Replaces the bottleneck in iNGP with a larger view-dependent MLP to reduce floaters.

### [NeUDF Network Architecture](3732-neudf-network-architecture.md)
Represents geometry as a UDF and color separately using two MLPs.

### [InstantNGP Network Architecture](5049-instantngp-network-architecture.md)
Combines multi-resolution hash encoding with compact MLPs for fast training and querying.

### Recursive Neural Field
A sub-network $F_{\Phi_i}$ that takes its parent branch's output $y_{p_i}$ and viewing direction $d$ as inputs, predicting color $c_i$, density $\sigma_i$, uncertainty $\delta_i$, and a latent vector $y_i$ [S18]. The Branch module predicts uncertainty and distributes high-uncertainty points to different sub-networks based on distances to cluster centers [S18].

### VFR Network Architecture
Uses a Multiresolution Hash Grid for feature representation and a Spatial MLP (2 layers, 256 hidden neurons) + Directional MLP (4 layers, 256 hidden neurons) with GELU activation [S19].

### CT-NeRF Network Architecture
Follows the BARF approach with a single 8-layer MLP network with a width of 128 [S20, unofficial].

## Operating characteristics
*   **Time Complexity**: Training is slow, typically taking 12+ hours per scene on a single GPU [S2]. Rendering requires many network queries (e.g., 150-200 million per image), taking ~30 seconds per frame on a V100 [S2].
*   **Memory Complexity**: The network weights are small (~5 MB), significantly less than voxel-based methods [S2].
*   **Strengths**: High-quality novel view synthesis, multiview consistency, compact storage [S2].
*   **Limitations**: Slow training and rendering, struggles with unbounded scenes without parameterization, spectral bias without positional encoding [S2].
*   **When to Use**: When high-quality novel views are needed from sparse inputs and computational resources allow for long training times.
*   **When Not to Use**: For real-time applications or unbounded scenes without additional parameterization.

## Background
The NeRF MLP Architecture was introduced by Mildenhall et al. in the paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (ECCV 2020) [S2]. It addresses the limitations of previous neural rendering methods like SRN by using a fully-connected MLP with positional encoding and a two-branch design to model view-dependent appearance and multiview consistency [S2]. The method optimizes the network weights to minimize the difference between rendered and ground truth images, leveraging differentiable volume rendering [S2].

## Sources
[S1] Efficient View Synthesis with Neural Radiance Distribution Field, https://doi.org/10.1109/iccv51070.2023.01696, fetched 2026-10-26
[S2] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S3] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
[S4] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S5] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S6] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S7] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S8] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S9] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S10] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S11] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S12] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S13] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S14] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S15] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S16] sr-bang/NeRF, https://github.com/sr-bang/NeRF, fetched 2026-10-27 (unofficial)
[S17] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
[S18] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S19] PDF Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://papers.neurips.cc/paper_files/paper/2023/file/ce182e31662883d4decc84a0255335b6-Paper-Conference.pdf, fetched 2026-10-27
[S20] CT-NeRF: Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory, https://doi.org/10.48550/arxiv.2404.13896, fetched 2026-10-26 (unofficial)
