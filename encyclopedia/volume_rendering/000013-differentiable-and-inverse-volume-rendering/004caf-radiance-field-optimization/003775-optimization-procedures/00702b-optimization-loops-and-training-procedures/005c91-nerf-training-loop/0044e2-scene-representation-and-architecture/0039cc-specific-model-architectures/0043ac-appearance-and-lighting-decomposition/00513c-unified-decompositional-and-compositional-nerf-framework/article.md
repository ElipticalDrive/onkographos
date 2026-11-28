# Unified Decompositional and Compositional NeRF Framework

Also called: Unified Decompositional and Compositional NeRF, Joint Framework for Editable Novel View Synthesis

## Overview
This method solves the problem of disentangling foreground objects from background environments in a radiance field to enable editable novel view synthesis, where foreground objects can be removed or replaced without corrupting the background [S1]. It sets itself apart from sibling methods like [Ref-NeRF](10843-ref-nerf.md) by explicitly modeling separate background and object radiance fields that are composited via a differentiable one-hot gating mechanism, rather than relying solely on view-dependent lighting models or anisotropic scattering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera poses, ray directions $\mathbf{d}$, and 3D coordinates $\mathbf{x}$ sampled along rays [S1].
*   **Output:** Composited pixel color $C(\mathbf{r})$ and decomposed background/object radiance fields [S1].

**Data Structures:**
The network consists of a shared backbone and three specific heads [S1].
1.  **Shared Backbone:** A 4-layer Multi-Layer Perceptron (MLP) [S1].
    *   Input: Positional encoded coordinates $\gamma(\mathbf{x})$ [S1].
    *   Output: Generic implicit scene feature vector $\mathbf{f}_{scene} \in \mathbb{R}^{256}$ [S1].
2.  **Heads:** Three separate 3-layer MLPs (Background Head, Object Head, Guidance Head) [S1].
    *   Each head takes $\mathbf{f}_{scene}$ as input [S1].
    *   Each head outputs density $\sigma$ and color $\mathbf{c}$ for its respective field [S1].
3.  **Positional Encoding:**
    *   Coordinates $\mathbf{x}$: 10 frequency levels [S1].
    *   Directions $\mathbf{d}$: 4 frequency levels [S1].

**Numbered Steps:**

1.  **Feature Extraction:**
    For a query point $\mathbf{x}$ and direction $\mathbf{d}$, compute positional encodings $\gamma(\mathbf{x})$ and $\gamma(\mathbf{d})$ [S1]. Pass $\gamma(\mathbf{x})$ through the Shared Backbone to obtain the latent feature $\mathbf{f}_{scene}$:
    $$
    \mathbf{f}_{scene} = \text{Backbone}(\gamma(\mathbf{x}))
    $$
    [S1]

2.  **Radiance Field Prediction:**
    Pass $\mathbf{f}_{scene}$ through the three heads to predict parameters for the background ($b$), object ($o$), and guidance ($g$) fields:
    $$
    (\sigma_b, \mathbf{c}_b) = \text{Head}_b(\mathbf{f}_{scene})
    $$
    $$
    (\sigma_o, \mathbf{c}_o) = \text{Head}_o(\mathbf{f}_{scene})
    $$
    $$
    (\sigma_g, \mathbf{c}_g) = \text{Head}_g(\mathbf{f}_{scene})
    $$
    [S1]

3.  **One-Hot Density Regularization (Background):**
    To constrain uncertain regions in the background, apply a regularization term that encourages the background density to be sparse or one-hot in nature, preventing background leakage into object regions [S1].

4.  **Gumbel-Softmax Composition (Coarse Stage):**
    To composite the background and object fields, compute one-hot weights using Gumbel-Softmax activation [S1]. Let $w_b$ and $w_o$ be the weights for background and object.
    $$
    w_b, w_o = \text{GumbelSoftmax}(\text{logits}(\sigma_b, \sigma_o))
    $$
    The composited density $\sigma$ and color $\mathbf{c}$ for the coarse stage are:
    $$
    \sigma = w_b \sigma_b + w_o \sigma_o
    $$
    $$
    \mathbf{c} = w_b \mathbf{c}_b + w_o \mathbf{c}_o
    $$
    [S1]
    *Note: Directly adding $\sigma_b$ and $\sigma_o$ without these weights causes the background to collapse to black [S1].*

5.  **Residual Radiance Composition (Fine Stage):**
    Use the Guidance Head's radiance field ($\sigma_g, \mathbf{c}_g$) to provide coarse sampling guidance [S1]. In the fine stage, the guidance field is merged as residual information [S1]. The exact residual merge formula is not given in the sources.

6.  **2D In-painting Pseudo Supervision:**
    Apply pseudo-supervision signals to the background radiance field [S1]. This involves masking out foreground objects in the input images and using the remaining pixels to supervise the background head, ensuring the background can extrapolate correctly into regions occupied by foreground objects [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SharedBackbone(nn.Module):
    def __init__(self):
        super().__init__()
        # 4-layer MLP, output dim 256 [S1]
        self.layers = nn.Sequential(
            nn.Linear(20, 256), # 10 levels * 2 (sin/cos)
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256)
        )

    def forward(self, x):
        return self.layers(x)

