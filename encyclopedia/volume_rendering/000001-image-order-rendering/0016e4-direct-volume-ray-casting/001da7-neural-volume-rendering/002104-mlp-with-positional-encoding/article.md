# MLP with Positional Encoding

Also called: MLP Render with Positional Encoding

## Overview
A neural rendering module that predicts RGB color from input features, view directions, and 3D positions using a Multi-Layer Perceptron (MLP) with positional encoding applied to both position and view direction [S1, S2, unofficial]. It builds on [Neural volume rendering](1576-neural-volume-rendering.md) by providing the specific mapping function $f_\theta$ that transforms encoded spatial and directional inputs into radiance values. It is distinguished from [MLP with Positional Encoding and IOR Activation](4947-mlp-with-positional-encoding-and-ior-activation.md) by applying a standard sigmoid activation for color output without Index of Refraction adjustments [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `pts`: 3D spatial coordinates $(x, y, z)$ [S1, S2, unofficial].
    *   `viewdirs`: 2D viewing direction $(\theta, \phi)$ or 3D direction vector [S1, unofficial][S3].
    *   `features`: Input feature vector from the volume representation (e.g., from a feature grid or previous layer) [S1, S2, unofficial].
*   **Outputs:**
    *   `rgb`: Predicted RGB color vector [S1, S2, unofficial].

**Data Structures and Parameters:**
*   `inChanel`: Input feature dimension from the volume representation [S2, unofficial].
*   `viewpe`: Number of frequency bands for view direction positional encoding. Default: 6 [S1, unofficial].
*   `pospe`: Number of frequency bands for position positional encoding. Default: 6 [S1, unofficial].
*   `featureC`: Hidden dimension size for the MLP layers. Default: 128 [S2, unofficial].
*   `mlp_in`: Concatenated input vector to the MLP [S1, unofficial].

**Input Dimension Calculation:**
The total input dimension to the MLP, $in\_mlpC$, is calculated as [S2, unofficial]:
$$ in\_mlpC = (3 + 2 \cdot viewpe \cdot 3) + (3 + 2 \cdot pospe \cdot 3) + inChanel $$
This accounts for:
*   Raw viewdirs (3 components) [S2, unofficial].
*   PE-encoded viewdirs ($2 \cdot viewpe \cdot 3$ components) [S2, unofficial].
*   Raw pts (3 components) [S2, unofficial].
*   PE-encoded pts ($2 \cdot pospe \cdot 3$ components) [S2, unofficial].
*   Raw features (`inChanel`) [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize the bias of the final layer (layer3) to 0 [S2, unofficial].
2.  **Input Assembly:** Initialize a list `indata` with `[features, viewdirs]` [S2, unofficial].
3.  **Positional Encoding (Position):** If `pospe > 0`, compute the positional encoding of `pts` and append it to `indata` [S1, unofficial].
4.  **Positional Encoding (View):** If `viewpe > 0`, compute the positional encoding of `viewdirs` and append it to `indata` [S1, unofficial].
5.  **Concatenation:** Concatenate all elements in `indata` along the last dimension to form `mlp_in` [S1, unofficial].
    $$ mlp\_in = \text{concat}(indata, \text{dim}=-1) $$
6.  **MLP Forward Pass:** Pass `mlp_in` through the MLP sequence:
    *   Layer 1: Linear transformation from $in\_mlpC$ to `featureC`, followed by ReLU activation.
    *   Layer 2: Linear transformation from `featureC` to `featureC`, followed by ReLU activation.
    *   Layer 3: Linear transformation from `featureC` to 3 (RGB).
7.  **Output Activation:** Apply sigmoid activation to the output of Layer 3 to constrain RGB values to $[0, 1]$ [S1, S2, unofficial].
    $$ rgb = \text{sigmoid}(\text{Layer3}(\text{ReLU}(\text{Layer2}(\text{ReLU}(\text{Layer1}(mlp\_in)))))) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class MLPWithPE(nn.Module):
    def __init__(self, inChanel, viewpe=6, pospe=6, featureC=128):
        super().__init__()
        self.viewpe = viewpe
        self.pospe = pospe
        self.featureC = featureC
        
        # Calculate input dimension
        # Raw viewdirs (3) + PE viewdirs (2*viewpe*3) + Raw pts (3) + PE pts (2*pospe*3) + features (inChanel)
        self.in_mlpC = (3 + 2 * viewpe * 3) + (3 + 2 * pospe * 3) + inChanel
        
        # MLP Layers
        self.layer1 = nn.Linear(self.in_mlpC, featureC)
        self.layer2 = nn.Linear(featureC, featureC)
        self.layer3 = nn.Linear(featureC, 3)
        
        # Initialize bias of final layer to 0
        nn.init.constant_(self.layer3.bias, 0)

    def forward(self, pts, viewdirs, features):
        # pts: [N, 3]
        # viewdirs: [N, 3]
        # features: [N, inChanel]
        
        indata = [features, viewdirs]
        
        if self.pospe > 0:
            # Assuming a standard positional encoding function exists
            # PE expands 3D vector to (2*L+1)*3 dimensions
            pts_pe = positional_encoding(pts, self.pospe)
            indata.append(pts_pe)
            
        if self.viewpe > 0:
            viewdirs_pe = positional_encoding(viewdirs, self.viewpe)
            indata.append(viewdirs_pe)
            
        mlp_in = torch.cat(indata, dim=-1)
        
        x = self.layer1(mlp_in)
        x = torch.relu(x)
        x = self.layer2(x)
        x = torch.relu(x)
        rgb = self.layer3(x)
        rgb = torch.sigmoid(rgb)
        
        return rgb
```

**Edge Cases and Pitfalls:**
*   **Precision:** Standard positional encoding uses sinusoidal functions which can be sensitive to the scale of input coordinates. Ensure inputs are normalized appropriately before encoding [S1, S2, unofficial].
*   **Boundary Conditions:** The sigmoid activation ensures output is within $[0, 1]$, preventing invalid color values [S1, unofficial].
*   **Degenerate Input:** If `viewpe` or `pospe` is 0, the corresponding encoded components are omitted, reducing the input dimension [S1, unofficial].

## Subtopics

### MLP with Per-layer Inputs Incorporation
An improved MLP network structure for few-shot view synthesis that mitigates overfitting by incorporating both 3D positions and 2D viewing directions at each layer input, ensuring each layer explicitly perceives the five-dimensional input embedding [S3]. It differs from the standard MLP with PE by injecting the encoded input $\gamma_L(X)$ at every layer, rather than only at the input. The input fusion formula is $f_i = \phi_i(f_{i-1}, \gamma_L(X))$, with $f_1 = \phi_1(\gamma_L(X))$, where $X$ represents the five-dimensional information $(x,y,z,\theta,\phi)$ [S3].

### Ablation variants for positional encoding
Variants studied in [NeRF in detail: Learning to sample for view synthesis](4848-nerf-in-detail-learning-to-sample-for-view-synthesis.md) [S4, unofficial]:
*   **Blind proposer:** Ignores input coarse features, always outputs identical proposals. Performs badly [S4, unofficial].
*   **No-position proposer:** Ignores coarse sample positions along the ray, making it insensitive to feature ordering. Produces identical proposals for colinear rays of opposite directionality. Competitive on LLFF-NeRF (forward-facing scenes) but fails on Blender [S4, unofficial].
*   **Concat proposer:** Concatenates positional encodings with features instead of summing. Performs similarly to the standard Pool proposer [S4, unofficial].
*   **Learnt-position proposer:** Learns positional encodings instead of using sinusoidal ones, summed with features. Performs similarly to the standard Pool proposer [S4, unofficial].

### MLCA (Hybrid Spatial Feature Learning Module)
A module added after position encoding and before the MLP to capture spatial feature information and mix channels of these features, establishing mixed associations in feature dimension space [S5]. It processes the concatenated position-encoded spatial position $x$ and direction $d$ ($X_{input} = Concat(PE(x), PE(d))$) through two fully connected layers with GELU activation: $X_{token} = GELU(W_{token} X_{input} + b_{token})$ and $X_{channel} = GELU(W_{channel} X_{token} + b_{channel})$ [S5].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sample points $N$ and the input dimension $in\_mlpC$. The positional encoding step is also linear in $N$ and the number of frequency bands.
*   **Memory Complexity:** Linear with respect to $N$ for storing intermediate activations. The model parameters are fixed based on `featureC` and `inChanel`.
*   **Strengths:** Simple architecture, differentiable, effective for learning view-dependent effects when combined with positional encoding [S1, S2, unofficial].
*   **Limitations:** Can overfit on small datasets without regularization or multi-input structures [S3]. Standard positional encoding may struggle with high-frequency details compared to learned or hybrid approaches [S4, unofficial][S5].
*   **When to use:** Standard neural volume rendering where view-dependent effects are important and computational resources are limited.
*   **When not to use:** When high-frequency detail recovery is critical and overfitting is a concern (consider [MLP with Per-layer Inputs Incorporation](4051-mlp-with-per-layer-inputs-incorporation.md) or [MLCA](4947-mlca-hybrid-spatial-feature-learning-module.md)).

## Background
Originated in the context of Neural Radiance Fields (NeRF) [S1, S2, unofficial]. The use of positional encoding for MLPs in neural rendering was popularized by [NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis](1576-neural-volume-rendering.md) [S2, S4, unofficial]. The specific implementation details described here are derived from common implementations in the community [S1, S2, unofficial].

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S2] Collaborative Neural Radiance Fields for Novel View Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S3] DASNeRF: depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse view neural radiance fields., https://doi.org/10.1371/journal.pone.0321878, fetched 2026-10-27
[S4] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S5] Mixer-NeRF: Research on 3D Reconstruction Methods of Neural Radiance Fields Based on Hybrid Spatial Feature Information, https://doi.org/10.54097/0ek2xd53, fetched 2026-10-27
