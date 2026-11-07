# MLPRender_Fea

Also called: Feature Encoding and MLP Decoding, MLP Render, MLP Render with Feature Encoding, MLP Render with Features, Scene Reconstruction MLP

## Overview
MLPRender_Fea is a neural rendering module that predicts RGB color from input features, view directions, and their positional encodings using a Multi-Layer Perceptron (MLP) [S3, unofficial]. It builds on [NeRF MLP Architecture](910-nerf-mlp-architecture.md) by decoupling the spatial feature representation from the view-dependent color prediction, allowing the MLP to focus solely on mapping features and viewing angles to radiance [S1][S3, unofficial]. Unlike the parent architecture which maps raw 5D coordinates directly to density and color, MLPRender_Fea takes pre-computed features as input, enabling hybrid approaches where features are stored in grids or learned per-image [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `features`: A tensor of spatial features. The dimension is `inChanel` (or `input_feature_dim`) [S3, S4, unofficial].
    *   `viewdirs`: A tensor of viewing directions, shape `(batch, 3)` [S2, S3, unofficial].
    *   `position`: (Optional in some variants, but typically excluded from the MLP input in the core definition cited in [S3, unofficial]). The core definition states position is NOT included in the input [S3, unofficial].
*   **Outputs:**
    *   `rgb`: Predicted RGB color values, shape `(batch, 3)`, range `[0, 1]` [S2, S3, unofficial].
    *   `sigma`: (Optional, depending on whether this module is used for density or just color. In the context of [S3, unofficial], it predicts color. In [S2, unofficial], the full NeRF predicts sigma. MLPRender_Fea specifically refers to the color prediction part in [S3, unofficial] and [S1]. If used for density, it is often handled by a separate branch or the feature encoder itself. The notes for [S3, unofficial] indicate it predicts RGB. The notes for [S2, unofficial] describe the full NeRF. We focus on the color prediction aspect as "MLPRender_Fea" implies the rendering of features.)

**Data Structures:**
*   **MLP Layers:** Sequential linear layers with ReLU activations, ending with a Sigmoid activation for RGB output [S3, unofficial].
*   **Positional Encoding:** Sinusoidal encoding applied to `viewdirs` and optionally `features` [S3, unofficial].

**Parameters:**
*   `viewpe`: Number of sinusoidal encoding functions for view directions. Default: Not explicitly given in [S3, unofficial] code snippet, but implied by `direction_embedding_size` in [S2, unofficial] (default 4) or calculated in [S3, unofficial] formula. [S3, unofficial] formula uses `viewpe`.
*   `feape`: Number of sinusoidal encoding functions for features. Default: 0 (not applied in [S3, unofficial] variant 2) or specified in [S3, unofficial] variant 1.
*   `featureC`: Hidden dimension size of the MLP. Default: 128 [S3, unofficial].
*   `inChanel`: Input feature dimension. Default: Not given in sources, varies by implementation (e.g., 3 in [S3, unofficial] variant 1, 12 in [S6]).

**Steps:**
1.  **Positional Encoding:**
    *   If `viewpe > 0`, apply positional encoding to `viewdirs`.
    *   If `feape > 0`, apply positional encoding to `features`.
    *   Positional encoding function $\text{PE}(x, L)$ for input $x$ and embedding size $L$:
        $$ \text{out} = [x, \sin(2^0 x), \cos(2^0 x), \dots, \sin(2^{L-1} x), \cos(2^{L-1} x)] $$
        Implemented as:
        ```python
        def positional_encoding(x, embedding_size):
            out = [x]
            for i in range(embedding_size):
                for fn in [torch.sin, torch.cos]:
                    out.append(fn(2 ** i * x))
            return torch.cat(out, dim=-1)
        ```
        [S2, S3, unofficial]

2.  **Input Construction:**
    *   Concatenate the raw and encoded inputs along the last dimension.
    *   The order is: `[features, viewdirs, positional_encoding(features, feape), positional_encoding(viewdirs, viewpe)]` [S3, unofficial].
    *   Note: Position coordinates are **not** included in the input to this specific MLP [S3, unofficial].
    *   Input dimension $D_{in}$:
        $$ D_{in} = inChanel + 3 + 2 \cdot feape \cdot inChanel + 2 \cdot viewpe \cdot 3 $$
        (Assuming features are encoded element-wise if `feape > 0`, or if features are 3D, $2 \cdot feape \cdot 3$. The source [S3, unofficial] formula is ambiguous: $in\_mlpC = (3+2\cdot viewpe\cdot 3) + (3+2\cdot feape\cdot 3) + inChanel$. This suggests features might be treated as 3D for PE or there is a redundancy. We follow the concatenation order: raw features, raw viewdirs, PE features, PE viewdirs. If `feape=0`, PE features term is 0. If `viewpe=0`, PE viewdirs term is 0.) [S3, unofficial]

3.  **MLP Forward Pass:**
    *   Pass the concatenated input tensor through the MLP.
    *   Architecture (from [S3, unofficial]):
        *   Layer 1: Linear($D_{in}$, `featureC`) -> ReLU
        *   Layer 2: Linear(`featureC`, `featureC`) -> ReLU
        *   Layer 3: Linear(`featureC`, 3) -> Sigmoid
    *   Bias of the final layer is initialized to 0 [S3, unofficial].

4.  **Output:**
    *   Return the output of Layer 3 as RGB color [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Feature Dimension Mismatch:** The input feature dimension must match `inChanel`. If features are from a different source (e.g., voxel grid, per-image embedding), they must be projected or resized to match [S1][S3, unofficial].
*   **View Direction Normalization:** View directions should be normalized to unit length before encoding [S4, unofficial][S6].
*   **Color Range:** The Sigmoid activation ensures output is in $[0, 1]$. If HDR rendering is needed, this activation should be removed or changed [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class MLPRenderFea(nn.Module):
    def __init__(self, inChanel, featureC=128, viewpe=4, feape=0):
        super().__init__()
        self.inChanel = inChanel
        self.featureC = featureC
        self.viewpe = viewpe
        self.feape = feape
        
        # Calculate input dimension
        # Raw viewdirs: 3
        # PE viewdirs: 2 * viewpe * 3
        # Raw features: inChanel
        # PE features: 2 * feape * inChanel (assuming element-wise PE)
        # Note: Source [S3, unofficial] formula is ambiguous regarding feature PE dimension. 
        # We assume PE is applied to the feature vector itself if feape > 0.
        # If features are 3D, it would be 2 * feape * 3. 
        # Given the code snippet in [S3, unofficial] uses `positional_encoding(features, self.feape)`, 
        # and features are likely multi-dimensional, we apply PE to the feature dim.
        # However, standard PE in NeRF is for 3D coords. 
        # Let's stick to the concatenation order and dimensions implied by [S3, unofficial].
        
        # If feape > 0, we assume PE is applied to features. 
        # If features are not 3D, the PE function might need adaptation.
        # For simplicity, we assume the PE function handles the last dimension.
        
        self.in_mlpC = 3 + 2 * viewpe * 3 + inChanel + 2 * feape * inChanel
        
        self.mlp = nn.Sequential(
            nn.Linear(self.in_mlpC, featureC),
            nn.ReLU(),
            nn.Linear(featureC, featureC),
            nn.ReLU(),
            nn.Linear(featureC, 3),
            nn.Sigmoid()
        )
        
        # Initialize bias of last layer to 0
        nn.init.zeros_(self.mlp[-1].bias)

    def positional_encoding(self, x, embedding_size):
        out = [x]
        for i in range(embedding_size):
            for fn in [torch.sin, torch.cos]:
                out.append(fn(2 ** i * x))
        return torch.cat(out, dim=-1)

    def forward(self, features, viewdirs):
        # features: (batch, inChanel)
        # viewdirs: (batch, 3)
        
        # Encode viewdirs
        if self.viewpe > 0:
            encoded_viewdirs = self.positional_encoding(viewdirs, self.viewpe)
        else:
            encoded_viewdirs = viewdirs
            
        # Encode features
        if self.feape > 0:
            encoded_features = self.positional_encoding(features, self.feape)
        else:
            encoded_features = features
            
        # Concatenate: [features, viewdirs, PE_features, PE_viewdirs]
        # Note: The order in [S3, unofficial] is [features, viewdirs, PE_features, PE_viewdirs]
        # But the dimension calculation in [S3, unofficial] suggests:
        # in_mlpC = (3+2*viewpe*3) + (3+2*feape*3) + inChanel
        # This implies features are treated as 3D for PE? Or inChanel=3?
        # Let's follow the explicit concatenation order from [S3, unofficial] code comment:
        # [features, viewdirs, positional_encoding(features, feape), positional_encoding(viewdirs, viewpe)]
        
        mlp_in = torch.cat([features, viewdirs, encoded_features, encoded_viewdirs], dim=-1)
        
        rgb = self.mlp(mlp_in)
        return rgb
```

## Subtopics

### [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md)
A model variant that learns a per-image embedding for each scene and injects it into the NeRF MLP during color prediction. Unlike Stable View Synthesis (SVS), which modifies scene brightness and color to explain camera variation, this method neutralizes photometric variation during training, yielding consistent global brightness and color across views. It builds on [MLPRender_Fea](2414-mlprender-fea.md).

### [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md)
This method replaces the nonlinear Multi-Layer Perceptron (MLP) decoder used in hybrid neural rendering models with a white-box linear decoder that uses spatially-localized features as coefficients for a learned basis to describe view-dependent color and density. It sets itself apart from its parent [MLPRender_Fea](2414-mlprender-fea.md) by explicitly modeling view dependence through a learned basis rather than a black-box MLP. This approach enables adaptivity for scenes with variable appearance and avoids the limited expressivity of spherical harmonic decoders.

### [Hybrid View-dependent Color Modeling](5083-hybrid-view-dependent-color-modeling.md)
This method models view-dependent color emissions by combining a sparse feature voxel grid with a shallow Multi-Layer Perceptron (MLP). It solves the trade-off between the slow convergence of pure neural fields and the limited expressivity of static voxel grids. Unlike its sibling [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which uses global image vectors, or [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md) which uses linear projections, this hybrid approach uses a local spatial lookup followed by a non-linear shallow MLP to capture complex specularities and shading.

### [Neural BRDF and Geometry Decoding](8139-neural-brdf-and-geometry-decoding.md)
This method decodes appearance features $a(x)$ into Bidirectional Reflectance Distribution Function (BRDF) values, surface normals, and RGB colors using distinct neural networks. It builds on [MLPRender_Fea](2414-mlprender-fea.md) by separating the decoding of geometric properties (normals, BRDF) from photometric properties (color) into specialized MLPs, allowing for explicit physical parameterization. Unlike [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which adjusts global appearance, or [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md) which uses linear projections, this approach employs non-linear MLPs to map features to physical quantities.

### Volume Radiance Network
A class `VolumeRadiance` that encapsulates the feature encoding and MLP decoding process [S4, unofficial].
*   **Layout:** Attributes include `n_dir_dims` (default 3), `n_output_dims` (3), `encoding` (from `get_encoding`), `n_input_dims` (sum of `input_feature_dim` and encoding output dims), and `network` (from `get_mlp`) [S4, unofficial].
*   **Steps:**
    1.  Normalize directions from `(-1, 1)` to `(0, 1)` by `(dirs + 1.) / 2.` [S4, unofficial].
    2.  Encode directions using `self.encoding` after viewing as `(-1, n_dir_dims)` [S4, unofficial].
    3.  Concatenate flattened features, encoded directions, and any additional args along the last dimension to form `network_inp` [S4, unofficial].
    4.  Pass `network_inp` through `self.network` and reshape to match input feature shape plus `n_output_dims` [S4, unofficial].
    5.  If `color_activation` is in config, apply the activation function to the color [S4, unofficial].
*   **Parameters:** `input_feature_dim`, `dir_encoding_config`, `mlp_network_config`, `color_activation` [S4, unofficial].

### Neural Texture Decoder
A decoder for texture generation using 3D convolutions [S5, unofficial].
*   **Layout:** Input FC: Fully-connected layer mapping 199-dim vector to (32*32*32*4)-dim vector. Reshape: Reshape to tensor of size 32x32x32x4. 3D Convs: Series of 3D convolutions with channels {4, 8, 4}, kernel sizes {4, 4, 4}, strides {1, 2, 1}. Output: Tensor of size 64x64x64x4 [S5, unofficial].

### Color Network Architecture
A specific MLP architecture for color prediction [S6].
*   **Layout:** Three layers of fully connected layers with width 128. Input dimensions: position (3), view direction (2), feature (12). Output: color (3). Activation: ReLU between layers, sigmoid for color output [S6].
*   **Code:** Input: [position (3), view_dir (2), feature (12)] -> FC(128) -> ReLU -> FC(128) -> ReLU -> FC(128) -> ReLU -> Output: [color (3)] with sigmoid [S6].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of MLP layers and the input dimension (features + encoded viewdirs). The positional encoding is also linear with respect to the embedding size.
*   **Memory Complexity:** Proportional to the number of MLP weights, which is $O(D_{in} \cdot featureC + featureC^2 + featureC \cdot 3)$.
*   **Performance:** The MLP decoder is fast to evaluate but can be a bottleneck if the feature lookup is slow. It is less expressive than complex BRDF models but sufficient for many diffuse and mildly specular scenes [S1][S3, unofficial].
*   **Strengths:** Simple to implement, differentiable, compatible with various feature representations (voxels, embeddings, grids) [S1][S3, unofficial].
*   **Limitations:** Limited expressivity for complex view-dependent effects (e.g., sharp specularities) compared to dedicated BRDF models [S1][S3, unofficial]. Performance depends heavily on the quality of the input features [S1].

## Background
MLPRender_Fea is a component of the Neural Radiance Fields (NeRF) architecture, specifically the part that handles view-dependent color prediction from features [S1][S2, S3, unofficial]. It was introduced in the original NeRF paper [S2, unofficial] and has been widely adopted in subsequent neural rendering works [S1][S3, S4, unofficial][S6]. The method allows for the decoupling of spatial representation (features) from appearance (color), enabling hybrid approaches that combine neural fields with traditional graphics primitives [S1][S3, unofficial].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] AlbertAffectus/NeRF, https://github.com/AlbertAffectus/NeRF, fetched 2026-10-27 (unofficial)
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S4] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S5] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S6] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
