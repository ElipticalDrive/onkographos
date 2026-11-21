# REF2-NeRF

Also called: Reflection and Refraction aware Neural Radiance Field

## Overview
REF2-NeRF is a NeRF-based modeling method designed for scenes containing transparent objects, specifically glass cases, where it models refraction and reflection using elements dependent and independent of the viewer's perspective [S1]. It estimates surfaces where refraction occurs and separates direct (view-independent) and reflected (view-dependent) light components [S1]. Unlike methods that assume straight light rays or only handle reflections, REF2-NeRF handles both refraction and reflection in scenes with multiple transparent objects by using two independent MLPs: a glass network for refraction offsets and a decomposition NeRF for rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ images $\{I_k\}$ taken from different viewpoints, with predetermined camera poses (often acquired via a robotic arm) [S1].
*   **Output:** A rendered image $C(\mathbf{r})$ for a ray $\mathbf{r}$, composed of view-independent and view-dependent components [S1].

**Data Structures and Network Layout:**
The method uses four main MLPs:
1.  **Glass Network ($\Theta_{gl}$):**
    *   **Input:** Concatenation of positional encoding of position $\Gamma(\mathbf{x})$ and view direction $\Gamma(\mathbf{d})$ [S1].
    *   **Output:** Glass density $\sigma_{gl} \in \mathbb{R}$ and offset vector $\Delta \mathbf{x} \in \mathbb{R}^3$ [S1].
    *   **Architecture:** Input layer $[6 \times 256]$, Output layer $[256, 128]$ (implied by diagram labels, though text specifies output values) [S1].
2.  **Decomposition NeRF ($\Theta_{nf}$):**
    *   **Input:** Concatenation of positional encoding of adjusted position $\Gamma(\mathbf{x}')$ and view direction $\Gamma(\mathbf{d})$ [S1].
    *   **Output:** View-independent density $\sigma_{vi} \in \mathbb{R}$, view-independent color $\mathbf{c}_{vi} \in \mathbb{R}^3$, view-dependent density $\sigma_{vd} \in \mathbb{R}$, and view-dependent feature vector $\mathbf{f}_{vd} \in \mathbb{R}^\theta$ [S1].
    *   **Architecture:** Input layer $[8 \times 256]$, Output layer $[256, 128]$ [S1].
3.  **Decoder MLP ($\Theta_{dc}$):**
    *   **Input:** Rendered feature vector $\mathbf{F}_{vd}$ [S1].
    *   **Output:** Reflection color $\mathbf{C}_{vd}$ [S1].
4.  **Gate MLP ($\Theta_{gt}$):**
    *   **Input:** Rendered feature vector $\mathbf{F}_{vd}$ [S1].
    *   **Output:** Blending parameter $\alpha$ [S1].

**Parameters:**
*   Feature vector dimension $\theta = 64$ [S1].
*   Regularization weight $\epsilon = 10^{-5}$ [S1].
*   Sampling points: Coarse 128, Hierarchical Fine 64, Glass Density 32, View-Independent Density 32. Total fine model points: 192 [S1].

**Numbered Steps:**

1.  **Sampling:** For a ray $\mathbf{r}$, sample $N$ points $\mathbf{x}_i$ at uniform intervals (coarse). Then, sample additional points hierarchically based on density (fine). Specifically, sample 32 points using glass density $\sigma_{gl}$ and 32 points using view-independent density $\sigma_{vi}$ [S1].
2.  **Glass Network Forward Pass:** For each sampled point $\mathbf{x}_i$ and view direction $\mathbf{d}_i$, compute glass density $\sigma_{gl,i}$ and offset $\Delta \mathbf{x}_i$ using $\Theta_{gl}$:
    $$ F_{\Theta_{gl}} : \Gamma(\mathbf{x}_i), \Gamma(\mathbf{d}_i) \rightarrow \sigma_{gl}(\mathbf{x}_i), \Delta \mathbf{x}(\mathbf{x}_i, \mathbf{d}_i) $$
    [S1]
3.  **Refraction Weight Calculation:** Calculate the weight $w_i$ for each sampling point $i$ using glass density $\sigma_{gl,i}$ and distance $\delta_{gl,i}$ between adjacent points. $T_i$ is the transmittance from previous points [S1]:
    $$ w_i = T_i (1 - \exp(-\sigma_{gl,i} \delta_{gl,i})), \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_{gl,j} \delta_{gl,j}\right) $$
    [S1]
