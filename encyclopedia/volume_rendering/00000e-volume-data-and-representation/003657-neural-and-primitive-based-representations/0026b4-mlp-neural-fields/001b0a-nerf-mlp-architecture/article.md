# NeRF MLP Architecture

Also called: NeRF MLP, Neural Radiance Field MLP

## Overview
The NeRF MLP Architecture is a multi-layer perceptron (MLP) that maps 3D spatial coordinates and viewing directions to volume density and view-dependent color, serving as the core implicit representation in Neural Radiance Fields [S1, S2, S3]. It solves the problem of continuous scene representation by learning a 5D function $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\mathbf{c}, \sigma)$ that outputs density $\sigma$ and RGB color $\mathbf{c}$ for any point in space [S2][S11, unofficial]. What sets it apart from siblings like [Scene Representation Networks (SRN)](1150-scene-representation-networks-srn.md) is its specific two-branch design: a view-independent density backbone and a view-dependent color head, combined with sinusoidal positional encoding to overcome spectral bias [S2][S12, unofficial]. Unlike [SIREN Neural Field](1250-siren-neural-field.md) which uses periodic activations throughout, the standard NeRF MLP uses ReLU activations with skip connections [S2][S18, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A batch of 5D coordinates. The spatial position $\mathbf{x} = (x, y, z)$ and the viewing direction $\mathbf{d}$ (typically a 3D Cartesian unit vector derived from spherical coordinates $(\theta, \phi)$) [S2][S11, unofficial].
*   **Output:** A volume density scalar $\sigma$ and a 3D RGB color vector $\mathbf{c} = (r, g, b)$ [S2][S11, unofficial].

### Data Structures
The architecture consists of two main neural network components implemented as fully connected layers:
1.  **Position MLP ($MLP_{pos}$):** An 8-layer network.
    *   Input dimension: $D_{pos} = 3 \times (1 + 2L_{pos})$, where $L_{pos}$ is the number of frequency bands for positional encoding (default 10) [S12, S16, unofficial].
    *   Hidden layers: 8 layers, each with $W=256$ neurons [S2][S12, unofficial].
    *   Skip connection: The original encoded position input is concatenated to the output of the 5th layer (index 4, 0-based) [S2][S12, unofficial].
    *   Output: A scalar $\sigma$ and a 256-dimensional feature vector $\mathbf{e}$ [S2][S12, unofficial].
2.  **Color MLP ($MLP_{rgb}$):** A 1-layer network (plus final output layer).
    *   Input: Concatenation of the feature vector $\mathbf{e}$ (256 dims) and the encoded viewing direction $\gamma(\mathbf{d})$ [S2][S12, unofficial].
    *   Hidden layer: 1 layer with $W_{dir}=128$ neurons [S2][S12, unofficial].
    *   Output: 3 RGB values [S2][S12, unofficial].

### Positional Encoding
Inputs are mapped to a higher-dimensional space using Fourier features (positional encoding) to enable the network to learn high-frequency details [S2][S16, unofficial].
For an input coordinate $v \in \{x, y, z, d_x, d_y, d_z\}$ and frequency band $l \in [0, L-1]$, the encoding is:
$$ \gamma(v)_l = (\sin(2^l \pi v), \cos(2^l \pi v)) $$
The full encoding vector for position $\mathbf{x}$ is:
$$ \gamma(\mathbf{x}) = [\mathbf{x}, \sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L_{pos}-1} \pi \mathbf{x}), \cos(2^{L_{pos}-1} \pi \mathbf{x})] $$
[S3][S16, unofficial].
Typical values: $L_{pos}=10$ for position, $L_{dir}=4$ or $6$ for direction [S11, S16, unofficial].

### Algorithm Steps
1.  **Encode Inputs:** Apply positional encoding $\gamma(\mathbf{x})$ to the 3D spatial coordinates and $\gamma(\mathbf{d})$ to the viewing direction [S2][S16, unofficial].
2.  **Process Position (Density Backbone):**
    *   Pass $\gamma(\mathbf{x})$ through the first linear layer ($D_{pos} \rightarrow 256$).
    *   Apply ReLU activation.
    *   Iterate through layers 2 to 8. For each layer $i$ (1 to 7, 0-based index 1 to 7):
        *   If $i == 4$ (the 5th layer), concatenate the original $\gamma(\mathbf{x})$ to the current hidden state [S2][S12, unofficial].
        *   Apply linear transformation ($256 \rightarrow 256$).
        *   Apply ReLU activation [S2][S12, unofficial].
