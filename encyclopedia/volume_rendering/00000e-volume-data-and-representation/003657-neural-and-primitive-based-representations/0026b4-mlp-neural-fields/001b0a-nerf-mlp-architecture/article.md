# NeRF MLP Architecture

Also called: NeRF MLP, NeRF Network, NeRF Architecture, NeRF Model Architecture, NeRF Network Architecture, Neural Radiance Field MLP, Neural Radiance Field Network

## Overview
The NeRF MLP Architecture is a multi-layer perceptron (MLP) that maps 3D spatial coordinates and viewing directions to volume density and view-dependent emitted color, serving as the core implicit representation in Neural Radiance Fields [S1][S2][S4, S5, S6, S7, S8, S10, unofficial][S12][S13, S14, S15, S16, unofficial][S17]. It solves the problem of continuous volumetric scene representation by approximating the 5D function $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\mathbf{c}, \sigma)$, where $\mathbf{x}$ is position and $\mathbf{d}$ is viewing direction [S2][S10, unofficial]. It is distinguished from its siblings by its specific two-branch topology: a view-independent density backbone that processes only spatial coordinates, and a view-dependent color head that combines spatial features with directional encoding [S2][S8, S11, unofficial]. This separation enforces multiview consistency for geometry while allowing for complex specularities in appearance [S2, S12]. Unlike [Scene Representation Networks (SRN)](1150-scene-representation-networks-srn.md) which uses a single unified network for all outputs, or [SIREN Neural Field](1250-siren-neural-field.md) which relies on sine activations for high-frequency learning, NeRF uses ReLU activations with positional encoding and skip connections [S2][S16, unofficial].

## How it works
**Inputs and Outputs:**
The network takes a batch of 5D coordinates as input. Each coordinate consists of a 3D spatial position $\mathbf{x} = (x, y, z)$ and a 3D Cartesian unit vector $\mathbf{d} = (d_x, d_y, d_z)$ representing the viewing direction [S2][S10, unofficial]. The output is a volume density scalar $\sigma \in [0, \infty)$ and a 3D RGB color vector $\mathbf{c} = (r, g, b) \in [0, 1]$ [S2][S10, unofficial].

**Data Structures and Layout:**
The architecture consists of two main components: a Position MLP (density backbone) and a Direction MLP (color head) [S2][S8, unofficial].

1.  **Position MLP:**
    *   **Input:** Positional encoding of $\mathbf{x}$, denoted $\gamma(\mathbf{x})$.
    *   **Layers:** 8 fully-connected (linear) layers.
    *   **Width:** 256 hidden units per layer [S2][S8, S11, unofficial].
    *   **Activation:** ReLU for all layers [S2][S8, unofficial].
    *   **Skip Connection:** The original positional encoding $\gamma(\mathbf{x})$ is concatenated to the output of the 5th layer (index 4, 0-based) [S2][S8, S11, unofficial].
    *   **Output Heads:**
        *   Density Head: A linear layer mapping 256 units to 1 unit. Output $\sigma$ is rectified by ReLU (or softplus with shift, see Pitfalls) [S2][S8, S11, unofficial].
        *   Feature Head: A linear layer mapping 256 units to 256 units. Output is a feature vector $\mathbf{f}$ [S2][S8, S11, unofficial].

2.  **Direction MLP:**
    *   **Input:** Concatenation of the feature vector $\mathbf{f}$ (256 units) and the positional encoding of the viewing direction $\gamma(\mathbf{d})$ [S2][S8, S11, unofficial].
    *   **Layers:** 1 fully-connected layer [S2][S8, S11, unofficial].
    *   **Width:** 128 hidden units [S2][S8, S11, unofficial].
    *   **Activation:** ReLU [S2][S8, S11, unofficial].
    *   **Output Head:** A linear layer mapping 128 units to 3 units (RGB). Output is clamped by Sigmoid activation [S2][S8, S11, unofficial].

**Positional Encoding:**
Inputs are expanded using a positional encoding function $\gamma(\cdot)$ to map low-frequency inputs to a higher-dimensional space, mitigating spectral bias [S2][S10, unofficial]. For a 1D input $v$, the encoding is:
$$ \gamma(v) = [\sin(2^0 \pi v), \cos(2^0 \pi v), \dots, \sin(2^{L-1} \pi v), \cos(2^{L-1} \pi v)] $$
where $L$ is the number of frequency bands [S14, unofficial].
*   For position $\mathbf{x}$: $L_{pos} = 10$. Input dimension $D_{pos} = 3 + 3 \times 2 \times 10 = 63$ [S10, S11, unofficial].
*   For direction $\mathbf{d}$: $L_{dir} = 4$ (in the original paper) or $6$ (in some implementations) [S10, S14, unofficial]. Input dimension $D_{dir} = 3 + 3 \times 2 \times L_{dir}$ [S11, S14, unofficial].

