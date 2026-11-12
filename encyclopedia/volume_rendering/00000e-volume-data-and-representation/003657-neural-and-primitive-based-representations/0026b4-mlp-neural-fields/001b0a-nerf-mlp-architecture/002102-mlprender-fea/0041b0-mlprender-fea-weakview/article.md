# MLPRender_Fea_WeakView

Also called: WeakView

## Overview
MLPRender_Fea_WeakView is a neural rendering module that predicts RGB color from input features and view directions using a Multi-Layer Perceptron (MLP) with Gaussian positional encoding applied to features and view directions, but not positions [S1, unofficial]. It builds on the parent module [MLPRender_Fea](2414-mlprender-fea.md) by explicitly processing view directions alongside features through a dedicated encoding and concatenation step [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: `pts` (positions, listed but unused in the forward pass), `viewdirs` (view directions), `features` (input feature vectors) [S1, unofficial].
- Outputs: `rgb` (predicted RGB color values) [S1, unofficial].

**Data Structures and Layout:**
The module is implemented as a neural network with layer dimensions derived from input parameters [S1, unofficial]:
- Parameters: `inChanel` (input feature dimension), `viewpe=6` (view positional encoding degree), `feape=6` (feature positional encoding degree), `featureC=128` (internal feature channel count, noted but not explicitly used in the forward pass formula) [S1, unofficial].
- Internal Channel Calculations: `self.in_mlpC = (2 * feape + 1) * inChanel` and `self.mid_mlpC = (2 * viewpe) * 3` [S1, unofficial].
- Layers: `layer1`, `layer2`, `layer3` (Linear layers), `gaussian` (Gaussian activation function) [S1, unofficial].

**Gaussian Activation Formula:**
The Gaussian activation is defined as [S1, unofficial]:
$$ out = \exp(-0.5 \cdot (x - \mu)^2 / \sigma^2) $$
Where:
- $\sigma = 0.1$ (fixed parameter `opt.arch.gaussian.sigma`) [S1, unofficial].
- $\mu$ is the mean of the linearly transformed input along the last axis [S1, unofficial].

**Algorithm Steps:**
1. Initialize `indata` by applying Gaussian positional encoding to `features`: `indata = fea_gaussian_init(features)` [S1, unofficial].
2. Initialize `middata` by applying Gaussian positional encoding to `viewdirs`: `middata = [view_gaussian_init(viewdirs)]` [S1, unofficial].
3. Pass `indata` through `layer1`: `x = self.layer1(indata)` [S1, unofficial].
4. Apply Gaussian activation to `x`: `x = self.gaussian(x)` [S1, unofficial].
5. Append `x` to `middata` [S1, unofficial].
6. Concatenate all tensors in `middata` along the last dimension: `x = torch.cat(middata, dim=-1)` [S1, unofficial].
7. Pass `x` through `layer2`: `x = self.layer2(x)` [S1, unofficial].
8. Apply Gaussian activation to `x`: `x = self.gaussian(x)` [S1, unofficial].
9. Pass `x` through `layer3` to get raw RGB values: `x = self.layer3(x)` [S1, unofficial].
10. Apply sigmoid activation to `x` to bound RGB values: `rgb = torch.sigmoid(x)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class GaussianActivation(nn.Module):
    def __init__(self, sigma=0.1):
        super().__init__()
        self.sigma = sigma

    def forward(self, x):
        mu = x.mean(dim=-1, keepdim=True)
        return torch.exp(-0.5 * ((x - mu) ** 2) / (self.sigma ** 2))

def fea_gaussian_init(features, feape):
    # [G] Standard positional encoding expansion to match (2*feape+1)*inChanel dimensions
    out = [features]
    for i in range(feape):
        freq = 2 ** i
        out.append(torch.sin(freq * features * 3.14159265))
        out.append(torch.cos(freq * features * 3.14159265))
    return torch.cat(out, dim=-1)

def view_gaussian_init(viewdirs, viewpe):
    # [G] Standard positional encoding expansion to match (2*viewpe)*3 dimensions
    out = []
    for i in range(viewpe):
        freq = 2 ** i
        out.append(torch.sin(freq * viewdirs * 3.14159265))
        out.append(torch.cos(freq * viewdirs * 3.14159265))
    return torch.cat(out, dim=-1)

class MLPRender_Fea_WeakView(nn.Module):
    def __init__(self, inChanel, viewpe=6, feape=6):
        super().__init__()
        self.in_mlpC = (2 * feape + 1) * inChanel
        self.mid_mlpC = (2 * viewpe) * 3
        self.layer1 = nn.Linear(self.in_mlpC, 128)
        self.layer2 = nn.Linear(self.mid_mlpC, 128)
        self.layer3 = nn.Linear(128, 3)
        self.gaussian = GaussianActivation(sigma=0.1)
        for m in self.modules():
            if isinstance(m, nn.Linear):
                nn.init.uniform_(m.weight, -0.1, 0.1)
                if m.bias is not None:
                    nn.init.zeros_(m.bias)

    def forward(self, features, viewdirs):
        indata = fea_gaussian_init(features, feape)
        middata = [view_gaussian_init(viewdirs, viewpe)]
        x = self.layer1(indata)
        x = self.gaussian(x)
        middata.append(x)
        x = torch.cat(middata, dim=-1)
        x = self.layer2(x)
        x = self.gaussian(x)
        x = self.layer3(x)
        rgb = torch.sigmoid(x)
        return rgb
```

**Edge Cases and Pitfalls:**
- The position `pts` is listed as an input parameter but is not used in the color prediction path per the source code logic [S1, unofficial].
- The Gaussian activation depends on the mean of the input, which can be sensitive to input scaling [S1, unofficial].
- Weight initialization range is small (0.1), which may require careful learning rate tuning [S1, unofficial].

## Subtopics

### [MLPRender_Fea_WeakView_Density](10543-mlprender-fea-weakview-density.md)
A neural rendering module that predicts RGB color and volume density ($\sigma$) separately using a feature-based MLP with weak view dependence and Gaussian positional encoding for features and view directions. It sets itself apart from standard NeRF MLPs by decoupling the density prediction from the view-dependent color prediction, allowing the density to be derived from a specific feature channel while maintaining view-independence for the volumetric properties.

## Operating characteristics
- Time and Memory Complexity: Not given in the sources.
- Measured Performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths and Limitations: Not given in the sources.
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
- Origin: Not given in the sources.
- History: Not given in the sources.
- Usage: Not given in the sources.

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