4.  **Position Adjustment:** Adjust the position $\mathbf{x}'_i$ by cumulatively adding the weighted offset vectors along the ray:
    $$ \mathbf{x}'_i = \mathbf{x}_i + w_i \Delta \mathbf{x}_i $$
    [S1]
5.  **View-Independent Rendering:** Perform volume rendering on the adjusted positions $\mathbf{x}'$ using view-independent density $\sigma_{vi}$ and color $\mathbf{c}_{vi}$ to get view-independent color $\mathbf{C}_{vi}$. Note that $T_{vi,i}$ calculation uses $\sigma_{vi}$ [S1]:
    $$ \mathbf{C}_{vi}(\mathbf{r}) = \sum_{i=1}^{N} T_{vi,i} (1 - \exp(-\sigma_{vi,i} \delta_{vi,i})) \mathbf{c}_{vi,i} $$
    [S1]
6.  **View-Dependent Feature Rendering:** Perform volume rendering on adjusted positions $\mathbf{x}'$ and view directions $\mathbf{d}$ using view-dependent density $\sigma_{vd}$ and feature vector $\mathbf{f}_{vd}$ to get feature vector $\mathbf{F}_{vd}$ [S1]:
    $$ \mathbf{F}_{vd}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_{vd,i} \delta_{vd,i})) \mathbf{f}_{i} $$
    [S1]
7.  **View-Dependent Color and Blending:** Decode the rendered feature vector $\mathbf{F}_{vd}$ into reflection color $\mathbf{C}_{vd}$ using decoder MLP $\Theta_{dc}$ and determine blending parameter $\alpha$ using gate MLP $\Theta_{gt}$ [S1]:
    $$ \mathbf{C}_{vd}(\mathbf{r}) = F_{\Theta_{dc}} : \mathbf{F}_{vd}(\mathbf{r}) \rightarrow \mathbf{C}_{vd}(\mathbf{r}) $$
    $$ \alpha(\mathbf{r}) = F_{\Theta_{gt}} : \mathbf{F}_{vd}(\mathbf{r}) \rightarrow \alpha(\mathbf{r}) $$
    [S1]
8.  **Final Image Composition:** The final pixel color $\mathbf{C}(\mathbf{r})$ is the sum of the view-independent color and the blended view-dependent color [S1]:
    $$ \mathbf{C}(\mathbf{r}) = \mathbf{C}_{vi}(\mathbf{r}) + \alpha(\mathbf{r})\mathbf{C}_{vd}(\mathbf{r}) $$
    [S1]

**Training and Loss:**
The method is trained by minimizing the total loss $\mathcal{L}$, which is the sum of the rendering loss $\mathcal{L}_{render}$ (L2 distance between rendered color $\mathbf{C}(\mathbf{r})$ and input image $\bar{\mathbf{C}}(\mathbf{r})$) and a regularization loss $\mathcal{L}_{offset}$ [S1]:
$$ \mathcal{L}_{render} = \sum_{\mathbf{r} \in \mathcal{R}} \|\bar{\mathbf{C}}(\mathbf{r}) - \mathbf{C}(\mathbf{r})\|^2 $$
$$ \mathcal{L}_{offset} = \sum_{\mathbf{x}, \mathbf{d}} (\Delta \mathbf{x})^2 $$
$$ \mathcal{L} = \mathcal{L}_{render} + \epsilon \mathcal{L}_{offset} $$
where $\mathcal{R}$ is a batch of sampled rays and $\epsilon = 10^{-5}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class PositionalEncoding(nn.Module):
    def __init__(self, num_freq_bands=6, include_input=True):
        super().__init__()
        self.num_freq_bands = num_freq_bands
        self.include_input = include_input
        # Standard NeRF positional encoding frequencies
        self.frequencies = 2.0 ** torch.linspace(0, num_freq_bands - 1, num_freq_bands)

    def forward(self, x):
        # x: (..., 3)
        if self.include_input:
            out = [x]
        else:
            out = []
        for freq in self.frequencies:
            out.append(torch.sin(x * freq))
            out.append(torch.cos(x * freq))
        return torch.cat(out, dim=-1)