**Formulas:**
The scene function is:
$$ F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\mathbf{c}, \sigma) $$
Density prediction:
$$ \sigma = \text{ReLU}(W_\sigma \mathbf{h}_8 + b_\sigma) $$
where $\mathbf{h}_8$ is the output of the 8th layer of the Position MLP [S2][S11, unofficial].
Color prediction:
$$ \mathbf{c} = \text{Sigmoid}(W_c \text{ReLU}(W_{dir} [\mathbf{f}; \gamma(\mathbf{d})]) + b_c) $$
where $[\mathbf{f}; \gamma(\mathbf{d})]$ denotes concatenation [S2][S8, S11, unofficial].

**Pseudocode:**
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
        # Output shape: (..., 3 * (1 + 2 * num_frequencies))
        # Concatenate original x, then sin/cos of 2^i * pi * x
        freq_bands = 2.0 ** torch.arange(self.num_frequencies, device=x.device).float()
        x = x.unsqueeze(-1) # (..., 3, 1)
        freq_bands = freq_bands.view(1, 1, -1) # (1, 1, num_frequencies)
        
        # sin/cos expansion
        x_sin = torch.sin(x * freq_bands * torch.pi)
        x_cos = torch.cos(x * freq_bands * torch.pi)
        
        # Concatenate original x, then sin/cos bands
        encoded = torch.cat([x, x_sin, x_cos], dim=-1)
        return encoded

class NeRFMLP(nn.Module):
    def __init__(self, pos_freqs=10, dir_freqs=4, hidden_dim=256, skip_layer=4):
        super().__init__()
        self.pos_enc = PositionalEncoding(pos_freqs)
        self.dir_enc = PositionalEncoding(dir_freqs)
        
        # Position MLP: 8 layers of 256 units
        # Layer 0: input_dim -> hidden_dim
        # Layers 1-7: hidden_dim -> hidden_dim
        # Skip connection at layer 4 (index 4)
        self.pos_layers = nn.ModuleList()
        
        # First layer
        self.pos_layers.append(nn.Linear(3 * (1 + 2 * pos_freqs), hidden_dim))
        
        # Middle layers
        for _ in range(1, 8):
            self.pos_layers.append(nn.Linear(hidden_dim, hidden_dim))
            
        # Density head: 256 -> 1
        self.sigma_head = nn.Linear(hidden_dim, 1)
        
        # Feature head: 256 -> 256
        self.feature_proj = nn.Linear(hidden_dim, hidden_dim)
        
        # Direction MLP: 1 layer of 128 units
        # Input: hidden_dim (features) + dir_enc_dim
        dir_enc_dim = 3 * (1 + 2 * dir_freqs)
        self.dir_layer = nn.Linear(hidden_dim + dir_enc_dim, 128)
        
        # Color head: 128 -> 3
        self.color_head = nn.Linear(128, 3)
        
        self.skip_layer = skip_layer

    def forward(self, positions, directions):
        # positions: (N, 3)
        # directions: (N, 3)
        
        # 1. Encode inputs
        pos_emb = self.pos_enc(positions)
        dir_emb = self.dir_enc(directions)
        
        # 2. Process through Position MLP
        x = pos_emb
        for i, layer in enumerate(self.pos_layers):
            if i == self.skip_layer:
                # Concatenate original positional encoding
                x = torch.cat([x, pos_emb], dim=-1)
            x = F.relu(layer(x))
            
        # 3. Extract density and features
        # Density: ReLU of linear output
        sigma = F.relu(self.sigma_head(x))
        
        # Features: Linear projection (no activation specified in original, but often identity or sigmoid in variants)
        # Original paper uses linear layer for features, no activation mentioned for feature vector itself before concat
        feat = self.feature_proj(x)
        
        # 4. Process through Direction MLP
        # Concatenate features with direction encoding
        dir_input = torch.cat([feat, dir_emb], dim=-1)
        
        # 1 Hidden Layer
        dir_out = F.relu(self.dir_layer(dir_input))
        
        # 5. Output Color
        # Sigmoid activation for RGB
        rgb = torch.sigmoid(self.color_head(dir_out))
        
        return rgb, sigma