class Head(nn.Module):
    def __init__(self):
        super().__init__()
        # 3-layer MLP ending with color and density [S1]
        self.layers = nn.Sequential(
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 3 + 1) # 3 for color, 1 for density
        )

    def forward(self, f_scene):
        out = self.layers(f_scene)
        c = torch.sigmoid(out[:, :3])
        sigma = torch.relu(out[:, 3:])
        return sigma, c

def positional_encoding(x, levels=10):
    # x shape: [N, 3]
    # Output shape: [N, 2*levels] [S1]
    freq_bands = 2.0 ** torch.linspace(0, levels - 1, levels).to(x.device)
    x = x.unsqueeze(-1) * freq_bands
    x = torch.cat([torch.sin(x), torch.cos(x)], dim=-1)
    return x

def gumbel_softmax(logits, tau=1.0, hard=False):
    # Standard Gumbel-Softmax implementation [G]
    # logits: [N, 2] for background and object
    gumbels = (-torch.empty_like(logits).exponential_().log() + 
               torch.log(torch.empty_like(logits).exponential_() + 1e-10))
    gumbels = (logits + gumbels) / tau
    y_soft = F.softmax(gumbels, dim=-1)
    if hard:
        index = y_soft.max(dim=-1, keepdim=True)[1]
        y_hard = torch.zeros_like(logits).scatter_(-1, index, 1.0)
        ret = y_hard - y_soft.detach() + y_soft
    else:
        ret = y_soft
    return ret

class UnifiedNeRF(nn.Module):
    def __init__(self):
        super().__init__()
        self.backbone = SharedBackbone()
        self.head_bg = Head()
        self.head_obj = Head()
        self.head_guid = Head()
        
    def forward(self, x, d):
        # x: [N, 3], d: [N, 3]
        # 1. Positional Encoding: 10 levels for coords, 4 for directions [S1]
        gamma_x = positional_encoding(x, levels=10)
        gamma_d = positional_encoding(d, levels=4)
        
        # 2. Shared Backbone takes gamma(x) [S1]
        f_scene = self.backbone(gamma_x)
        
        # 3. Heads take f_scene [S1]
        sigma_bg, c_bg = self.head_bg(f_scene)
        sigma_obj, c_obj = self.head_obj(f_scene)
        sigma_guid, c_guid = self.head_guid(f_scene)
        
        # 4. Gumbel-Softmax Composition (Coarse Stage) [S1]
        logits = torch.stack([sigma_bg, sigma_obj], dim=-1)
        weights = gumbel_softmax(logits, tau=1.0) # [N, 2]
        
        w_bg = weights[:, 0].unsqueeze(-1)
        w_obj = weights[:, 1].unsqueeze(-1)
        
        sigma_comp = w_bg * sigma_bg + w_obj * sigma_obj
        c_comp = w_bg * c_bg + w_obj * c_obj
        
        # 5. Residual Composition (Fine Stage) [S1]
        # The guidance field is used for coarse sampling and merged as residual in fine stage.
        # Exact merge formula not given in sources.
        sigma_fine = sigma_comp + sigma_guid
        c_fine = c_comp + c_guid
        
        return {
            'sigma': sigma_comp,
            'c': c_comp,
            'sigma_fine': sigma_fine,
            'c_fine': c_fine,
            'sigma_bg': sigma_bg,
            'c_bg': c_bg,
            'sigma_obj': sigma_obj,
            'c_obj': c_obj,
            'sigma_guid': sigma_guid,
            'c_guid': c_guid
        }
```

**Parameters:**
*   **Backbone Layers:** 4 [S1].
*   **Head Layers:** 3 per head [S1].
*   **Feature Dimension:** 256 [S1].
*   **Positional Encoding Levels:** 10 for coordinates, 4 for directions [S1].
*   **Training:** End-to-end from scratch, no voxel or geometric initialization [S1].

**Edge Cases and Pitfalls:**
*   **Background Collapse:** Directly adding local object/background radiance fields without Gumbel-Softmax weights causes the background radiance field to collapse to all black [S1].
*   **Extrapolation Failure:** Disabling 2D in-painting pseudo supervision results in significantly downgraded extrapolation capability in 3D regions occupied by foreground objects [S1].
*   **Quality Degradation:** Removing residual radiance composition degrades both overall qualitative metrics and background decomposition quality [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the ScanNet dataset, the method achieves PSNR 26.1349, SSIM 0.8249, and LPIPS 0.3951 [S1]. On the ToyDesk dataset, it achieves PSNR 23.1435, SSIM 0.4711, and LPIPS 0.4691 [S1].
*   **Strengths:** Enables explicit editing of foreground objects; disentangles background from foreground effectively [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method was introduced in "Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis" [S1]. The framework is trained end-to-end from scratch, explicitly avoiding voxel feature initialization and geometric initialization [S1]. It is evaluated on the ScanNet and ToyDesk datasets against ObjectNeRF and ObjectSDF [S1].

## Sources
[S1] Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis, https://arxiv.org/abs/2308.02840v1, fetched 2026-10-26