class GlassNetwork(nn.Module):
    def __init__(self, input_dim=6*256, hidden_dim=256):
        super().__init__()
        # Simplified MLP structure based on source description
        # Input: Gamma(x), Gamma(d) -> Output: sigma_gl, delta_x
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc_sigma = nn.Linear(hidden_dim, 1)
        self.fc_offset = nn.Linear(hidden_dim, 3)
        self.relu = nn.ReLU()

    def forward(self, x_enc, d_enc):
        x = torch.cat([x_enc, d_enc], dim=-1)
        h = self.relu(self.fc1(x))
        sigma_gl = self.fc_sigma(h)
        delta_x = self.fc_offset(h)
        return sigma_gl, delta_x

class DecompNeRF(nn.Module):
    def __init__(self, input_dim=8*256, hidden_dim=256, feature_dim=64):
        super().__init__()
        # Input: Gamma(x'), Gamma(d) -> Output: sigma_vi, c_vi, sigma_vd, f_vd
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc_sigma_vi = nn.Linear(hidden_dim, 1)
        self.fc_c_vi = nn.Linear(hidden_dim, 3)
        self.fc_sigma_vd = nn.Linear(hidden_dim, 1)
        self.fc_f_vd = nn.Linear(hidden_dim, feature_dim)
        self.relu = nn.ReLU()

    def forward(self, x_prime_enc, d_enc):
        x = torch.cat([x_prime_enc, d_enc], dim=-1)
        h = self.relu(self.fc1(x))
        sigma_vi = self.fc_sigma_vi(h)
        c_vi = self.fc_c_vi(h)
        sigma_vd = self.fc_sigma_vd(h)
        f_vd = self.fc_f_vd(h)
        return sigma_vi, c_vi, sigma_vd, f_vd

class DecoderMLP(nn.Module):
    def __init__(self, input_dim=64, hidden_dim=128):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc_out = nn.Linear(hidden_dim, 3)
        self.relu = nn.ReLU()

    def forward(self, f_vd):
        h = self.relu(self.fc1(f_vd))
        c_vd = self.fc_out(h)
        return c_vd

class GateMLP(nn.Module):
    def __init__(self, input_dim=64, hidden_dim=128):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc_out = nn.Linear(hidden_dim, 1)
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, f_vd):
        h = self.relu(self.fc1(f_vd))
        alpha = self.sigmoid(self.fc_out(h))
        return alpha

