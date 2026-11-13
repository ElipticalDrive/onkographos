# NeRF MLP Architecture

Also called: NeRF MLP, Neural Radiance Field MLP

## Overview
The NeRF MLP is a multi-layer perceptron that maps 3D spatial coordinates and viewing directions to volume density and view-dependent color, serving as the core implicit representation in Neural Radiance Fields [S1, S2, S3]. It differs from its siblings, such as [Scene Representation Networks (SRN)](1150-scene-representation-networks-srn.md) which uses a single network for all outputs, and [SIREN Neural Field](1250-siren-neural-field.md) which uses periodic activations, by employing a two-branch architecture with ReLU activations, skip connections, and positional encoding to enforce multiview consistency (view-independent density, view-dependent color) [S2][S9, S12, unofficial].

## How it works
**Inputs and Outputs:**
The network $F_\Theta$ takes a 5D input vector consisting of a 3D spatial position $\mathbf{x} = (x, y, z)$ and a 3D Cartesian viewing direction $\mathbf{d}$ (derived from spherical coordinates $\theta, \phi$) [S2]. It outputs a scalar volume density $\sigma$ and a 3D RGB color vector $\mathbf{c} = (r, g, b)$ [S2][S11, unofficial].

**Data Structures and Layout:**
The architecture consists of two main components: a position MLP (density backbone) and a color MLP (view-dependent head) [S12, unofficial].
1.  **Positional Encoding:** Inputs are expanded using Fourier features (positional encoding) [S2][S16, unofficial]. For position $\mathbf{x}$, $L_{pos}$ frequency bands are used, resulting in input dimension $D_{pos} = 3 \times (1 + 2L_{pos})$. For direction $\mathbf{d}$, $L_{dir}$ bands are used, resulting in $D_{dir} = 3 \times (1 + 2L_{dir})$ [S12, unofficial]. Default values are $L_{pos}=10$ and $L_{dir}=4$ (or 6 in some implementations) [S11, S16, unofficial].
2.  **Position MLP:** A sequence of $D$ fully connected layers (default $D=8$) with hidden dimension $W$ (default 256) [S2][S12, unofficial].
    *   Layer 0: Linear($D_{pos}, W$).
    *   Layers $i \in \{1, \dots, D-1\}$: Linear($W, W$).
    *   Skip Connection: At layer index $S$ (default $S=4$), the original encoded position $\gamma(\mathbf{x})$ is concatenated with the layer output [S2][S12, unofficial].
    *   Activations: ReLU is applied after every layer except the final output projection [S2][S12, unofficial].
3.  **Output Heads:**
    *   **Density Head:** A Linear($W, 1$) layer followed by a ReLU activation to ensure $\sigma \ge 0$ [S2][S16, unofficial]. Some implementations use Softplus($h - 1$) for sparser initialization [S9, unofficial].
    *   **Feature Projection:** A Linear($W, W$) layer (no activation) produces a 256-dimensional feature vector $\mathbf{e}$ [S2][S12, unofficial].
4.  **Color MLP:**
    *   Input: Concatenation of feature vector $\mathbf{e}$ and encoded direction $\gamma(\mathbf{d})$ [S2].
    *   Hidden Layer: Linear($W + D_{dir}, W/2$) (default 128) with ReLU activation [S2][S12, unofficial].
    *   Output Layer: Linear($W/2, 3$) followed by Sigmoid activation to clamp RGB to $[0, 1]$ [S2][S16, unofficial].

**Numbered Steps:**
1.  Normalize spatial coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ to range $[-1, 1]$ [S4, S2].
2.  Apply positional encoding $\gamma(\mathbf{x})$ and $\gamma(\mathbf{d})$ using the formula:
    $$ \gamma(\mathbf{v}) = [\mathbf{v}, \sin(2^0 \pi \mathbf{v}), \cos(2^0 \pi \mathbf{v}), \dots, \sin(2^{L-1} \pi \mathbf{v}), \cos(2^{L-1} \pi \mathbf{v})] $$
    for each component of $\mathbf{v} \in \{\mathbf{x}, \mathbf{d}\}$ [S3][S16, unofficial].
3.  Pass $\gamma(\mathbf{x})$ through the Position MLP:
    *   Initialize $h_0 = \text{Linear}_0(\gamma(\mathbf{x}))$.
    *   For $i = 1$ to $D-1$:
        *   If $i == S$, $h_i = \text{Linear}_i(\text{ReLU}(\text{Concat}(h_{i-1}, \gamma(\mathbf{x}))))$.
        *   Else, $h_i = \text{Linear}_i(\text{ReLU}(h_{i-1}))$.
