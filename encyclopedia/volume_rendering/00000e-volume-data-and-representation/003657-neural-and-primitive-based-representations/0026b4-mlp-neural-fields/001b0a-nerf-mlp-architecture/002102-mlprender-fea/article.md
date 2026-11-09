# MLPRender_Fea

Also called: MLP Render, MLP Render with Features, MLP Render with Feature Encoding, MLP Render with Feature and View Positional Encoding

## Overview
MLPRender_Fea is a neural rendering module that predicts RGB color from input features, view directions, and positions using a Multi-Layer Perceptron (MLP) with positional encoding applied to features and view direction, but NOT position [S5, unofficial]. It sets itself apart from standard NeRF MLPs by decoupling the density prediction (handled by the parent architecture or a separate branch) from the view-dependent color prediction, allowing the color network to focus exclusively on appearance synthesis given pre-computed geometric features [S5, unofficial]. Unlike [Position-Only NeRF Field](1253-position-only-nerf-field.md) which ignores view direction, or [View-Dependent MLP](2444-view-dependent-mlp.md) which encodes position, MLPRender_Fea explicitly encodes the viewing direction and input features to model view-dependent effects like specularities and lighting changes [S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `features`: A tensor of latent geometric/appearance features associated with the 3D point. Shape: $(N, \text{inChanel})$.
    *   `viewdirs`: A tensor of viewing directions for each sample. Shape: $(N, 3)$.
    *   `viewpe`: Integer, number of sinusoidal frequencies for view direction encoding. Default: 6 [S3, unofficial].
    *   `feape`: Integer, number of sinusoidal frequencies for feature encoding. Default: 0 or positive integer [S3, unofficial].
*   **Outputs:**
    *   `rgb`: A tensor of predicted RGB color values. Shape: $(N, 3)$. Values are in $[0, 1]$ due to sigmoid activation.

**Data Structures:**
The MLP consists of 3 sequential linear layers.
*   `layer1`: Linear transformation from input dimension `in_mlpC` to `featureC`.
*   `layer2`: Linear transformation from `featureC` to `featureC`.
*   `layer3`: Linear transformation from `featureC` to 3 (RGB).
*   `featureC`: Hidden dimension size. Default: 128 [S5, unofficial].
*   `in_mlpC`: Input dimension calculated as:
    $$ in\_mlpC = (3 + 2 \cdot viewpe \cdot 3) + (3 + 2 \cdot feape \cdot 3) + \text{inChanel} $$
    *Note: The formula in [S5, unofficial] suggests features are treated as 3D for PE expansion if `feape > 0`, or the code assumes a specific feature dimensionality. The term `(3+2*feape*3)` implies the positional encoding is applied to a 3D representation of the features, or the features themselves are 3D. However, `inChanel` is added separately. If `feape=0`, the term is just `inChanel`. If `viewpe=0`, the view term is just 3.*
    *   Raw viewdirs: 3 dimensions.
    *   Encoded viewdirs: $2 \cdot viewpe \cdot 3$ dimensions.
    *   Raw features: `inChanel` dimensions.
    *   Encoded features: If `feape > 0`, $2 \cdot feape \cdot 3$ dimensions are added (assuming 3D feature encoding) or potentially `2 * feape * inChanel` if encoding each channel (see [S5, unofficial] layout note which says `2*feape*inChanel`). The definition note [S5, unofficial] says `(3+2*feape*3)`, but the layout note says `2*feape*inChanel`. We follow the layout note for implementation as it is more general for arbitrary `inChanel`.
    *   Corrected Input Dimension:
        $$ in\_mlpC = 3 + 2 \cdot viewpe \cdot 3 + \text{inChanel} + (2 \cdot feape \cdot \text{inChanel} \text{ if } feape > 0 \text{ else } 0) $$
        *Wait, [S5, unofficial] layout explicitly states: `in_mlpC = 2*viewpe*3 + 2*feape*inChanel + 3 + inChanel`. This confirms the feature PE expands the `inChanel` dimension.*

**Parameters:**
*   `viewpe`: Number of view positional encoding frequencies. Default: 6 [S3, unofficial].
*   `feape`: Number of feature positional encoding frequencies. Default: 0 [S3, unofficial].
*   `featureC`: Hidden layer channel size. Default: 128 [S5, unofficial].

**Step-by-Step Method:**

1.  **Positional Encoding of View Directions:**
    If `viewpe > 0`, apply sinusoidal positional encoding to `viewdirs`.
    $$ \text{encoded\_viewdirs} = \text{PE}(\text{viewdirs}, \text{viewpe}) $$
    Where $\text{PE}(x, L)$ produces a tensor of shape $(N, 3 + 2 \cdot L \cdot 3)$ containing $x$ and $\sin(2^i x), \cos(2^i x)$ for $i \in [0, L-1]$.

2.  **Positional Encoding of Features:**
    If `feape > 0`, apply sinusoidal positional encoding to `features`.
    $$ \text{encoded\_features} = \text{PE}(\text{features}, \text{feape}) $$
    Where $\text{PE}(x, L)$ produces a tensor of shape $(N, \text{inChanel} + 2 \cdot L \cdot \text{inChanel})$.

3.  **Input Concatenation:**
    Concatenate the raw and encoded tensors along the last dimension to form `mlp_in`.
    $$ \text{mlp\_in} = \text{Concat}([\text{features}, \text{viewdirs}, \text{encoded\_features (if } feape>0), \text{encoded\_viewdirs (if } viewpe>0)]) $$
    The order is: `[features, viewdirs, positional_encoding(features, feape), positional_encoding(viewdirs, viewpe)]` [S5, unofficial].

4.  **MLP Forward Pass:**
    Pass `mlp_in` through the 3-layer MLP.
    *   $h_1 = \text{ReLU}(\text{Linear}_1(\text{mlp\_in}))$
    *   $h_2 = \text{ReLU}(\text{Linear}_2(h_1))$
    *   $rgb_{raw} = \text{Linear}_3(h_2)$

5.  **Output Activation:**
    Apply sigmoid activation to clamp RGB values to $[0, 1]$.
    $$ rgb = \sigma(rgb_{raw}) $$

6.  **Initialization:**
    The bias of the final layer (`layer3`) is initialized to 0 [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def positional_encoding(x, embedding_size):
    """
    Apply sinusoidal positional encoding to input tensor.
    Args:
        x: Input tensor of shape (..., D)
        embedding_size: Number of frequencies L
    Returns:
        Encoded tensor of shape (..., D + 2*L*D)
    """
    out = [x]
    for i in range(embedding_size):
        for fn in [torch.sin, torch.cos]:
            out.append(fn(2 ** i * x))
    return torch.cat(out, dim=-1)

class MLPRender_Fea(nn.Module):
    def __init__(self, inChanel, viewpe=6, feape=0, featureC=128):
        super().__init__()
        self.viewpe = viewpe
        self.feape = feape
        self.featureC = featureC
        
        # Calculate input dimension
        # Raw viewdirs: 3
        # Encoded viewdirs: 2 * viewpe * 3
        # Raw features: inChanel
        # Encoded features: 2 * feape * inChanel
        self.in_mlpC = 3 + 2 * viewpe * 3 + inChanel + (2 * feape * inChanel if feape > 0 else 0)
        
        # 3-layer MLP
        self.layer1 = nn.Linear(self.in_mlpC, featureC)
        self.layer2 = nn.Linear(featureC, featureC)
        self.layer3 = nn.Linear(featureC, 3)
        
        # Initialize bias of final layer to 0
        nn.init.constant_(self.layer3.bias, 0)

    def forward(self, features, viewdirs):
        """
        Args:
            features: Tensor of shape (N, inChanel)
            viewdirs: Tensor of shape (N, 3)
        Returns:
            rgb: Tensor of shape (N, 3)
        """
        indata = [features, viewdirs]
        
        if self.feape > 0:
            indata.append(positional_encoding(features, self.feape))
        if self.viewpe > 0:
            indata.append(positional_encoding(viewdirs, self.viewpe))
            
        mlp_in = torch.cat(indata, dim=-1)
        
        h = torch.relu(self.layer1(mlp_in))
        h = torch.relu(self.layer2(h))
        rgb_raw = self.layer3(h)
        
        rgb = torch.sigmoid(rgb_raw)
        return rgb
```

**Edge Cases and Pitfalls:**
*   **Feature Dimensionality:** The positional encoding for features assumes the input `features` tensor has a dimension that can be encoded. If `feape > 0`, the code in [S5, unofficial] implies `inChanel` is the dimension being encoded. If `inChanel` is not 3, the PE expansion factor is `2 * feape * inChanel`.
*   **View Direction Normalization:** View directions should be normalized before encoding to ensure consistent frequency scaling, although the source code does not explicitly show normalization in the forward pass [S5, unofficial].
*   **Bias Initialization:** Failing to initialize the final layer bias to 0 may lead to slow convergence or biased initial predictions [S5, unofficial].

## Subtopics

### [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md)
Extends MLPRender_Fea by adding a per-image embedding vector to the input, allowing the MLP to account for global appearance shifts like lighting changes across different views.

### [Volume Radiance Network](2576-volume-radiance-network.md)
A variant that may use different feature sources or MLP structures for radiance prediction.

### [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md)
Replaces the nonlinear MLP decoder with a linear decoder using a learned basis, explicitly modeling view dependence without a black-box MLP.

### [Hybrid View-dependent Color Modeling](5083-hybrid-view-dependent-color-modeling.md)
Combines a sparse feature voxel grid with a shallow MLP, separating view-independent and view-dependent components, whereas MLPRender_Fea applies PE to features and view direction but not position.

### [Color Network Architecture](7200-color-network-architecture.md)
Defines specific layer widths and activation functions for the MLP module in dynamic view synthesis contexts.

### [Neural BRDF and Geometry Decoding](8139-neural-brdf-and-geometry-decoding.md)
Decodes features into explicit BRDF values and normals, extending the color prediction to physically based rendering.

### [MLPRender_Fea_WeakView](10541-mlprender-fea-weakview.md)
A variant that decouples density from color, deriving density from a specific feature channel while maintaining view-independence for volumetric properties.

### [Gaussian Activated Radiance Fields Render](10546-gaussian-activated-radiance-fields-render.md)
Uses Gaussian activation functions instead of ReLU to model view-dependent appearance.

### Neural Texture Decoder
A decoder for texture generation that maps a 199-dim vector to a 32x32x32x4 tensor via fully-connected and 3D convolutional layers, outputting a 64x64x64x4 tensor [S7, unofficial].

### Volumetric Appearance Network Layout
An MLP that takes encoded position, spherical harmonics encoding of view direction, normal vector, and geometric feature vector as input. It uses three layers with 128, 128, and 64 features, GELU activations, and a Sigmoid output for RGB and transparency [S8].

### MLP Opacity Regression
A fusion ablation technique where opacity is regressed from an MLP using 8-dim feature vectors sampled from HexPlane, boosting PSNR/SSIM/LPIPS at the cost of slower rendering [S9].

## Operating characteristics
*   **Time Complexity:** The MLP forward pass is $O(N \cdot (\text{in\_mlpC} \cdot \text{featureC} + \text{featureC}^2 + \text{featureC} \cdot 3))$, where $N$ is the number of samples. Positional encoding is $O(N \cdot \text{inChanel} \cdot \text{feape})$ and $O(N \cdot 3 \cdot \text{viewpe})$.
*   **Memory Complexity:** $O(\text{in\_mlpC} \cdot \text{featureC} + \text{featureC}^2 + \text{featureC} \cdot 3)$ for weights.
*   **Performance:** The use of positional encoding for view directions allows the network to capture high-frequency view-dependent effects like specular highlights [S5, unofficial]. The separation of feature and view encoding allows for flexible integration with various feature extraction methods (e.g., from a voxel grid or hash table).
*   **Strengths:** Simple architecture, easy to integrate with different feature representations, effective for view-dependent appearance.
*   **Limitations:** The MLP may struggle with very high-frequency details compared to more complex architectures like InstantNGP [5049-instantngp-network-architecture.md]. The reliance on positional encoding can lead to spectral bias, where the network learns low-frequency functions first.

## Background
MLPRender_Fea is a component of the Neural Radiance Fields (NeRF) family, specifically designed for color prediction given features and view directions. It builds on the original NeRF MLP architecture [910-nerf-mlp-architecture.md] by decoupling the color prediction from the density prediction and explicitly encoding features and view directions. The method is used in various neural rendering pipelines where features are extracted from a separate representation (e.g., TensoRF, InstantNGP) and decoded into color.

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S3] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S4] AlbertAffectus/NeRF, https://github.com/AlbertAffectus/NeRF, fetched 2026-10-27 (unofficial)
[S5] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S6] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
[S7] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S8] Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes, https://doi.org/10.1109/cvpr52734.2025.01991, fetched 2026-10-26
[S9] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