3.  **Extract Density and Features:**
    *   The output of the 8th layer is split.
    *   **Density:** Pass the output through a linear layer ($256 \rightarrow 1$). Apply ReLU activation to ensure $\sigma \ge 0$. Alternatively, some implementations use $\sigma = \text{softplus}(h - 1)$ for sparser initialization [S9, S16, unofficial].
    *   **Feature Vector:** The remaining 256 dimensions form the feature vector $\mathbf{e}$ [S2][S12, unofficial].
4.  **Process Color (View-Dependent Head):**
    *   Concatenate the feature vector $\mathbf{e}$ with the encoded direction $\gamma(\mathbf{d})$ [S2][S12, unofficial].
    *   Pass through a linear layer ($256 + D_{dir} \rightarrow 128$).
    *   Apply ReLU activation [S2][S12, unofficial].
    *   Pass through a final linear layer ($128 \rightarrow 3$).
    *   Apply Sigmoid activation to clamp RGB values to $[0, 1]$ [S2][S16, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def positional_encoding(x, num_frequencies):
    """
    Applies Fourier feature positional encoding.
    x: Tensor of shape (..., 3)
    num_frequencies: int L
    Returns: Tensor of shape (..., 3 * (1 + 2*L))
    """
    # x shape: (..., 3)
    # freq_bands shape: (num_frequencies, 2)
    freq_bands = 2.0 ** torch.arange(num_frequencies, device=x.device).float()
    # Expand x to (..., 1, 3) and freq_bands to (1, num_frequencies, 2)
    # We want sin(2^l * pi * x) and cos(2^l * pi * x)
    
    # Reshape for broadcasting: x -> (..., 1, 3), freq -> (1, L, 1)
    x_expanded = x.unsqueeze(-2) # (..., 1, 3)
    freqs = freq_bands.unsqueeze(-1) * torch.pi # (L, 1)
    
    # Compute sin and cos
    # sin_input: (..., L, 3)
    sin_input = x_expanded * freqs.unsqueeze(-2) 
    cos_input = sin_input.clone()
    
    sin_vals = torch.sin(sin_input) # (..., L, 3)
    cos_vals = torch.cos(cos_input) # (..., L, 3)
    
    # Concatenate original x, then sin/cos pairs
    # Original x is (..., 3)
    # sin_vals is (..., L, 3) -> reshape to (..., 2*L) ? No, keep dims for concat
    # Standard NeRF concat: [x, sin(2^0 x), cos(2^0 x), ...]
    # So we stack sin and cos along the last dimension, then concat x
    
    # sin_vals shape: (..., L, 3)
    # cos_vals shape: (..., L, 3)
    # We want to interleave or just stack? 
    # Eq 3 in [S3]: (sin(2^0 pi x), cos(2^0 pi x), ..., sin(2^{L-1} pi x), cos(2^{L-1} pi x))
    # This implies shape (..., 2*L*3) if flattened, or (..., L, 6)
    
    # Let's follow [S16, unofficial] structure: output = [x, sin..., cos...]
    # Actually [S16, unofficial] says: output = [x, sin(2^0 x), cos(2^0 x), ...]
    # This results in dim 3 + 2*L*3.
    
    sin_vals = sin_vals.flatten(start_dim=-2) # (..., L*3)
    cos_vals = cos_vals.flatten(start_dim=-2) # (..., L*3)
    
    # Concatenate: x, then sin, then cos? Or x, then sin/cos pairs?
    # [S16, unofficial] code comment: "concatenate sin/cos of 2^i * x for i in [0, L-1]"
    # Usually implemented as: [x, sin(2^0 x), cos(2^0 x), sin(2^1 x), cos(2^1 x)...]
    # Which is shape (..., 3 + 2*L*3)
    
    # Let's construct the pairs
    # sin_vals: (..., L*3)
    # cos_vals: (..., L*3)
    # We need to interleave them? Or just stack?
    # If we stack sin and cos, we get (..., 2*L*3).
    # Then concat x.
    
    # Correct interleaving for "sin(2^0), cos(2^0), sin(2^1), cos(2^1)"
    # sin_vals is (..., L, 3) -> flatten to (..., L*3) loses pair structure if not careful.
    # Better: reshape sin_vals to (..., L, 3) and cos_vals to (..., L, 3)
    # Stack along dim -1: (..., L, 6)
    # Flatten last two dims: (..., 6*L)
    
    paired = torch.cat([sin_vals, cos_vals], dim=-1) # (..., L*6)
    
    return torch.cat([x, paired], dim=-1)

class NeRFMLP(nn.Module):
    def __init__(self, pos_enc_len=10, dir_enc_len=4, hidden_dim=256, skip_layer=4):
        super().__init__()
        self.pos_enc_len = pos_enc_len
        self.dir_enc_len = dir_enc_len
        
        # Input dimensions
        self.input_ch = 3 * (1 + 2 * pos_enc_len)
        self.input_ch_views = 3 * (1 + 2 * dir_enc_len)
        
        # Position MLP
        # Layer 0: input_ch -> hidden_dim
        self.linear1 = nn.Linear(self.input_ch, hidden_dim)
        
        # Layers 1-7: hidden_dim -> hidden_dim
        self.linears = nn.ModuleList([
            nn.Linear(hidden_dim, hidden_dim) for _ in range(7)
        ])
        
        # Skip connection layer at skip_layer (index 4)
        # Input: hidden_dim + input_ch -> hidden_dim
        self.linear_skip = nn.Linear(hidden_dim + self.input_ch, hidden_dim)
        
        # Output heads from the last layer of pos MLP
        # Density head: hidden_dim -> 1
        self.sigma_head = nn.Linear(hidden_dim, 1)
        # Feature head: hidden_dim -> hidden_dim
        self.feature_head = nn.Linear(hidden_dim, hidden_dim)
        
        # Color MLP
        # Input: hidden_dim (feature) + input_ch_views (direction)
        self.color_linear1 = nn.Linear(hidden_dim + self.input_ch_views, hidden_dim // 2)
        self.color_linear2 = nn.Linear(hidden_dim // 2, 3)

    def forward(self, x, d):
        """
        x: (..., 3) spatial coordinates
        d: (..., 3) viewing directions
        """
        # 1. Positional Encoding
        x_enc = positional_encoding(x, self.pos_enc_len)
        d_enc = positional_encoding(d, self.dir_enc_len)
        
        # 2. Position MLP
        h = self.linear1(x_enc)
        h = F.relu(h)
        
        for i, layer in enumerate(self.linears):
            if i == self.skip_layer - 1: # skip_layer is 1-based index 5, so 0-based index 4
                # Concatenate original encoded position
                h = torch.cat([h, x_enc], dim=-1)
                h = self.linear_skip(h)
            else:
                h = layer(h)
            h = F.relu(h)
            
        # 3. Extract Density and Features
        # Density: ReLU of linear output
        sigma_raw = self.sigma_head(h)
        sigma = F.relu(sigma_raw)
        
        # Features: Linear output (no activation specified for features in [S2], 
        # but [S8, unofficial] uses Sigmoid. [S2] implies raw features concatenated with dir.
        # We follow [S2] which says "outputs sigma and a 256-dimensional feature vector"
        # and [S12, unofficial] "feature_head (Linear 256->256) for colour input".
        # [S9, unofficial] uses feature_proj (Linear).
        # We assume no activation on features before concatenation, 
        # but [S8, unofficial] uses Sigmoid. Let's stick to [S2] description: "outputs ... feature vector".
        # Usually features are not clamped.
        features = self.feature_head(h)
        
        # 4. Color MLP
        # Concatenate features and encoded direction
        h_color = torch.cat([features, d_enc], dim=-1)
        
        h_color = self.color_linear1(h_color)
        h_color = F.relu(h_color)
        
        rgb = self.color_linear2(h_color)
        rgb = torch.sigmoid(rgb)
        
        return sigma, rgb
```

### Parameters
*   **Hidden Width ($W$):** 256 for position MLP, 128 for color MLP [S2][S12, unofficial].
*   **Depth:** 8 layers for position MLP, 1 hidden layer for color MLP [S2][S12, unofficial].
*   **Positional Encoding Bands ($L$):** 10 for position, 4-6 for direction [S11, S16, unofficial].
*   **Skip Connection:** Layer index 4 (0-based) [S2][S12, unofficial].
*   **Activations:** ReLU for hidden layers, ReLU (or Softplus) for density, Sigmoid for color [S2][S9, S16, unofficial].

### Edge Cases and Pitfalls
*   **Spectral Bias:** Standard MLPs with ReLU struggle with high-frequency details. Positional encoding is mandatory [S2].
*   **Density Initialization:** Using a simple ReLU on the density head can lead to dense scenes. Shifting the bias by -1 and using Softplus ($\sigma = \text{softplus}(h - 1)$) encourages sparser initial densities [S9, unofficial].
*   **View Independence:** Density must be predicted solely from position $\mathbf{x}$ to ensure multiview consistency. If direction $\mathbf{d}$ is used for density, artifacts appear [S2, S13].
*   **Normalization:** Inputs should be normalized (e.g., to $[-1, 1]$) to match the scale expected by the positional encoding and network weights [S4, S2].

## Subtopics

### [Unbounded inward-facing scene parameterization](957-unbounded-inward-facing-scene-parameterization.md)
Parameterizes unbounded scenes by mapping $\mathbb{R}^3$ to a bounded domain, handling infinite ray sampling.

### [Position-Only NeRF Field](1253-position-only-nerf-field.md)
Omits viewing direction input, assuming isotropic appearance, reducing model size.

### [Tiny NeRF](1275-tiny-nerf.md)
Compresses voxel-based NeRFs via DCT and pruning, operating on frequency coefficients rather than MLP weights.

### [MLPRender_Fea](2414-mlprender-fea.md)
Decouples density and color, using an MLP to predict color from pre-computed features and view direction.

### [View-Dependent MLP](2444-view-dependent-mlp.md)
Expands the view-dependent processing stage to reduce "floaters" and capture complex specularities.

### [NeUDF Network Architecture](3732-neudf-network-architecture.md)
Uses separate MLPs for Unsigned Distance Field (geometry) and radiance (color).

### [InstantNGP Network Architecture](5049-instantngp-network-architecture.md)
Replaces positional encoding with multi-resolution hash grids for acceleration.

### Recursive Neural Field
A sub-network $F_{\Phi_i}$ that takes its parent branch's output $y_{p_i}$ and viewing direction $d$ as inputs, predicting color $c_i$, density $\sigma_i$, uncertainty $\delta_i$, and a latent vector $y_i$ [S22]. The root network $F_{\Phi_1}$ takes the query coordinate $(x, y, z)$ as input [S22]. It consists of an MLP module, a Branch module (predicting uncertainty and routing high-uncertainty points to sub-networks), and an Out module (decoding features) [S22].

### VFR Network Architecture
Uses a Multiresolution Hash Grid (MHG) for feature representation. The Color NN consists of a Spatial MLP (2 layers, 256 hidden neurons) and a Directional MLP (4 layers, 256 hidden neurons) with GELU activation and a 256-dimensional bottleneck feature [S23].

### CT-NeRF Network Architecture
Follows the BARF approach, using a single 8-layer MLP network with a width of 128 [S24, unofficial].

### Soft Zero-Skipping Gate
A component added to the MLP decoder to ensure that a feature compressed to 0 translates to zero density [S25]. It applies a differentiable gate $\hat{g}(\mathbf{v}) = \tanh(\alpha \|\mathbf{v}\|_2)$ to the MLP output $M(\mathbf{v})$, such that $\hat{M}(\mathbf{v}) = \hat{g}(\mathbf{v}) M(\mathbf{v})$ [S25]. The constant $\alpha$ is scheduled from $10^4$ to $10^5$ to allow smooth transitions for pruning [S25].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L \cdot W^2)$ per ray sample, where $N$ is the number of samples, $L$ is the number of layers, and $W$ is the hidden width. The positional encoding adds $O(N \cdot L_{freq})$.
*   **Memory Complexity:** $O(L \cdot W^2)$ for weights. The original NeRF MLP uses ~5 MB for weights [S2].
*   **Performance:** On an NVIDIA V100, rendering one frame (150-200 million queries) takes ~30 seconds [S2].
*   **Strengths:** High-quality novel view synthesis, compact storage, continuous representation.
*   **Limitations:** Slow training and inference due to dense MLP evaluation; spectral bias without positional encoding; requires careful hyperparameter tuning for encoding frequencies [S2].

## Background
The NeRF MLP Architecture was introduced by Mildenhall et al. in the paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S2]. It represents a scene as a continuous 5D function optimized via gradient descent to minimize the photometric error between rendered and observed images [S2, S3]. It builds on the principles of neural fields and addresses the limitations of discrete volumetric representations like Neural Volumes and SRN by offering superior quality and lower storage costs [S2].

## Sources
[S1] Efficient View Synthesis with Neural Radiance Distribution Field, https://doi.org/10.1109/iccv51070.2023.01696, fetched 2026-10-26
[S2] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S3] ScanNeRF: a Scalable Benchmark for Neural Radiance Fields, https://doi.org/10.1109/wacv56688.2023.00088, fetched 2026-10-26
[S4] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
[S5] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S6] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S7] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S8] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S9] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S10] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S11] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S12] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S13] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S14] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S15] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S16] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S17] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S18] sr-bang/NeRF, https://github.com/sr-bang/NeRF, fetched 2026-10-27 (unofficial)
[S19] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
[S20] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
[S21] MELON: NeRF with Unposed Images in SO(3), https://doi.org/10.1109/3dv62453.2024.00084, fetched 2026-10-30
[S22] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S23] PDF Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://papers.neurips.cc/paper_files/paper/2023/file/ce182e31662883d4decc84a0255335b6-Paper-Conference.pdf, fetched 2026-10-27
[S24] CT-NeRF: Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory, https://doi.org/10.48550/arxiv.2404.13896, fetched 2026-10-26 (unofficial)
[S25] HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation, https://doi.org/10.1109/iccv51070.2023.00322, fetched 2026-10-26