4.  Compute density $\sigma = \text{ReLU}(\text{Linear}_{\sigma}(h_{D-1}))$ [S2][S16, unofficial].
5.  Compute feature vector $\mathbf{e} = \text{Linear}_{feat}(h_{D-1})$ [S2].
6.  Concatenate $\mathbf{e}$ and $\gamma(\mathbf{d})$ to form $\mathbf{z} = \text{Concat}(\mathbf{e}, \gamma(\mathbf{d}))$ [S2].
7.  Compute color $\mathbf{c} = \text{Sigmoid}(\text{Linear}_{color}(\text{ReLU}(\text{Linear}_{hidden}(\mathbf{z}))))$ [S2][S16, unofficial].

**Parameters:**
*   $L_{pos}$: Positional encoding depth for position. Default: 10 [S11, S16, unofficial].
*   $L_{dir}$: Positional encoding depth for direction. Default: 4 [S11, unofficial] or 6 [S16, unofficial].
*   $D$: Number of hidden layers in position MLP. Default: 8 [S2][S6, unofficial].
*   $W$: Hidden layer width. Default: 256 [S2][S16, unofficial].
*   $S$: Index of skip connection layer. Default: 4 [S9, S16, unofficial].
*   Optimizer: Adam with learning rate $10^{-4}$, $\beta_1=0.9$, $\beta_2=0.999$ [S21].
*   Batch size: 128 pixels per batch [S21].