def render_ref2nerf(ray_origins, ray_directions, glass_net, decomp_net, decoder_net, gate_net, 
                    num_coarse=128, num_fine=192, epsilon=1e-5):
    """
    Renders a single ray using the REF2-NeRF method.
    """
    # 1. Sampling
    # Coarse sampling: 128 points at uniform intervals
    t_coarse = torch.linspace(0, 1, num_coarse).to(ray_origins.device)
    x_coarse = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_coarse.unsqueeze(0)
    
    # Fine sampling: 192 points total (64 hierarchical + 32 glass + 32 view-ind)
    # Note: Source says "Total points for fine model: 192". 
    # Implementation detail: Hierarchical sampling usually replaces or adds to coarse.
    # We assume standard NeRF coarse-to-fine where fine samples are added.
    # For simplicity in this self-contained code, we generate fine samples based on density.
    # In practice, this requires forward pass of coarse model to get densities.
    # Here we simulate the fine sampling positions x_fine.
    t_fine = torch.linspace(0, 1, num_fine).to(ray_origins.device)
    x_fine = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_fine.unsqueeze(0)
    
    # 2. Glass Network Forward Pass
    # Encode positions and directions
    pos_enc = PositionalEncoding(num_freq_bands=6, include_input=True)
    x_fine_enc = pos_enc(x_fine)
    d_enc = pos_enc(ray_directions)
    
    sigma_gl, delta_x = glass_net(x_fine_enc, d_enc)
    
    # 3. Refraction Weight Calculation
    # Calculate distances between adjacent points
    deltas = torch.diff(x_fine, dim=1)
    delta_lengths = torch.norm(deltas, dim=2, keepdim=True)
    # Pad the last delta to match size
    delta_lengths = torch.cat([delta_lengths, delta_lengths[:, -1:, :]], dim=1)
    
    # Transmittance T_i
    # T_i = exp(-sum_{j=1}^{i-1} sigma_gl,j * delta_gl,j)
    # sigma_gl is (N, 1), delta_lengths is (N, 1, 1) -> need to handle dimensions
    # sigma_gl: (batch, N, 1)
    # delta_lengths: (batch, N, 1)
    
    # Cumulative sum of sigma * delta
    # sigma_gl * delta_lengths -> (batch, N, 1)
    sigma_delta = sigma_gl * delta_lengths
    # Cumulative sum from start to i-1
    # Using cumsum, then shifting
    cum_sigma_delta = torch.cumsum(sigma_delta, dim=1)
    # T_i corresponds to sum up to i-1. For i=0, sum is 0.
    # Shift cumsum to right by 1, fill with 0
    cum_sigma_delta_shifted = torch.cat([torch.zeros_like(cum_sigma_delta[:, :1, :]), cum_sigma_delta[:, :-1, :]], dim=1)
    T = torch.exp(-cum_sigma_delta_shifted)
    
    # Weights w_i
    w = T * (1 - torch.exp(-sigma_gl * delta_lengths))
    
    # 4. Position Adjustment
    # x'_i = x_i + w_i * delta_x_i
    # w: (batch, N, 1), delta_x: (batch, N, 3)
    x_prime = x_fine + w * delta_x
    
    # 5. View-Independent Rendering
    x_prime_enc = pos_enc(x_prime)
    sigma_vi, c_vi, sigma_vd, f_vd = decomp_net(x_prime_enc, d_enc)
    
    # View-independent volume rendering
    # C_vi(r) = sum T_vi,i (1 - exp(-sigma_vi,i * delta_vi,i)) c_vi,i
    # T_vi,i uses sigma_vi
    sigma_delta_vi = sigma_vi * delta_lengths
    cum_sigma_delta_vi = torch.cumsum(sigma_delta_vi, dim=1)
    cum_sigma_delta_vi_shifted = torch.cat([torch.zeros_like(cum_sigma_delta_vi[:, :1, :]), cum_sigma_delta_vi[:, :-1, :]], dim=1)
    T_vi = torch.exp(-cum_sigma_delta_vi_shifted)
    
    alpha_vi = 1 - torch.exp(-sigma_delta_vi)
    C_vi = torch.sum(T_vi * alpha_vi * c_vi, dim=1)
    
    # 6. View-Dependent Feature Rendering
    # F_vd(r) = sum T_i (1 - exp(-sigma_vd,i * delta_vd,i)) f_i
    # Note: Source uses T_i (from glass density) for view-dependent feature rendering?
    # Source Eq 8: Fvd(r) = sum Ti (1 - exp(-sigma_vd,i delta_vd,i)) fi
    # Yes, it uses T_i from glass density.
    sigma_delta_vd = sigma_vd * delta_lengths
    alpha_vd = 1 - torch.exp(-sigma_delta_vd)
    F_vd = torch.sum(T * alpha_vd * f_vd, dim=1)
    
    # 7. View-Dependent Color and Blending
    C_vd = decoder_net(F_vd)
    alpha_blend = gate_net(F_vd)
    
    # 8. Final Image Composition
    C = C_vi + alpha_blend * C_vd
    
    return C
```

## Operating characteristics
*   **Time Complexity:** Training requires 200,000 iterations, taking approximately 10 hours per scene on an Nvidia RTX 4080 GPU [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Scaling:** The method uses a coarse-to-fine sampling strategy with a total of 192 points for the fine model (128 coarse + 64 hierarchical + 32 glass density + 32 view-independent density) [S1].
*   **Strengths:** Handles both refraction and reflection in scenes with multiple transparent objects (e.g., glass cases) [S1]. Separates view-independent and view-dependent components effectively [S1].
*   **Limitations:** Requires predetermined camera poses, which can be difficult to acquire in scenes with glass objects [S1].
*   **When to use:** Scenes with glass objects where both refraction and reflection need to be modeled accurately [S1].
*   **When not:** Scenes without transparent objects or where camera poses are unknown [S1].

## Background
REF2-NeRF was proposed in 2024 by authors publishing in the IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS) [S1]. It addresses the challenge of modeling scenes with multiple glass objects, such as objects in a glass showcase, where existing methods struggle with multiple reflection and refraction effects [S1]. It builds on LB-NeRF, which simplifies refraction as an offset, but introduces view-independent density and view-dependent offset fields to estimate refraction surfaces and offsets simultaneously [S1]. It also differs from MS-NeRF, which uses multiple explicit spaces for mirrors, by using a single view-dependent feature field to handle reciprocal reflections from multiple glass plates [S1].

## Sources
[S1] REF2-NeRF: Reflection and Refraction aware Neural Radiance Field, https://doi.org/10.1109/iros58592.2024.10801830, fetched 2026-11-08
