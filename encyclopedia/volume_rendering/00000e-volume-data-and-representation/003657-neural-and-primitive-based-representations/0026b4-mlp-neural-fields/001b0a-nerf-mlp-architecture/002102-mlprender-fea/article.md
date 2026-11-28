# MLPRender_Fea

Also called: MLP Render with Features, MLP Render with Feature Encoding

## Overview
MLPRender_Fea is a neural rendering module that predicts RGB color from input features and view directions using a Multi-Layer Perceptron (MLP) [S3, S8, unofficial]. It solves the problem of mapping continuous spatial features to view-dependent radiance values in neural implicit representations [S3, S8, unofficial]. It sets itself apart from its parent [NeRF MLP Architecture](910-nerf-mlp-architecture.md) by accepting pre-computed features as primary input instead of raw 3D coordinates, while still processing view directions to model appearance variations [S3, S8, unofficial]. Unlike [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which injects global scene context, MLPRender_Fea focuses on local feature-to-color mapping [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `features`: Input feature tensor of shape `(batch, inChanel)` [S3, S8, unofficial].
    *   `viewdirs`: View direction tensor of shape `(batch, 3)` [S3, S8, unofficial].
    *   `viewpe`: Integer specifying the number of sinusoidal frequencies for view direction encoding [S3, S8, unofficial].
    *   `feape`: Integer specifying the number of sinusoidal frequencies for feature encoding [S3, S8, unofficial].
*   **Outputs:**
    *   `rgb`: Predicted RGB color tensor of shape `(batch, 3)` with values in $[0, 1]$ [S3, S8, unofficial].

**Data Structures:**
*   **MLP Architecture:** A sequential stack of 3 Linear layers [S3, S8, unofficial].
    *   Layer 1: `Linear(in_mlpC, featureC)`
    *   Layer 2: `Linear(featureC, featureC)`
    *   Layer 3: `Linear(featureC, 3)`
*   **Parameters:**
    *   `inChanel`: Input feature channel size [S3, S8, unofficial].
    *   `featureC`: Hidden dimension size, default 128 [S3, S8, unofficial].
    *   `viewpe`: Default 6 [S3, S8, unofficial].
    *   `feape`: Default 0 [S3, S8, unofficial].

**Formulas:**
The input dimension to the MLP is calculated as [S3, S8, unofficial]:
$$ in\_mlpC = inChanel + 3 + 2 \cdot viewpe \cdot 3 $$
Input concatenation order [S3, S8, unofficial]:
$$ mlp\_in = \text{concat}([features, viewdirs, \text{PE}(features, feape), \text{PE}(viewdirs, viewpe)]) $$
Where $\text{PE}(x, L)$ is the positional encoding [S4, unofficial]:
$$ \text{out} = [x, \sin(2^0 x), \cos(2^0 x), \dots, \sin(2^{L-1} x), \cos(2^{L-1} x)] $$

**Steps:**
1.  Initialize `indata` list with `[features, viewdirs]` [S3, unofficial].
2.  If `feape > 0`, append `positional_encoding(features, feape)` to `indata` [S3, unofficial].
3.  If `viewpe > 0`, append `positional_encoding(viewdirs, viewpe)` to `indata` [S3, unofficial].
4.  Concatenate all tensors in `indata` along the last dimension to form `mlp_in` [S3, unofficial].
5.  Pass `mlp_in` through Layer 1, apply ReLU [S3, S8, unofficial].
6.  Pass result through Layer 2, apply ReLU [S3, S8, unofficial].
7.  Pass result through Layer 3 [S3, S8, unofficial].
8.  Apply Sigmoid activation to the output to constrain RGB values to $[0, 1]$ [S3, S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class PositionalEncoding(nn.Module):
    def __init__(self, embedding_size):
        super().__init__()
        self.embedding_size = embedding_size

    def forward(self, x):
        out = [x]
        for i in range(self.embedding_size):
            for fn in [torch.sin, torch.cos]:
                out.append(fn(2 ** i * x))
        return torch.cat(out, dim=-1)

class MLPRender_Fea(nn.Module):
    def __init__(self, inChanel, featureC=128, viewpe=6, feape=0):
        super().__init__()
        self.inChanel = inChanel
        self.viewpe = viewpe
        self.feape = feape
        self.featureC = featureC
        
        self.in_mlpC = inChanel + 3 + 2 * viewpe * 3
        
        self.pos_enc_view = PositionalEncoding(viewpe)
        self.pos_enc_feat = PositionalEncoding(feape) if feape > 0 else None
        
        self.mlp = nn.Sequential(
            nn.Linear(self.in_mlpC, self.featureC),
            nn.ReLU(),
            nn.Linear(self.featureC, self.featureC),
            nn.ReLU(),
            nn.Linear(self.featureC, 3)
        )
        
        torch.nn.init.constant_(self.mlp[-1].bias, 0)

    def forward(self, features, viewdirs):
        indata = [features, viewdirs]
        
        if self.feape > 0:
            indata += [self.pos_enc_feat(features)]
            
        if self.viewpe > 0:
            indata += [self.pos_enc_view(viewdirs)]
            
        mlp_in = torch.cat(indata, dim=-1)
        rgb = self.mlp(mlp_in)
        rgb = torch.sigmoid(rgb)
        return rgb
```

**Edge Cases and Pitfalls:**
*   **Feature Dimensionality:** Positional encoding is typically defined for 3D coordinates. If `features` have a different dimensionality, the PE application must be adapted or `feape` should be 0 [S3, S8, unofficial].
*   **Bias Initialization:** The final layer bias must be initialized to 0 to ensure the network starts with neutral color predictions [S3, S8, unofficial].

## Subtopics

### [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md)
Extends standard NeRF by learning per-image embedding vectors injected into the color network to model global appearance shifts. Sets itself apart by explicitly modeling global appearance variations across input views. Choose when scenes exhibit significant photometric changes across views.

### [Volume Radiance Network](2576-volume-radiance-network.md)
Predicts RGB from features, view directions, and positions using MLP with PE on features and view direction, but not position. Sets itself apart by applying positional encoding to features and view directions. Choose for hybrid rendering pipelines requiring feature-based color prediction.

### [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md)
Replaces the nonlinear MLP decoder with a linear decoder using spatially-localized features as coefficients for a learned basis. Sets itself apart by explicitly modeling view dependence through a learned basis rather than a black-box MLP. Choose for scenes with variable appearance requiring adaptivity.

### [Hybrid View-dependent Color Modeling](5083-hybrid-view-dependent-color-modeling.md)
Combines sparse feature voxel grid with shallow MLP, decoupling view-independent diffuse color from view-dependent shading. Sets itself apart by addressing the trade-off between slow convergence of pure neural fields and limited quality of grid-only representations. Choose for faster training and real-time rendering.

### [Color Network Architecture](7200-color-network-architecture.md)
Defines the standard MLP structure (linear layers with ReLU) for radiance field prediction, often with specific weight scaling. Sets itself apart by explicitly defining the standard MLP structure. Choose when implementing standard NeRF color prediction heads.

### [Neural BRDF and Geometry Decoding](8139-neural-brdf-and-geometry-decoding.md)
Maps features to BRDF parameters, normals, and RGB using distinct MLPs for disentangling geometry and appearance. Sets itself apart by focusing on per-point material decoding rather than global image-level biases. Choose for accurate relighting and normal estimation.

### [MLPRender_Fea_WeakView](10541-mlprender-fea-weakview.md)
Uses Gaussian positional encoding on features and view directions, but not positions. Sets itself apart by explicitly processing view directions alongside features through a dedicated encoding and concatenation step. Choose when weak view dependence is desired.

### [Gaussian Activated Radiance Fields Render](10546-gaussian-activated-radiance-fields-render.md)
Applies Gaussian kernels to encoded inputs before the MLP. Sets itself apart by applying Gaussian kernels to the encoded inputs. Choose for architectures using Gaussian activations.

### [Neural Texture Decoder](6704-neural-texture-decoder.md)
Decoder for texture generation. Sets itself apart by operating on spatial grids to preserve local geometric coherence. Choose for generating high-resolution texture maps for UV unwrapping or style transfer.

### [MLPRender_Fea_WeakView](10541-mlprender-fea-weakview.md)
Input includes encoded position, spherical harmonics encoding of view direction, normal vector, and geometric feature vector. MLP structure uses GELU activations. Sets itself apart by handling volumetric surfaces with layered meshes. Choose for fuzzy geometry representation.

### [MLPRender_Fea_WeakView](10541-mlprender-fea-weakview.md)
Fusion ablation technique where opacity is regressed from an MLP using 8-dim feature vectors from HexPlane. Sets itself apart by boosting PSNR, SSIM, LPIPS at the cost of slower rendering. Choose when accuracy is prioritized over speed.

### [MLPRender_Fea_WeakView](10541-mlprender-fea-weakview.md)
Output feature computed as $F_{output} = F_{input} + \phi(W_f \cdot F_{input}) \odot \sigma(W_g \cdot F_{input})$. Sets itself apart by dynamically selecting features for each channel and spatial location. Choose for filtering invalid features in driving scenes.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of MLP layers and input dimension. The MLP is shallow (3 layers), making it fast per sample [S3, S8, unofficial].
*   **Memory Complexity:** Proportional to the number of parameters in the MLP ($O(featureC \cdot in\_mlpC)$) [S3, S8, unofficial].
*   **Performance:** Efficient for scenes with pre-computed features (e.g., from triplanes or voxel grids) [S6].
*   **Strengths:** Decouples geometry/feature extraction from color prediction, allowing reuse of feature encoders [S3, S8, unofficial].
*   **Limitations:** Relies on the quality of input features; if features are poor, color prediction will be limited [S3, S8, unofficial].

## Background
Originated from the Neural Radiance Fields (NeRF) framework [S2, S4, unofficial]. The "MLPRender_Fea" variant specifically adapts the NeRF color prediction head to accept features instead of raw coordinates, enabling integration with other feature-based representations like Triplanes [S6] or learned embeddings [S1]. It is widely used in hybrid neural rendering methods [S3, S8, unofficial].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S3] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S4] AlbertAffectus/NeRF, https://github.com/AlbertAffectus/NeRF, fetched 2026-10-27 (unofficial)
[S5] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S6] TriDF: Triplane-Accelerated Density Fields for Few-Shot Remote Sensing Novel View Synthesis, https://doi.org/10.1109/tgrs.2025.3643432, fetched 2026-10-30
[S7] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29
[S8] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08 (unofficial)
[S9] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26
[S10] Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes, https://doi.org/10.1109/cvpr52734.2025.01991, fetched 2026-10-26
[S11] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
[S12] DGNR: Density-Guided Neural Point Rendering of Large Driving Scenes, https://doi.org/10.1109/tase.2024.3410891, fetched 2026-10-27
