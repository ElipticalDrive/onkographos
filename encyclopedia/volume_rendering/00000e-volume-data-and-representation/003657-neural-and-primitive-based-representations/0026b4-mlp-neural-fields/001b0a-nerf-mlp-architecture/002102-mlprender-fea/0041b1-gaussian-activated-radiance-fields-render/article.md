# Gaussian Activated Radiance Fields Render
## Overview
Gaussian Activated Radiance Fields Render (GaRF_Render) is a neural rendering module that predicts RGB color from input features and view directions using a Multi-Layer Perceptron (MLP) with Gaussian activation applied to the feature and view direction branches [S1, unofficial]. It builds on [MLPRender_Fea](2414-mlprender-fea.md) by applying Gaussian kernels to the encoded inputs before the MLP, distinguishing it from sibling architectures that use standard activations or linear projections [S1, unofficial].
## How it works
**Inputs:** Input feature vector of size `inChanel` [S1, unofficial]. View direction vector of size 3 [S1, unofficial].
**Outputs:** RGB color vector of size 3 [S1, unofficial].
**Data Structures:** The module consists of five `torch.nn.Linear` layers: `fea_gaussian_linear` (maps `inChanel` to `featureC`), `view_gaussian_linear` (maps 3 to `featureC`), `layer1` (maps `featureC` to `featureC`), `layer2` (maps `2 * featureC` to `featureC`), and `layer3` (maps `featureC` to 3) [S1, unofficial].
**Parameters:** `inChanel`: Input feature channel size [S1, unofficial]. `viewpe`: Number of view positional encoding frequencies, default 6 [S1, unofficial]. `feape`: Number of feature positional encoding frequencies, default 6 [S1, unofficial]. `featureC`: Hidden layer channel size, default 128 [S1, unofficial]. `mid_mlpC`: Fixed to 3 [S1, unofficial].
**Initialization:** All weight matrices for the linear layers are initialized using a uniform distribution [S1, unofficial]. Bias terms use PyTorch's default initialization [G].
**Algorithm:**
1. Apply positional encoding to the input features if `feape > 0`, producing $F_{enc}$ [G].
2. Apply positional encoding to the view direction if `viewpe > 0`, producing $V_{enc}$ [G].
3. Pass $F_{enc}$ through `fea_gaussian_linear` and apply a Gaussian activation function to obtain $G_f$ [G].
4. Pass $V_{enc}$ through `view_gaussian_linear` and apply a Gaussian activation function to obtain $G_v$ [G].
5. Pass $G_f$ through `layer1` to obtain $H_1$ [G].
6. Concatenate $H_1$ and $G_v$ along the channel dimension to form $C_{12}$ [G].
7. Pass $C_{12}$ through `layer2` to obtain $H_2$ [G].
8. Pass $H_2$ through `layer3` to output the RGB color vector [G].

**Pseudocode:**
```python
import torch
import math

def gaussian_activation(x):
    """Standard Gaussian activation: exp(-0.5 * x^2) [G]."""
    return torch.exp(-0.5 * x ** 2)

def positional_encoding(x, num_frequencies):
    """Standard NeRF positional encoding [G]."""
    if num_frequencies == 0:
        return x
    freq_bands = torch.linspace(1, num_frequencies, num_frequencies, device=x.device).reshape(1, -1, 1)
    x = x.unsqueeze(-1)
    out = torch.cat([torch.sin(x * math.pi * freq_bands), torch.cos(x * math.pi * freq_bands)], dim=-1)
    return out.reshape(*x.shape[:-1], -1)

class GaRF_Render(torch.nn.Module):
    def __init__(self, inChanel, viewpe=6, feape=6, featureC=128):
        super(GaRF_Render, self).__init__()
        self.in_mlpC = inChanel
        self.mid_mlpC = 3
        self.viewpe = viewpe
        self.feape = feape
        self.featureC = featureC
        self.fea_gaussian_linear = torch.nn.Linear(self.in_mlpC, featureC) [S1, unofficial]
        self.view_gaussian_linear = torch.nn.Linear(self.mid_mlpC, featureC) [S1, unofficial]
        self.layer1 = torch.nn.Linear(featureC, featureC) [S1, unofficial]
        self.layer2 = torch.nn.Linear(featureC + featureC, featureC) [S1, unofficial]
        self.layer3 = torch.nn.Linear(featureC, 3) [S1, unofficial]
        self._uniform_init_weights()

    def _uniform_init_weights(self):
        """Uniform initialization for all layer weights [S1, unofficial]."""
        for m in [self.fea_gaussian_linear, self.view_gaussian_linear, self.layer1, self.layer2, self.layer3]:
            torch.nn.init.uniform_(m.weight)

    def forward(self, features, view_dirs):
        # Forward pass structure inferred from layer dimensions [G]
        features_enc = positional_encoding(features, self.feape)
        view_dirs_enc = positional_encoding(view_dirs, self.viewpe)

        # Gaussian activation on linear projections [G]
        fea_gaussian = gaussian_activation(self.fea_gaussian_linear(features_enc))
        view_gaussian = gaussian_activation(self.view_gaussian_linear(view_dirs_enc))

        # MLP stages inferred from layer dimensions [G]
        h1 = self.layer1(fea_gaussian)
        h2 = self.layer2(torch.cat([h1, view_gaussian], dim=-1))
        rgb = self.layer3(h2)
        return rgb
## Operating characteristics
- Time complexity: Linear with respect to the number of input features and the hidden dimension `featureC` [G].
- Memory complexity: Proportional to the number of parameters in the five linear layers [G].
- Measured performance: Not given in the sources [S1, unofficial].
- Scaling: Scales linearly with input feature dimension and hidden width [G].
- Strengths: Gaussian activations can capture sharp view-dependent variations without saturating like ReLU [G].
- Limitations: The exact forward-pass concatenation order and intermediate activations are not specified in the source code, requiring implementation inference [G].
- When to use: Neural rendering pipelines requiring explicit Gaussian modeling of feature/view interactions [G].
- When not to use: When standard MLP activations are sufficient or strict source parity is required [G].
## Background
The method originates from the Joint-TensoRF project, which provides the `GaRF_Render` module as a Gaussian Random Field rendering component [S1, unofficial]. It extends the [MLPRender_Fea](2414-mlprender-fea.md) architecture by introducing Gaussian activation functions to the feature and view direction branches before the color prediction MLP [S1, unofficial].
## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
