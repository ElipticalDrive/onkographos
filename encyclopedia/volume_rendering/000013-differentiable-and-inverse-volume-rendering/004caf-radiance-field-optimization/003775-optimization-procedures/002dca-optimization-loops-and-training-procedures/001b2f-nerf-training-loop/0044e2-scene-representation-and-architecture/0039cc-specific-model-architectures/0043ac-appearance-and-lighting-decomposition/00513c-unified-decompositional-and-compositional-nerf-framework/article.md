# Unified Decompositional and Compositional NeRF Framework

Also called: Unified Decompositional and Compositional NeRF, Joint Framework for Editable Novel View Synthesis

## Overview
This method solves the problem of disentangling foreground objects from background environments in a radiance field to enable editable novel view synthesis, where foreground objects can be removed or replaced without corrupting the background [S1]. It sets itself apart from sibling methods like [Ref-NeRF](10843-ref-nerf.md) by explicitly modeling separate background and object radiance fields that are composited via a differentiable one-hot gating mechanism, rather than relying solely on view-dependent lighting models or anisotropic scattering [S1]. Unlike [Convex Combination Color Decoder](11505-convex-combination-color-decoder.md) which focuses on color mixing, this framework enforces spatial separation through density regularization and Gumbel-Softmax activation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera poses, ray directions $\mathbf{d}$, and 3D coordinates $\mathbf{x}$ sampled along rays.
*   **Output:** Composited pixel color $C(\mathbf{r})$ and decomposed background/object radiance fields.

**Data Structures:**
The network consists of a shared backbone and three specific heads.
1.  **Shared Backbone:** A 4-layer Multi-Layer Perceptron (MLP).
    *   Input: Positional encoded coordinates $\gamma(\mathbf{x})$.
    *   Output: Generic implicit scene feature vector $\mathbf{f}_{scene} \in \mathbb{R}^{256}$ [S1].
2.  **Heads:** Three separate 3-layer MLPs (Background Head, Object Head, Guidance Head).
    *   Each head takes $\mathbf{f}_{scene}$ as input.
    *   Each head outputs density $\sigma$ and color $\mathbf{c}$ for its respective field [S1].
3.  **Positional Encoding:**
    *   Coordinates $\mathbf{x}$: 10 frequency levels.
    *   Directions $\mathbf{d}$: 4 frequency levels [S1].

**Numbered Steps:**

1.  **Feature Extraction:**
    For a query point $\mathbf{x}$ and direction $\mathbf{d}$, compute positional encodings $\gamma(\mathbf{x})$ and $\gamma(\mathbf{d})$. Pass $\gamma(\mathbf{x})$ through the Shared Backbone to obtain the latent feature $\mathbf{f}_{scene}$:
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
    To constrain uncertain regions in the background, apply a regularization term that encourages the background density to be sparse or one-hot in nature, preventing background leakage into object regions. This is implemented via a specific loss term on $\sigma_b$ [S1].

4.  **Gumbel-Softmax Composition (Coarse Stage):**
    To composite the background and object fields, compute one-hot weights using Gumbel-Softmax activation. Let $w_b$ and $w_o$ be the weights for background and object.
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
    Use the Guidance Head's radiance field ($\sigma_g, \mathbf{c}_g$) to provide coarse sampling guidance. In the fine stage, the guidance field is merged as residual information. The final color is computed by combining the composited field from step 4 with the residual from the guidance field [S1].

6.  **2D In-painting Pseudo Supervision:**
    Apply pseudo-supervision signals to the background radiance field. This involves masking out foreground objects in the input images and using the remaining pixels to supervise the background head, ensuring the background can extrapolate correctly into regions occupied by foreground objects [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SharedBackbone(nn.Module):
    def __init__(self):
        super().__init__()
        # 4-layer MLP, output dim 256
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
        # 3-layer MLP ending with color and density
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
    # Output shape: [N, 2*levels]
    freq_bands = 2.0 ** torch.linspace(0, levels - 1, levels).to(x.device)
    x = x.unsqueeze(-1) * freq_bands
    x = torch.cat([torch.sin(x), torch.cos(x)], dim=-1)
    return x

def gumbel_softmax(logits, tau=1.0, hard=False):
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
        # 1. Positional Encoding
        # 10 levels for coords, 4 for directions (directions not used in backbone input per source, 
        # but source says "ten levels for coordinates and four levels for ray directions" for representation ability.
        # Standard NeRF uses both. Source implies backbone takes coords. 
        # We assume standard PE for x. d is typically used in color network, 
        # but here heads take f_scene. We follow source: backbone takes gamma(x).
        gamma_x = positional_encoding(x, levels=10)
        
        # 2. Shared Backbone
        f_scene = self.backbone(gamma_x)
        
        # 3. Heads
        sigma_bg, c_bg = self.head_bg(f_scene)
        sigma_obj, c_obj = self.head_obj(f_scene)
        sigma_guid, c_guid = self.head_guid(f_scene)
        
        # 4. Gumbel-Softmax Composition
        # Combine densities to get logits for gating
        # Source implies using density or a learned logit. 
        # We use the predicted densities as logits for the gating mechanism.
        logits = torch.stack([sigma_bg, sigma_obj], dim=-1)
        weights = gumbel_softmax(logits, tau=1.0) # [N, 2]
        
        w_bg = weights[:, 0].unsqueeze(-1)
        w_obj = weights[:, 1].unsqueeze(-1)
        
        sigma_comp = w_bg * sigma_bg + w_obj * sigma_obj
        c_comp = w_bg * c_bg + w_obj * c_obj
        
        # 5. Residual Composition (Fine Stage Logic)
        # Source: "guidance radiance field for fine sampling and merges it as residual information"
        # In a single forward pass, we return the coarse composition and the guidance field.
        # The renderer handles the sampling.
        return {
            'sigma': sigma_comp,
            'c': c_comp,
            'sigma_guid': sigma_guid,
            'c_guid': c_guid,
            'sigma_bg': sigma_bg,
            'c_bg': c_bg,
            'sigma_obj': sigma_obj,
            'c_obj': c_obj
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
*   **Time Complexity:** Depends on the number of samples per ray and the depth of the MLPs (4-layer backbone + 3-layer heads). The Gumbel-Softmax operation is negligible.
*   **Memory Complexity:** Stores parameters for one backbone and three heads. No voxel grid storage is used (unlike [Parent](11061-appearance-and-lighting-decomposition.md) variants that might use voxels).
*   **Strengths:** Enables explicit editing of foreground objects; disentangles background from foreground effectively.
*   **Limitations:** Requires careful tuning of the Gumbel-Softmax temperature and regularization weights; sensitive to the in-painting supervision mask quality.
*   **When to use:** When the goal is to separate and edit specific objects within a scene.
*   **When not to use:** When only novel view synthesis without editing is required, simpler NeRF variants may suffice.

## Background
The method was introduced in "Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis" [S1]. It addresses limitations in previous methods like ObjectNeRF [37] and ObjectSDF [35] by providing a unified framework that jointly optimizes background and object fields. The approach avoids the need for explicit geometric initialization or voxel feature initialization, training entirely from scratch [S1].

## Sources
[S1] Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis, https://arxiv.org/abs/2308.02840v1, fetched 2026-10-26
