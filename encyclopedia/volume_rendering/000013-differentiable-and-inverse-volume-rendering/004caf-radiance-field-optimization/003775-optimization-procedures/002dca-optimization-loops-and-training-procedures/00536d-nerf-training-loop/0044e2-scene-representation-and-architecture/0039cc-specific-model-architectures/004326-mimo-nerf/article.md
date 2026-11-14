# MIMO-NeRF

Also called: MIMO-NeRF-distill, MIMO-NeRF-naive

## Overview
MIMO-NeRF is a neural rendering method that accelerates inference by replacing the standard Single-Input Single-Output (SISO) Multi-Layer Perceptron (MLP) with a Multi-Input Multi-Output (MIMO) MLP [S1]. It groups neighboring samples along a ray and processes them simultaneously, reducing the number of MLP evaluations (# Run) required per pixel [S1]. Unlike knowledge distillation methods that require training a separate teacher network, MIMO-NeRF can be trained via self-supervised learning using a 3D consistency loss to resolve ambiguities in density and color inherent in grouped predictions [S1]. It maintains the original NeRF hyperparameters (sample counts and network width) while achieving superior speed-quality trade-offs compared to methods that reduce sample counts or network width [S1].

## How it works
**Inputs and Outputs:**
The method takes a set of camera views and their poses as input [S1]. For each ray $\mathbf{r}$, it samples $N_c$ coarse points and $N_f$ fine points [S1]. The MIMO MLP $f_{\text{MIMO}}$ takes a group of $N_p$ positions $\{\mathbf{x}_i, \dots, \mathbf{x}_j\}$ and the ray direction $\mathbf{d}$ as input, and outputs $N_p$ colors $\{\mathbf{c}_i, \dots, \mathbf{c}_j\}$ and densities $\{\sigma_i, \dots, \sigma_j\}$ [S1].

**Data Structures:**
The network architecture is identical to the baseline NeRF [Parent] except for input/output dimensions [S1].
*   **Positional Encoding:** Input positions $\mathbf{x} \in \mathbb{R}^3$ and directions $\mathbf{d} \in \mathbb{S}^2$ are encoded using positional encoding $\gamma(\cdot)$ [Parent, S1]. $\gamma(\mathbf{x})$ is 63-dimensional, $\gamma(\mathbf{d})$ is 27-dimensional [S1].
*   **MIMO MLP:**
    *   **Density Network:** Takes concatenated encoded positions $\gamma(\mathbf{x}_i), \dots, \gamma(\mathbf{x}_j)$ (total dimension $63 \times N_p$) and applies an 8-layer MLP with ReLU activations, 256 hidden units, and skip connections from input to the 5th layer [S1]. Output is $N_p$ density values $\sigma$ [S1].
    *   **Color Network:** Takes the output of the last hidden layer of the density network (dimension 256), concatenates with encoded directions $\gamma(\mathbf{d})$ (dimension 27), passes through a 1-layer MLP (ReLU, 128 units), and finally a linear layer to produce $N_p$ RGB colors $\mathbf{c}$ [S1].
    *   Note: The total parameter count increases because the input dimension of the first layer scales with $N_p$, but the hidden layer size remains constant [S1].

**Numbered Steps:**
1.  **Grouping:** For a ray with $N$ total samples (coarse + fine), group $N_p$ neighboring samples starting from the camera. Create $N/N_p$ groups [S1].
2.  **Forward Pass:** Apply $f_{\text{MIMO}}$ to each group. For group $m$ containing samples $i$ through $j$ ($j=i+N_p-1$), compute:
    $$ f_{\text{MIMO}} : (\mathbb{R}^3)^{N_p} \times \mathbb{S}^2 \rightarrow (\mathbb{R}^3 \times \mathbb{R}^+)^{N_p} $$
    $$ (\mathbf{x}_i, \dots, \mathbf{x}_j, \mathbf{d}) \mapsto (\mathbf{c}_i, \dots, \mathbf{c}_j, \sigma_i, \dots, \sigma_j) $$
    [S1]
3.  **Rendering:** Compute pixel color $C(\mathbf{r})$ using standard volumetric rendering integration with the predicted $\sigma$ and $\mathbf{c}$ [S1].
4.  **Training Objective:** Minimize $\mathcal{L}_{\text{MIMO}} = \mathcal{L}_{\text{pixel}}^{\text{MIMO}} + \lambda \mathcal{L}_{\text{3D}}$ [S1].
    *   $\mathcal{L}_{\text{pixel}}^{\text{MIMO}}$: Standard MSE between rendered and ground truth pixel colors [S1].
    *   $\mathcal{L}_{\text{3D}}$: 3D consistency loss comprising color and alpha consistency [S1].
5.  **3D Consistency Loss:** To address ambiguity in grouped predictions, enforce consistency between different reformulated MLPs (or different groups/views) [S1].
    *   Color 3D Loss:
        $$ \mathcal{L}_{\text{color 3D}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\mathbf{c}_i^{m_1} - \text{sg}(\mathbf{c}_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\mathbf{c}_i^{m_1}) - \mathbf{c}_i^{m_2}\|_2^2 \right] $$
    *   Alpha 3D Loss:
        $$ \mathcal{L}_{\text{alpha 3D}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\alpha_i^{m_1} - \text{sg}(\alpha_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\alpha_i^{m_1}) - \alpha_i^{m_2}\|_2^2 \right] $$
    *   Asymmetric Weight $\mu$:
        $$ \mu_{mj}^{mi} = \frac{\sqrt{m_j} \sqrt{R_{\max}}}{\sqrt{R_{m_i}}} $$
        where $R_{\max} = \max_{m \in \{1, \dots, M\}} R_m$. $R_m$ are scaling factors for reformulated MLPs [S1].
6.  **Optimization:** Train using Adam optimizer with learning rate $5 \times 10^{-4}$ and batch size 1024 rays [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def positional_encoding(x, L_freq=10):
    """
    x: (..., 3)
    Returns: (..., 63)
    """
    x = x.unsqueeze(-2)
    freq_bands = 2.0 ** torch.linspace(0, L_freq-1, L_freq).to(x.device)
    freq_bands = freq_bands.view(1, 1, 1, -1)
    scaled_x = x * freq_bands * torch.pi
    pe = torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)
    pe = pe.view(*x.shape[:-2], -1)
    return torch.cat([x.squeeze(-2), pe], dim=-1)

class MIMO_NeRF(nn.Module):
    def __init__(self, Np, F=256, L=8):
        super().__init__()
        self.Np = Np
        # Density network: 63*Np -> F -> ... -> F -> Np
        self.density_layers = nn.ModuleList()
        self.density_layers.append(nn.Linear(63 * Np, F))
        for _ in range(L - 2):
            self.density_layers.append(nn.Linear(F, F))
        self.density_layers.append(nn.Linear(F, Np))
        
        # Color network: F + 27 -> 128 -> 3*Np
        self.color_layers = nn.ModuleList()
        self.color_layers.append(nn.Linear(F + 27, 128))
        self.color_layers.append(nn.Linear(128, 3 * Np))
        
    def forward(self, x_grouped, d):
        """
        x_grouped: (B, Np, 3) Batch of groups of positions
        d: (B, 3) View direction for the ray
        Returns:
        colors: (B, Np, 3)
        densities: (B, Np, 1)
        """
        B, Np, _ = x_grouped.shape
        assert Np == self.Np
        
        gamma_x = positional_encoding(x_grouped) # (B, Np, 63)
        x_flat = gamma_x.view(B, -1) # (B, 63*Np)
        
        h = x_flat
        for i, layer in enumerate(self.density_layers[:-1]):
            h = layer(h)
            if i == 3: # Skip connection to 5th layer (index 4)
                h = h + x_flat
            h = F.relu(h)
            
        densities = self.density_layers[-1](h).view(B, Np, 1) # (B, Np, 1)
        
        gamma_d = positional_encoding(d.unsqueeze(1)).squeeze(1) # (B, 27)
        color_input = torch.cat([h, gamma_d], dim=-1) # (B, F+27)
        
        c = self.color_layers[0](color_input) # (B, 128)
        c = F.relu(c)
        c = self.color_layers[1](c).view(B, Np, 3) # (B, Np, 3)
        
        return c, densities

def compute_loss(colors, densities, gt_colors, alpha, lambda_3d=1.0):
    l_pixel = F.mse_loss(colors, gt_colors)
    # 3D consistency loss requires multiple reformulated MLPs (M > 1).
    # The sources do not give a single-network implementation for L_3D.
    # It is computed between predictions from different reformulations as per the formulas.
    l_3d = 0.0 # Placeholder for multi-reformulation consistency logic
    return l_pixel + lambda_3d * l_3d
```

**Parameters:**
*   $N_p$: Number of grouped samples. Typical values: 2, 4, 8 [S1].
*   $N_c, N_f$: Coarse and fine samples. Blender: $N_c=64, N_f=128$. LLFF: $N_c=64, N_f=64$ [S1].
*   $\lambda$: Weight for 3D consistency loss. Blender: 1. LLFF: 0.4 [S1].
*   $R_m$: Scaling factors for asymmetric weighting. $N_p=2 \rightarrow R=[1,1]$; $N_p=4 \rightarrow R=[1,2]$; $N_p=8 \rightarrow R=[1,2,4]$ [S1].
*   Optimizer: Adam, lr $5 \times 10^{-4}$, batch size 1024 rays [S1].

**Edge Cases and Pitfalls:**
*   **Ambiguity:** Pixel-wise loss alone leads to ambiguity in color/density for individual points in a group. Solved by 3D consistency loss [S1].
*   **Asymmetric Weights:** When $N_p=2$, $R_1=R_2=1$, making asymmetric weights unity (disabled) [S1].
*   **Parameter Increase:** Parameter count increases by $1.06\times$ ($N_p=2$), $1.17\times$ ($N_p=4$), $1.39\times$ ($N_p=8$) due to expanded positional encoding input dimension [S1].

## Subtopics

### MIMO-TensoRF
Application of MIMO-NeRF to the TensoRF architecture [S1]. It improves inference and training speed without deteriorating image quality by selecting appropriate $N_p$ [S1].
*   **Parameters:** $N_p$ (Blender: $\le 2$, LLFF: $\le 4$); Tensor ranks $R_\sigma, R_c$ (e.g., $R_\sigma=16, R_c=48$) [S1].
*   **Performance:** Improves I-time and T-time of original TensoRF with similar quality [S1].
*   **Implementation:** Based on official TensoRF source code [S1].

## Operating characteristics
*   **Time Complexity:** Inference cost is measured by # Run (number of MLP evaluations). # Run $= N_c/N_p + (N_c+N_f)/N_p$. For $N_c=64, N_f=128, N_p=2$, # Run is 128 (vs 192 for NeRF) [S1].
*   **Memory Complexity:** Parameter count increases slightly (up to $1.39\times$ for $N_p=8$) [S1].
*   **Performance:**
    *   **Blender ($N_p=2$):** PSNR 31.26, SSIM 0.953, LPIPS 0.054, I-time 5.15s, T-time 5.36h [S1].
    *   **LLFF ($N_p=2$):** PSNR 27.70, SSIM 0.870, LPIPS 0.155, I-time 4.55s, T-time 3.97h [S1].
    *   **FLOPs:** MIMO-NeRF ($N_p=8$) reduces FLOPs to 52.72M (Blender) vs 303.82M for NeRF [S1].
*   **Strengths:** Superior speed-quality trade-off compared to NeRF-few and NeRF-small (which reduce samples or width to match FLOPs) [S1]. Self-supervised variant outperforms distillation variants [S1].
*   **Limitations:** Trade-off between speed and quality as $N_p$ increases [S1].

## Background
MIMO-NeRF was introduced in 2023 [S1]. It addresses the high computational cost of standard NeRF by leveraging parallel processing of ray samples [S1]. It builds on the NeRF training loop [Parent] and positional encoding techniques [Parent].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
