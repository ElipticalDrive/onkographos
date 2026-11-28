# MLP with Positional Encoding

Also called: MLP Render with Positional Encoding, View-Dependent Color MLP

## Overview
A neural rendering module that predicts RGB color from input features, view directions, and 3D positions using a Multi-Layer Perceptron (MLP) with positional encoding applied to both position and view direction [S2, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `pts`: 3D position coordinates, shape $(N, 3)$ [S2, unofficial].
    *   `viewdirs`: View direction vectors, shape $(N, 3)$ [S2, unofficial].
    *   `features`: Input feature vector from the volume representation, shape $(N, inChanel)$ [S2, unofficial][S3, unofficial].
*   **Outputs:**
    *   `rgb`: Predicted RGB color values, shape $(N, 3)$, bounded to $[0, 1]$ by a sigmoid activation [S2, unofficial][S3, unofficial].

**Parameters:**
*   `inChanel`: Input feature dimension from the volume representation [S3, unofficial].
*   `viewpe`: Number of frequency bands for view direction positional encoding. Default: 6 [S2, unofficial].
*   `pospe`: Number of frequency bands for position positional encoding. Default: 6 [S2, unofficial].
*   `featureC`: Hidden dimension size for the MLP layers. Default: 128 [S2, unofficial][S3, unofficial].

**Data Structures:**
The MLP consists of a sequence of fully connected layers [S2, unofficial][S3, unofficial]. A standard implementation uses 3 linear layers for the core mapping [S3, unofficial]:
*   Layer 1: Linear($in\_mlpC$, $featureC$) [S3, unofficial].
*   Layer 2: Linear($featureC$, $featureC$) [S3, unofficial].
*   Layer 3: Linear($featureC$, 3) [S3, unofficial].
*   Activations: ReLU applied after Layer 1 and Layer 2 [S2, unofficial][S3, unofficial]. Sigmoid applied to the output of Layer 3 [S2, unofficial][S3, unofficial].

**Formulas:**
The input dimension to the MLP, $in\_mlpC$, is calculated as [S3, unofficial]:
$$ in\_mlpC = (3 + 2 \cdot viewpe \cdot 3) + (3 + 2 \cdot pospe \cdot 3) + inChanel $$
This accounts for raw viewdirs (3), encoded viewdirs ($2 \cdot viewpe \cdot 3$), raw pts (3), encoded pts ($2 \cdot pospe \cdot 3$), and features ($inChanel$) [S3, unofficial].

The positional encoding function $\gamma$ for a coordinate $x$ with $L$ bands is implemented as [G]:
$$ \gamma(x) = [\sin(2^0 \pi x), \cos(2^0 \pi x), \dots, \sin(2^{L-1} \pi x), \cos(2^{L-1} \pi x)] $$

**Steps:**
1.  Initialize an input list `indata` with the raw features and raw view directions: `indata = [features, viewdirs]` [S3, unofficial].
2.  If `pospe > 0`, compute the positional encoding of `pts` and append it to `indata`: `indata += [positional_encoding(pts, pospe)]` [S2, unofficial][S3, unofficial]. Input features are not positional encoded in this standard variant [S3, unofficial].
3.  If `viewpe > 0`, compute the positional encoding of `viewdirs` and append it to `indata`: `indata += [positional_encoding(viewdirs, viewpe)]` [S2, unofficial][S3, unofficial].
4.  Concatenate all elements in `indata` along the last dimension to form the MLP input tensor `mlp_in`: `mlp_in = torch.cat(indata, dim=-1)` [S2, unofficial][S3, unofficial].
5.  Pass `mlp_in` through the MLP sequence: $h_1 = \text{ReLU}(\text{Linear}_1(mlp\_in))$, $h_2 = \text{ReLU}(\text{Linear}_2(h_1))$, $rgb_{raw} = \text{Linear}_3(h_2)$ [S2, unofficial][S3, unofficial].
6.  Apply sigmoid activation to the output: `rgb = torch.sigmoid(rgb_raw)` [S2, unofficial][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class PositionalEncoding(nn.Module):
    def __init__(self, num_bands):
        super().__init__()
        self.num_bands = num_bands

    def forward(self, x):
        # x shape: (..., D)
        # Output shape: (..., D * 2 * num_bands)
        freq_bands = 2.0 ** torch.arange(self.num_bands, device=x.device).float()
        scaled_x = x.unsqueeze(1) * freq_bands.unsqueeze(0).unsqueeze(-1)
        sin_x = torch.sin(scaled_x)
        cos_x = torch.cos(scaled_x)
        return torch.cat([sin_x, cos_x], dim=-1)

class MLPRenderPE(nn.Module):
    def __init__(self, inChanel, viewpe=6, pospe=6, featureC=128):
        super().__init__()
        self.viewpe = viewpe
        self.pospe = pospe
        self.inChanel = inChanel
        self.featureC = featureC
        
        # Calculate input dimension to MLP
        self.in_mlpC = (3 + 2 * viewpe * 3) + (3 + 2 * pospe * 3) + inChanel
        
        # Positional Encoders
        self.pos_encoder = PositionalEncoding(pospe)
        self.view_encoder = PositionalEncoding(viewpe)
        
        # MLP Layers
        self.layer1 = nn.Linear(self.in_mlpC, featureC)
        self.layer2 = nn.Linear(featureC, featureC)
        self.layer3 = nn.Linear(featureC, 3)
        
        # Initialize bias of final layer to 0
        torch.nn.init.constant_(self.layer3.bias, 0)

    def forward(self, pts, viewdirs, features):
        # pts: (N, 3)
        # viewdirs: (N, 3)
        # features: (N, inChanel)
        
        indata = [features, viewdirs]
        
        if self.pospe > 0:
            indata.append(self.pos_encoder(pts))
            
        if self.viewpe > 0:
            indata.append(self.view_encoder(viewdirs))
            
        # Concatenate along the last dimension
        mlp_in = torch.cat(indata, dim=-1)
        
        # Forward pass through MLP
        h1 = torch.relu(self.layer1(mlp_in))
        h2 = torch.relu(self.layer2(h1))
        rgb_raw = self.layer3(h2)
        
        # Apply sigmoid to get RGB in [0, 1]
        rgb = torch.sigmoid(rgb_raw)
        
        return rgb
```

**Edge Cases and Pitfalls:**
*   **Precision:** High-frequency positional encoding can lead to numerical instability if the input coordinates are not normalized to a small range (e.g., $[-1, 1]$ or $[0, 1]$) [G].
*   **Boundary Conditions:** The sigmoid activation ensures output is bounded, but if the raw output is very large negative, the gradient vanishes [G]. Initializing the final bias to 0 helps start training near the middle of the sigmoid [S2, unofficial][S3, unofficial].
*   **Feature Encoding:** In this specific variant, input features are *not* positional encoded, only `pts` and `viewdirs` [S3, unofficial].

## Subtopics

### MLP with Per-layer Inputs Incorporation
An improved MLP structure for few-shot view synthesis that mitigates overfitting by incorporating both 3D positions and 2D viewing directions at each layer input, ensuring each layer explicitly perceives the five-dimensional input embedding [S4]. It differs from the standard MLP with PE by injecting the encoded input $\gamma_L(X)$ into every layer's computation rather than just the first [S4]. The formula is $f_i = \phi_i(f_{i-1}, \gamma_L(X))$, with $f_1 = \phi_1(\gamma_L(X))$, where $X$ is the 5D info $(x,y,z,\theta,\phi)$ [S4].

### Ablation variants for positional encoding
Variants studied in [S5, unofficial] to understand the role of position in proposers:
*   **Blind proposer:** Ignores input coarse features, always outputs identical proposals [S5, unofficial].
*   **No-position proposer:** Ignores coarse sample positions along the ray, making it insensitive to feature ordering [S5, unofficial]. It produces identical proposals for colinear rays of opposite directionality and fails on non-forward-facing scenes like Blender, though it is competitive on LLFF-NeRF [S5, unofficial].
*   **Concat proposer:** Concatenates positional encodings with features instead of summing [S5, unofficial].
*   **Learnt-position proposer:** Learns positional encodings instead of using sinusoidal ones, summed with features [S5, unofficial].

### MLCA (Hybrid Spatial Feature Learning Module)
A module added after position encoding and before the MLP to capture spatial feature information and mix channels [S6]. It differs from the standard MLP with PE by adding two fully connected layers with GELU activation between the encoded input and the main MLP [S6]. Steps: 1. Concatenate $PE(x)$ and $PE(d)$ to get $X_{input}$. 2. $X_{token} = GELU(W_{token} X_{input} + b_{token})$. 3. $X_{channel} = GELU(W_{channel} X_{token} + b_{channel})$ [S6].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $N$ and the input dimension $in\_mlpC$ [G]. The positional encoding step is $O(N \cdot (viewpe + pospe))$ [G]. The MLP forward pass is $O(N \cdot (in\_mlpC \cdot featureC + featureC^2 + featureC \cdot 3))$ [G].
*   **Memory Complexity:** $O(N \cdot in\_mlpC)$ to store the concatenated input tensor [G]. The model parameters are $O(in\_mlpC \cdot featureC + featureC^2 + featureC \cdot 3)$ [G].
*   **Performance:** Standard parameters are `viewpe=6`, `pospe=6`, `featureC=128` [S2, unofficial]. This configuration is widely used in NeRF-based methods to achieve high-quality view synthesis [S2, unofficial][S3, unofficial].
*   **Strengths:** Allows the MLP to learn high-frequency details by lifting inputs to a high-frequency space [G]. Simple to implement and integrate into existing pipelines [G].
*   **Limitations:** Can overfit to training data if not regularized, especially in few-shot settings [S4]. The high input dimension can lead to larger memory footprint and slower training compared to methods that do not use PE or use fewer bands [G].

## Background
The method originates from the Neural Radiance Fields (NeRF) paper [S2, unofficial][S3, unofficial], which introduced the use of positional encoding for both position and view direction in an MLP to predict color and density. It has since become a standard component in many neural rendering architectures [S2, unofficial][S3, unofficial]. Variants and ablations have been studied to understand the impact of positional encoding on sampling and view synthesis quality [S5, unofficial].

## Sources
[S1] UV Volumes for Real-time Rendering of Editable Free-view Human Performance, https://doi.org/10.1109/cvpr52729.2023.01595, fetched 2026-10-26
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S4] DASNeRF: depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse view neural radiance fields., https://doi.org/10.1371/journal.pone.0321878, fetched 2026-10-27
[S5] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S6] Mixer-NeRF: Research on 3D Reconstruction Methods of Neural Radiance Fields Based on Hybrid Spatial Feature Information, https://doi.org/10.54097/0ek2xd53, fetched 2026-10-27