**Edge Cases and Pitfalls:**
*   **View Independence:** Density $\sigma$ must be computed solely from position $\mathbf{x}$ to enforce multiview consistency [S2][S9, unofficial].
*   **Initialization:** Using Softplus($h-1$) for density helps maintain sparser initial densities [S9, unofficial].
*   **Training Stability:** Ignoring view dependence (random directions) for the first 50k steps can aid convergence [S21].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class NeRFMLP(nn.Module):
    def __init__(self, L_pos=10, L_dir=4, D=8, W=256, skip_layer=4):
        super().__init__()
        self.L_pos = L_pos
        self.L_dir = L_dir
        self.D = D
        self.W = W
        self.skip_layer = skip_layer
        
        # Input dimensions after positional encoding
        # 3 coords * (1 + 2*L) frequencies
        self.input_ch = 3 * (1 + 2 * L_pos)
        self.input_ch_views = 3 * (1 + 2 * L_dir)
        
        # Position MLP layers
        # Layer 0: input_ch -> W
        self.pts_linears = nn.ModuleList([nn.Linear(self.input_ch, W)])
        # Layers 1 to D-1
        for i in range(1, D):
            if i == skip_layer:
                # Skip connection: concat original input
                self.pts_linears.append(nn.Linear(W + self.input_ch, W))
            else:
                self.pts_linears.append(nn.Linear(W, W))
                
        # Output heads
        self.sigma_head = nn.Linear(W, 1)
        self.feature_proj = nn.Linear(W, W)
        
        # Color MLP
        # Input: W (features) + input_ch_views (direction)
        self.color_hidden = nn.Linear(W + self.input_ch_views, W // 2)
        self.color_head = nn.Linear(W // 2, 3)

    def forward(self, x, d):
        # x: (N, 3), d: (N, 3)
        
        # 1. Positional Encoding
        x_emb = positional_encoding(x, self.L_pos)
        d_emb = positional_encoding(d, self.L_dir)
        
        # 2. Position MLP
        h = x_emb
        for i, layer in enumerate(self.pts_linears):
            if i == self.skip_layer:
                h = torch.cat([h, x_emb], dim=-1)
            h = F.relu(layer(h))
            
        # 3. Density (view-independent)
        # Using ReLU as per standard NeRF [S2][S16, unofficial]
        sigma = F.relu(self.sigma_head(h))
        
        # 4. Feature Projection
        feat = self.feature_proj(h)
        
        # 5. Color (view-dependent)
        feat_dir = torch.cat([feat, d_emb], dim=-1)
        rgb = torch.sigmoid(self.color_head(F.relu(self.color_hidden(feat_dir))))
        
        return rgb, sigma

def positional_encoding(x, L):
    """
    x: (N, 3)
    Returns: (N, 3 * (1 + 2*L))
    """
    # Original coordinates
    out = [x]
    # Sin/Cos frequencies
    for freq in range(L):
        scale = 2.0 ** freq * torch.pi
        sin_val = torch.sin(scale * x)
        cos_val = torch.cos(scale * x)
        out.append(sin_val)
        out.append(cos_val)
    return torch.cat(out, dim=-1)
```

## Subtopics

### [Unbounded inward-facing scene parameterization](957-unbounded-inward-facing-scene-parameterization.md)
Parameterizes unbounded scenes by warping infinite coordinates into a bounded unit sphere, enabling efficient voxel grid allocation and watertight mesh extraction.

### [Position-Only NeRF Field](1253-position-only-nerf-field.md)
Omits viewing direction input to reduce model size and training time, assuming isotropic appearance suitable for uniform lighting.

### [Tiny NeRF](1275-tiny-nerf.md)
Compresses voxel-based NeRFs via Discrete Cosine Transformation (DCT) on voxel grids, pruning and quantizing coefficients for >100x storage reduction.

### [MLPRender_Fea](2414-mlprender-fea.md)
Decouples density prediction from color prediction, using an MLP to predict RGB from pre-computed geometric features and view directions, but not position.

### [View-Dependent MLP](2444-view-dependent-mlp.md)
Replaces the small bottleneck in InstantNGP with a larger, deeper architecture to reduce "floaters" by better modeling view-dependent specularities.

### [NeUDF Network Architecture](3732-neudf-network-architecture.md)
Uses two distinct MLPs to represent geometry as an Unsigned Distance Field (UDF) and color separately, allowing explicit surface extraction.

### [InstantNGP Network Architecture](5049-instantngp-network-architecture.md)
Accelerates rendering by replacing sinusoidal positional encoding with a sparse, multi-resolution hash grid feature storage mechanism.

### Recursive Neural Field
A sub-network $F_{\Phi_i}$ that takes its parent branch's output $y_{p_i}$ and viewing direction $d$ as inputs, predicting color $c_i$, density $\sigma_i$, uncertainty $\delta_i$, and a latent vector $y_i$ [S22]. It consists of an MLP module, a Branch module (predicting uncertainty and routing high-uncertainty points to sub-networks), and an Out module [S22]. The root network $F_{\Phi_1}$ takes query coordinates $(x, y, z)$ as input [S22].

### VFR Network Architecture
Uses a Multiresolution Hash Grid (MHG) for feature representation. The Color NN comprises a Spatial MLP (2 layers, 256 hidden neurons) and a Directional MLP (4 layers, 256 hidden neurons) with GELU activation and a 256-dimensional bottleneck [S23].

### CT-NeRF Network Architecture
Follows the BARF approach with a single 8-layer MLP network with a width of 128 [S24, unofficial].

### Soft Zero-Skipping Gate
A component added to the MLP to ensure that a feature compressed to 0 translates to zero density, preventing non-zero densities from sparse zero features [S25]. It applies a differentiable gate $\hat{g}(\mathbf{v}) = \tanh(\alpha \|\mathbf{v}\|_2)$ to the MLP output $M(\mathbf{v})$, such that $\hat{M}(\mathbf{v}) = \hat{g}(\mathbf{v}) M(\mathbf{v})$ [S25]. The parameter $\alpha$ is scheduled from $10^4$ to $10^5$ to allow smooth transitions and non-zero gradients for effective pruning via ADMM, avoiding the zero-gradient issues of hard gates [S25].

### DoubleField Network
A joint implicit function $f_{db}$ represented by MLPs that fits both a surface field and a radiance field simultaneously, conditioned on pixel-aligned image features $\phi(\mathbf{x}, I)$ [S26]. It uses a shared MLP $E_{db}$ to produce a double embedding $e_{db}$ from positional encoding $\gamma(\mathbf{x})$ and image features, which is then split into a Geometry MLP $E_g$ (predicting surface field $s$ and density $\sigma$) and a Texture MLP $E_c$ (predicting color $c$ using viewing direction $d$) [S26].

## Operating characteristics
*   **Time Complexity:** Training is computationally expensive, requiring hundreds of millions of network queries per image. On an NVIDIA V100, rendering one frame takes approximately 30 seconds [S2].
*   **Memory Complexity:** The network weights are small (approx. 5 MB), offering a 3000x compression ratio compared to voxel-based methods like LLFF [S2]. Memory usage is constant with respect to spatial resolution [S1].
*   **Performance:** Achieves state-of-the-art view synthesis quality on complex scenes, outperforming SRN and LLFF in PSNR, SSIM, and LPIPS metrics [S2].
*   **Strengths:** Continuous representation allows infinite resolution; compact storage; handles complex geometry and view-dependent effects (specularities) [S2].
*   **Limitations:** Slow training and rendering times; requires significant GPU memory for large scenes or high-resolution training; sensitive to positional encoding frequency choices [S2][S16, unofficial].
*   **When to use:** High-quality novel view synthesis where storage is constrained and training time is acceptable.
*   **When not to use:** Real-time applications or scenarios requiring extremely fast training/inference without acceleration techniques (like InstantNGP) [S2].

## Background
The NeRF MLP architecture was introduced by Mildenhall et al. in the paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (ECCV 2020) [S1, S2]. It represents a scene as a continuous 5D function mapping spatial location and viewing direction to volume density and emitted radiance, optimized using differentiable volume rendering [S2]. The method builds on the principles of implicit neural representations and positional encoding to overcome the low-frequency bias of standard MLPs [S2].

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
[S26] DoubleField: Bridging the Neural Surface and Radiance Fields for High-fidelity Human Reconstruction and Rendering, https://doi.org/10.1109/cvpr52688.2022.01541, fetched 2026-10-29
