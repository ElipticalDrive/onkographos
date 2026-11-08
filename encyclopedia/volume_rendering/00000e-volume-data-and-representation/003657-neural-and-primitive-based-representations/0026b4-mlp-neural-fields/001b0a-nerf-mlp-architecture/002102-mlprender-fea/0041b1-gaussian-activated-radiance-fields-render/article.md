# Gaussian Activated Radiance Fields Render


## Overview
Gaussian Activated Radiance Fields Render (GaRF_Render) is a neural rendering module that predicts RGB color from input features and view directions using a Multi-Layer Perceptron (MLP) with Gaussian activation functions applied to the feature and view direction inputs [S1, unofficial]. It builds on [MLPRender_Fea](2414-mlprender-fea.md) by replacing standard activation functions with Gaussian kernels to model view-dependent appearance, distinguishing it from siblings like [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which use static embeddings, or [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md) which use linear projections [S1, unofficial].

## How it works
**Inputs:**
- Input feature vector of size `inChanel`.
- View direction vector (implicitly handled via the `view_gaussian_linear` layer which expects a 3D input, typically the view direction or a transformed version thereof).

**Outputs:**
- RGB color vector of size 3.

**Data Structures:**
The module consists of a sequence of linear layers with specific initialization and activation patterns.
1. `fea_gaussian_linear`: Linear layer mapping `inChanel` to `featureC`.
2. `view_gaussian_linear`: Linear layer mapping 3 (`mid_mlpC`) to `featureC`.
3. `layer1`: Linear layer mapping `featureC` to `featureC`.
4. `layer2`: Linear layer mapping `2 * featureC` to `featureC`.
5. `layer3`: Linear layer mapping `featureC` to 3 (RGB).

**Parameters:**
- `inChanel`: Input feature channel size. [S1, unofficial]
- `viewpe`: Number of view positional encoding frequencies. Default: 6. [S1, unofficial]
- `feape`: Number of feature positional encoding frequencies. Default: 6. [S1, unofficial]
- `featureC`: Hidden layer channel size. Default: 128. [S1, unofficial]
- `mid_mlpC`: Fixed to 3. [S1, unofficial]

**Initialization:**
All weight matrices for the linear layers (`fea_gaussian_linear`, `view_gaussian_linear`, `layer1`, `layer2`, `layer3`) are initialized using a uniform distribution. The bias terms are not explicitly initialized in the provided snippet, implying default initialization (typically zero). [S1, unofficial]

**Algorithm:**
The rendering process involves transforming the input features and view directions through Gaussian-activated linear layers, concatenating them, and passing them through subsequent MLP layers to predict color.

1. **Feature Transformation:**
   - Apply positional encoding to the input features if `feape > 0`. Let the encoded features be $F_{enc}$.
   - Pass $F_{enc}$ through `fea_gaussian_linear`.
   - Apply Gaussian activation: $G_f = \exp(-\frac{1}{2} (W_f F_{enc} + b_f)^2)$. Note: The source code name `gaussian_linear` implies the activation is Gaussian, but standard PyTorch `nn.Linear` does not include activation. The implementation likely applies a Gaussian activation function to the output of the linear layer. [S1, unofficial]

2. **View Direction Transformation:**
   - Apply positional encoding to the view direction if `viewpe > 0`. Let the encoded view direction be $V_{enc}$.
   - Pass $V_{enc}$ through `view_gaussian_linear`.
   - Apply Gaussian activation: $G_v = \exp(-\frac{1}{2} (W_v V_{enc} + b_v)^2)$. [S1, unofficial]

3. **Concatenation and MLP:**
   - Concatenate $G_f$ and $G_v$ along the channel dimension: $C = \text{concat}(G_f, G_v)$.
   - Pass $C$ through `layer1`: $H_1 = \text{Linear}_1(C)$.
   - Apply activation (not specified in source, likely ReLU or similar, but source only shows linear layers. Assuming standard MLP behavior or that the Gaussian activation is the primary non-linearity). [S1, unofficial]
   - Concatenate $H_1$ with $G_f$ and $G_v$? The source shows `layer2` takes `featureC + featureC`. This suggests concatenating the output of `layer1` with the original Gaussian features or just the two Gaussian features. Given `layer2` input is `featureC + featureC`, and `layer1` output is `featureC`, it is likely $H_2 = \text{Linear}_2(\text{concat}(H_1, G_f))$ or $\text{concat}(H_1, G_v)$. However, the source code `self.layer2 = torch.nn.Linear(featureC + featureC , featureC)` suggests the input to layer2 is the concatenation of two `featureC` vectors. The most logical interpretation in such architectures is concatenating the transformed feature and view branches. Let $H_{cat} = \text{concat}(G_f, G_v)$. Then $H_1 = \text{Linear}_1(H_{cat})$. Then $H_2 = \text{Linear}_2(\text{concat}(H_1, H_{cat}))$? Or simply $H_2 = \text{Linear}_2(\text{concat}(G_f, G_v))$? The source code structure is:
     - `layer1` takes `featureC` (output of `fea_gaussian_linear`? No, `layer1` input size is `featureC`).
     - `layer2` input size is `featureC + featureC`.
     - `layer3` input size is `featureC`.
     
     Re-evaluating the source code:
     ```python
     self.fea_gaussian_linear = torch.nn.Linear(self.in_mlpC, featureC)
     self.view_gaussian_linear = torch.nn.Linear(self.mid_mlpC, featureC)
     self.layer1 = torch.nn.Linear(featureC, featureC)
     self.layer2 = torch.nn.Linear(featureC + featureC , featureC)
     self.layer3 = torch.nn.Linear(featureC , 3 )
     ```
     This structure is ambiguous without the forward pass. However, a common pattern is:
     1. $F_g = \text{Gaussian}(W_f F + b_f)$
     2. $V_g = \text{Gaussian}(W_v V + b_v)$
     3. $H_1 = \text{Linear}(F_g)$ (or $V_g$)
     4. $H_2 = \text{Linear}(\text{concat}(H_1, F_g, V_g))$? No, size mismatch.
     
     Let's assume the following standard interpretation for such "Gaussian Activated" networks:
     1. Encode $F$ and $V$.
     2. $G_f = \text{Gaussian}(W_f F_{enc})$
     3. $G_v = \text{Gaussian}(W_v V_{enc})$
     4. $H_1 = \text{Linear}(G_f)$
     5. $H_2 = \text{Linear}(\text{concat}(H_1, G_v))$
     6. $RGB = \text{Linear}(H_2)$
     
     This matches the dimensions:
     - `layer1`: `featureC` -> `featureC` (takes $G_f$)
     - `layer2`: `featureC + featureC` -> `featureC` (takes $H_1$ and $G_v$)
     - `layer3`: `featureC` -> 3 (takes $H_2$)

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

def gaussian_activation(x):
    """Applies Gaussian activation: exp(-0.5 * x^2)"""
    return torch.exp(-0.5 * x ** 2)

def positional_encoding(x, num_frequencies):
    """
    Standard positional encoding for NeRF.
    x: [..., D]
    Returns: [..., D * 2 * num_frequencies]
    """
    if num_frequencies == 0:
        return x
    
    # Reshape for broadcasting
    # x shape: [..., D]
    # freq_bands shape: [num_frequencies, 2]
    freq_bands = torch.linspace(1, num_frequencies, num_frequencies, device=x.device).reshape(1, -1, 1)
    # x shape: [..., D, 1]
    x = x.unsqueeze(-1)
    # freq_bands shape: [1, num_frequencies, 2]
    # Output shape: [..., D, num_frequencies, 2]
    out = torch.cat([torch.sin(x * math.pi * freq_bands), torch.cos(x * math.pi * freq_bands)], dim=-1)
    # Reshape to [..., D * num_frequencies * 2]
    return out.reshape(*x.shape[:-1], -1)

class GaRF_Render(nn.Module):
    def __init__(self, inChanel, viewpe=6, feape=6, featureC=128):
        super(GaRF_Render, self).__init__()
        self.in_mlpC = inChanel
        self.mid_mlpC = 3
        self.viewpe = viewpe
        self.feape = feape
        self.featureC = featureC
        
        # Linear layers
        self.fea_gaussian_linear = nn.Linear(self.in_mlpC, self.featureC)
        self.view_gaussian_linear = nn.Linear(self.mid_mlpC, self.featureC)
        self.layer1 = nn.Linear(self.featureC, self.featureC)
        self.layer2 = nn.Linear(self.featureC + self.featureC, self.featureC)
        self.layer3 = nn.Linear(self.featureC, 3)
        
        # Uniform initialization of weights
        self._uniform_init_weights()

    def _uniform_init_weights(self):
        """Initialize weights with uniform distribution as per source."""
        for m in [self.fea_gaussian_linear, self.view_gaussian_linear, 
                  self.layer1, self.layer2, self.layer3]:
            nn.init.uniform_(m.weight)

    def forward(self, features, view_dirs):
        """
        features: [B, N, inChanel]
        view_dirs: [B, N, 3]
        Returns: [B, N, 3]
        """
        # 1. Positional Encoding
        if self.feape > 0:
            features_enc = positional_encoding(features, self.feape)
        else:
            features_enc = features
            
        if self.viewpe > 0:
            view_dirs_enc = positional_encoding(view_dirs, self.viewpe)
        else:
            view_dirs_enc = view_dirs

        # 2. Gaussian Activated Linear Layers
        # Apply linear transform then Gaussian activation
        fea_gaussian = gaussian_activation(self.fea_gaussian_linear(features_enc))
        view_gaussian = gaussian_activation(self.view_gaussian_linear(view_dirs_enc))

        # 3. MLP Layers
        # layer1 takes featureC (output of fea_gaussian_linear)
        h1 = self.layer1(fea_gaussian)
        
        # layer2 takes featureC + featureC (concat of h1 and view_gaussian)
        # Note: Source code layer2 input is featureC + featureC. 
        # h1 is featureC, view_gaussian is featureC.
        h2_input = torch.cat([h1, view_gaussian], dim=-1)
        h2 = self.layer2(h2_input)
        
        # layer3 takes featureC
        rgb = self.layer3(h2)
        
        return rgb
```

## Subtopics

### MLPRender_Fea
The parent module that predicts RGB from features and view directions using an MLP with positional encoding on features and view direction, but not position. [Link](2414-mlprender-fea.md)

### NeRF with Per-Image Embeddings
Uses per-image latent codes to capture view-independent appearance variations. [Link](2484-nerf-with-per-image-embeddings.md)

### Linear Feature Decoder with Learned Color Basis
Uses a linear decoder with a learned basis for color prediction. [Link](3352-linear-feature-decoder-with-learned-color-basis.md)

### Hybrid View-dependent Color Modeling
Combines different strategies for modeling view-dependent effects. [Link](5083-hybrid-view-dependent-color-modeling.md)

### Color Network Architecture
General architecture for color prediction in neural rendering. [Link](7200-color-network-architecture.md)

### Neural BRDF and Geometry Decoding
Decodes BRDF parameters and geometry simultaneously. [Link](8139-neural-brdf-and-geometry-decoding.md)

### MLPRender_Fea_WeakView
A variant with weak view dependence. [Link](10541-mlprender-fea-weakview.md)

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of features and hidden dimension. The Gaussian activation adds a constant overhead per element.
- **Memory Complexity**: Proportional to the number of parameters in the linear layers. For `inChanel`, `featureC`, the parameters are roughly $2 \cdot inChanel \cdot featureC + 2 \cdot 3 \cdot featureC + 2 \cdot featureC^2 + 2 \cdot featureC^2 + 2 \cdot featureC \cdot 3$.
- **Strengths**: Gaussian activations can model sharp view-dependent effects more effectively than standard activations like ReLU in some contexts.
- **Limitations**: The specific initialization and activation behavior may require careful tuning. The source code does not specify the activation for `layer1`, `layer2`, and `layer3`, which could lead to implementation ambiguity.
- **When to use**: When modeling view-dependent appearance with Gaussian-like kernels is beneficial.
- **When not to use**: When computational efficiency is critical and standard activations suffice.

## Background
The method is based on the GaRF source code from the Joint-TensoRF project [S1, unofficial]. It is part of a family of neural rendering modules that use MLPs for color prediction. The use of Gaussian activations is a specific design choice to handle view-dependent appearance.

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