```

**Parameters:**
*   `pos_freqs` ($L_{pos}$): 10 [S10, S14, unofficial].
*   `dir_freqs` ($L_{dir}$): 4 [S10, unofficial] or 6 [S14, unofficial].
*   `hidden_dim`: 256 [S2][S8, S13, unofficial].
*   `skip_layer`: 4 (0-based index, corresponding to the 5th layer) [S2][S8, S11, unofficial].
*   `netwidth`: 256 [S5, unofficial].
*   `netdepth`: 8 [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **View Dependence:** Removing the viewing direction input prevents the model from representing specular reflections [S2, S12].
*   **Density Non-negativity:** Volume density $\sigma$ must be non-negative. The original paper applies ReLU to the density head output [S2][S11, unofficial]. Some implementations use `softplus(h - 1)` for sparser initialization [S8, unofficial].
*   **Color Range:** RGB values must be in $[0, 1]$. Sigmoid activation is applied to the final color layer [S2][S8, S11, unofficial].
*   **Implementation Discrepancy:** The original NeRF paper supplement states there are 4 hidden layers in the view-dependent branch, but the official code and most implementations use only 1 hidden layer (128 units) [S5, S15, unofficial].
*   **Spectral Bias:** Without positional encoding, the MLP struggles to learn high-frequency details due to spectral bias [S2][S16, unofficial].

## Subtopics

### [Unbounded inward-facing scene parameterization](957-unbounded-inward-facing-scene-parameterization.md)
Parameterizes unbounded 360-degree scenes by mapping coordinates to a bounded domain, enabling voxel grid acceleration.

### [Position-Only NeRF Field](1253-position-only-nerf-field.md)
Omits viewing direction input to reduce model size, assuming isotropic appearance.

### [Tiny NeRF](1275-tiny-nerf.md)
Compresses voxel-based NeRFs via DCT and quantization.

### [MLPRender_Fea](2414-mlprender-fea.md)
Predicts RGB from latent features and viewing directions using an MLP.

### [View-Dependent MLP](2444-view-dependent-mlp.md)
Replaces iNGP's bottleneck with a larger view-dependent MLP to reduce floaters.

### [NeUDF Network Architecture](3732-neudf-network-architecture.md)
Represents geometry as UDF and color separately using two MLPs.

### [InstantNGP Network Architecture](5049-instantngp-network-architecture.md)
Uses multi-resolution hash encoding for real-time rendering.

### Recursive Neural Field
A sub-network $F_{\Phi_i}$ that takes its parent branch's output $y_{p_i}$ and viewing direction $d$ as inputs, predicting color $c_i$, density $\sigma_i$, uncertainty $\delta_i$, and a latent vector $y_i$ [S18]. The sub-network consists of an MLP module, a Branch module, and an Out module [S18]. The Branch module predicts uncertainty, forwards low-uncertainty points to the Out module, and distributes high-uncertainty points to different sub-networks based on distances to cluster centers [S18]. The root sub-network $F_{\Phi_1}$ takes the query coordinate $(x, y, z)$ as $y_{p_i}$ [S18].

### VFR Network Architecture
Uses a Multiresolution Hash Grid (MHG) for feature representation. The Color NN consists of a Spatial MLP (2 layers, 256 hidden neurons) and a Directional MLP (4 layers, 256 hidden neurons) with GELU activation and a 256-dimensional bottleneck feature [S19]. Optimized with Adam (lr 0.01) on RTX 3090 [S19].

### CT-NeRF Network Architecture
Follows the BARF approach, using a single 8-layer MLP network with a width of 128 [S20, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is slow, requiring 12+ hours per scene on a single GPU (e.g., NVIDIA V100) due to the need for millions of network queries per image [S2]. Rendering a single frame takes approximately 30 seconds on a V100 [S2].
*   **Memory Complexity:** The model weights are small (~5 MB), significantly less than input images or voxel grids [S2]. However, storing the positional encoding parameters and intermediate activations during training can be memory-intensive for large batches.
*   **Performance:** Achieves state-of-the-art view synthesis quality on forward-facing scenes, outperforming SRN and LLFF in PSNR, SSIM, and LPIPS [S2].
*   **Strengths:** High-quality novel view synthesis, compact representation, differentiable rendering allows optimization from images alone [S2].
*   **Limitations:** Slow training and rendering times, requires careful tuning of positional encoding frequencies, struggles with unbounded scenes without parameterization [S2, S17].
*   **When to use:** When high-quality novel views are needed and training time is not a constraint.
*   **When not to use:** For real-time applications or unbounded scenes without additional parameterization.

## Background
The NeRF MLP Architecture was introduced by Mildenhall et al. in the paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (ECCV 2020) [S2]. It builds on the concept of neural fields and addresses the limitations of previous methods like SRN and Neural Volumes by using positional encoding and a specific two-branch architecture to handle high-frequency details and view-dependent appearance [S2]. The method optimizes the MLP weights by minimizing the difference between rendered and ground truth images using gradient descent [S2, S17].

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
