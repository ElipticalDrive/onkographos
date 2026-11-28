# REF2-NeRF
Also called: Reflection and Refraction aware Neural Radiance Field

## Overview
REF2-NeRF is a NeRF-based modeling method designed for scenes containing transparent objects, specifically glass cases, where it models both refraction and reflection effects [S1]. It distinguishes itself from standard NeRF and reflection-only methods (e.g., NeRFReN, NeuS-HSR) by simultaneously estimating refraction surfaces and separating direct (view-independent) and reflected (view-dependent) light components using two independent MLPs: a Glass network for refraction offsets and a Decomposition NeRF for rendering [S1]. Unlike LB-NeRF, which simplifies refraction as a static offset, REF2-NeRF introduces a view-independent density field and a view-dependent offset field to estimate refraction surfaces and offsets simultaneously, handling multiple glass plates more effectively than explicit-space methods like MS-NeRF [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $n$ images $\{I_k\}$ taken from different viewpoints, with predetermined camera poses (often acquired via robotic arm) [S1].
*   **Outputs:** A rendered image $C(r)$ for a given ray $r$, and estimated fields for glass density, refraction offsets, view-independent color/density, and view-dependent features [S1].

**Data Structures and Network Architecture**
The method uses four main neural network components, all taking positional encoded inputs $\Gamma(\cdot)$ [S1]:
1.  **Glass Network ($\Theta_{gl}$):**
    *   Input: $\Gamma(x), \Gamma(d)$ where $x \in \mathbb{R}^3$ is position, $d \in \mathbb{R}^3$ is view direction.
    *   Output: Glass density $\sigma_{gl} \in \mathbb{R}$ (view-independent) and Offset vector $\Delta x \in \mathbb{R}^3$ (view-dependent) [S1].
    *   Architecture: MLP with input dimension $6 \times 256$ (assuming 256 encoding dim per component, total 512? Source says `[6*256]` in Fig 3 caption, likely meaning 6 channels of encoding or specific layer size. The text says "Input [Γ(x), Γ(d)]". We treat inputs as the encoded vectors) [S1].
2.  **Decomposition NeRF ($\Theta_{nf}$):**
    *   Input: $\Gamma(x'), \Gamma(d)$ where $x'$ is the adjusted position.
    *   Output: View-independent density $\sigma_{vi} \in \mathbb{R}$, View-independent color $c_{vi} \in \mathbb{R}^3$, View-dependent density $\sigma_{vd} \in \mathbb{R}$, and View-dependent feature vector $f_{vd} \in \mathbb{R}^\theta$ (where $\theta=64$) [S1].
    *   Architecture: Two branches. One branch (MLP `[256, 128]`) takes $\Gamma(x')$ to output $\sigma_{vi}, c_{vi}$. The other branch (MLP `[8*256]` input, `[256, 128]` output) takes $\Gamma(x'), \Gamma(d)$ to output $\sigma_{vd}, f_{vd}$ [S1].
3.  **Decoder MLP ($\Theta_{dc}$):**
    *   Input: Rendered feature vector $F_{vd}$.
    *   Output: Reflection color $C_{vd} \in \mathbb{R}^3$ [S1].
4.  **Gate MLP ($\Theta_{gt}$):**
    *   Input: Rendered feature vector $F_{vd}$.
    *   Output: Blending parameter $\alpha \in \mathbb{R}$ [S1].

**Sampling Strategy**
1.  **Coarse Sampling:** 128 points sampled at uniform intervals along the ray [S1].
2.  **Fine Sampling:**
    *   64 points hierarchically sampled based on coarse density (standard NeRF strategy) [S1].
    *   32 points sampled based on glass density $\sigma_{gl}$ [S1].
    *   32 points sampled based on view-independent density $\sigma_{vi}$ [S1].
    *   Total points for fine model: 192 [S1].

**Rendering Pipeline**
1.  **Refraction Offset Calculation:**
    For each sampling point $i$ along ray $r$, calculate the refraction weight $w_i$ using glass density $\sigma_{gl,i}$ and distance $\delta_{gl,i}$ between adjacent points [S1]:
    $$ w_i = T_i (1 - \exp(-\sigma_{gl,i} \delta_{gl,i})) $$
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_{gl,j} \delta_{gl,j}\right) $$
    The adjusted position $x'_i$ is computed by cumulatively adding the weighted offsets:
    $$ x'_i = x_i + \sum_{j=1}^{i} w_j \Delta x_j $$
    This represents the parallel shift of the ray due to refraction [S1].

2.  **View-Independent Rendering:**
    Perform volume rendering on adjusted positions $x'$ using view-independent density $\sigma_{vi}$ and color $c_{vi}$. The view-independent color $C_{vi}(r)$ is:
    $$ C_{vi}(r) = \sum_{i=1}^{N} T_{vi,i} (1 - \exp(-\sigma_{vi,i} \delta_{vi,i})) c_{vi,i} $$
    where $T_{vi,i}$ is the transmittance calculated similarly to Eq. 4 using $\sigma_{vi}$ [S1].

3.  **View-Dependent Feature Rendering:**
    Perform volume rendering on adjusted positions $x'$ and view directions $d$ using view-dependent density $\sigma_{vd}$ and feature vector $f_{vd}$ to get feature vector $F_{vd}$:
    $$ F_{vd}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_{vd,i} \delta_{vd,i})) f_{vd,i} $$
    Here $T_i$ is the transmittance based on $\sigma_{vd}$ [S1].

4.  **View-Dependent Color and Blending:**
    Decode the rendered feature vector $F_{vd}$ into reflection color $C_{vd}$ using decoder MLP $\Theta_{dc}$:
    $$ C_{vd}(r) = F_{\Theta_{dc}}(F_{vd}(r)) $$
    Determine blending parameter $\alpha$ using gate MLP $\Theta_{gt}$:
    $$ \alpha(r) = F_{\Theta_{gt}}(F_{vd}(r)) $$

5.  **Final Image Composition:**
    The final pixel color $C(r)$ is the sum of the view-independent color and the blended view-dependent color:
    $$ C(r) = C_{vi}(r) + \alpha(r) C_{vd}(r) $$
    [S1]

**Training and Loss**
*   **Loss Function:** The total loss $L$ is the sum of the rendering loss $L_{render}$ (L2 distance between rendered color $C(r)$ and input image $\bar{C}(r)$) and a regularization loss $L_{offset}$ [S1]:
    $$ L_{render} = \sum_{r \in R} \|\bar{C}(r) - C(r)\|^2 $$
    $$ L_{offset} = \sum_{x,d} (\Delta x)^2 $$
    $$ L = L_{render} + \epsilon L_{offset} $$
    where $\epsilon = 10^{-5}$ [S1].
*   **Parameters:** Feature vector dimension $\theta = 64$ [S1].
*   **Optimization:** Rays sampled from 1,024 pixels per iteration. 200,000 training iterations [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

def positional_encoding(p, L=10):
    """
    Standard NeRF positional encoding.
    p: tensor of shape (..., 3)
    Returns: tensor of shape (..., 6*L)
    """
    scales = [2**i for i in range(L)]
    scaled_p = p.unsqueeze(-1) * scales
    encoded = torch.cat([torch.sin(scaled_p), torch.cos(scaled_p)], dim=-1)
    return encoded

class GlassNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # Input: 6*L (pos) + 6*L (dir). Output: 1 (sigma_gl) + 3 (delta_x)
        # Architecture hint from Fig 3: [6*256] input layer? 
        # Assuming standard MLP structure based on description.
        self.mlp = nn.Sequential(
            nn.Linear(12 * 256, 256), # Assuming 256 dim encoding
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 4) # 1 for sigma, 3 for offset
        )

    def forward(self, x_enc, d_enc):
        inp = torch.cat([x_enc, d_enc], dim=-1)
        out = self.mlp(inp)
        sigma_gl = out[..., :1]
        delta_x = out[..., 1:]
        return sigma_gl, delta_x

class DecompNeRF(nn.Module):
    def __init__(self, theta=64):
        super().__init__()
        self.theta = theta
        # View-independent branch: Input pos_enc -> sigma_vi, c_vi
        self.vi_branch = nn.Sequential(
            nn.Linear(6 * 256, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, 4) # 1 sigma, 3 color
        )
        # View-dependent branch: Input pos_enc, dir_enc -> sigma_vd, f_vd
        self.vd_branch = nn.Sequential(
            nn.Linear(12 * 256, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, 1 + theta) # 1 sigma, theta features
        )

    def forward(self, x_prime_enc, d_enc):
        # VI
        vi_out = self.vi_branch(x_prime_enc)
        sigma_vi = vi_out[..., :1]
        c_vi = vi_out[..., 1:]
        
        # VD
        vd_inp = torch.cat([x_prime_enc, d_enc], dim=-1)
        vd_out = self.vd_branch(vd_inp)
        sigma_vd = vd_out[..., :1]
        f_vd = vd_out[..., 1:]
        
        return sigma_vi, c_vi, sigma_vd, f_vd

class DecoderMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.mlp = nn.Sequential(
            nn.Linear(64, 128),
            nn.ReLU(),
            nn.Linear(128, 3)
        )
    def forward(self, f_vd):
        return self.mlp(f_vd)

class GateMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.mlp = nn.Sequential(
            nn.Linear(64, 128),
            nn.ReLU(),
            nn.Linear(128, 1)
        )
    def forward(self, f_vd):
        return torch.sigmoid(self.mlp(f_vd))

def volume_render(sigma, color_or_feat, delta, transmittance_prev=None):
    """
    Standard volume rendering.
    sigma: (N,)
    color_or_feat: (N, C)
    delta: (N,) distance between points
    """
    alpha = 1.0 - torch.exp(-sigma * delta)
    T = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha + 1e-10], dim=-1)[..., :-1], dim=-1)
    # T[i] = product(1-alpha) for j < i
    # Corrected T calculation for standard NeRF:
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # Using cumulative sum of sigma*delta
    accumulated = torch.cumsum(sigma * delta, dim=-1)
    T = torch.exp(-torch.cat([torch.zeros_like(accumulated[..., :1]), accumulated[..., :-1]], dim=-1))
    
    weights = alpha * T
    if color_or_feat.dim() == 1:
        return torch.sum(weights * color_or_feat), T
    else:
        return torch.sum(weights.unsqueeze(-1) * color_or_feat, dim=-2), T

def render_ray(x, d, glass_net, decomp_net, decoder, gate, num_coarse=128, num_fine=192):
    """
    x: ray origin (3,)
    d: ray direction (3,)
    near, far: bounds
    """
    # 1. Coarse Sampling
    t_coarse = torch.linspace(0, 1, num_coarse)
    x_coarse = x.unsqueeze(0) + t_coarse.unsqueeze(1) * d.unsqueeze(0) # (1, N, 3)
    
    # 2. Fine Sampling (Hierarchical + Glass/VI specific)
    # In practice, this involves querying densities to resample.
    # For simplicity, we assume x_fine is generated as per source: 
    # 64 hierarchical + 32 glass + 32 VI.
    # We will assume x_fine is provided or generated by a helper not detailed in source.
    # We proceed with x_fine of shape (N_fine, 3)
    # Note: Source says "Total points for fine model: 192".
    # We need to generate these points. 
    # Standard NeRF fine sampling uses coarse density.
    # REF2-NeRF adds glass and VI density sampling.
    # Implementation detail: This part requires a sampling strategy function.
    # We will assume x_fine is available.
    
    # 3. Glass Network Forward Pass
    x_fine_enc = positional_encoding(x_fine)
    d_enc = positional_encoding(d.unsqueeze(0).expand_as(x_fine))
    
    sigma_gl, delta_x = glass_net(x_fine_enc, d_enc)
    
    # 4. Calculate Refraction Offset
    # Calculate weights w_i
    # T_i = exp(-sum_{j=1}^{i-1} sigma_gl_j * delta_gl_j)
    # delta_gl is distance between adjacent points.
    delta_gl = torch.norm(torch.diff(x_fine, dim=-2), dim=-1) # (N-1,)
    # Pad delta_gl to match N
    delta_gl = torch.cat([delta_gl, delta_gl[-1:]], dim=0)
    
    acc_gl = torch.cumsum(sigma_gl * delta_gl, dim=-1)
    T_gl = torch.exp(-torch.cat([torch.zeros_like(acc_gl[..., :1]), acc_gl[..., :-1]], dim=-1))
    w = T_gl * (1.0 - torch.exp(-sigma_gl * delta_gl))
    
    # Cumulative offset
    # x'_i = x_i + sum_{j=1}^i w_j * delta_x_j
    # Note: delta_x is (N, 3). w is (N,).
    weighted_offsets = w.unsqueeze(-1) * delta_x
    cumulative_offset = torch.cumsum(weighted_offsets, dim=-2)
    x_prime = x_fine + cumulative_offset
    
    # 5. Decomp NeRF Forward Pass
    x_prime_enc = positional_encoding(x_prime)
    sigma_vi, c_vi, sigma_vd, f_vd = decomp_net(x_prime_enc, d_enc)
    
    # 6. View-Independent Rendering
    delta_vi = delta_gl # Use same distances
    C_vi, _ = volume_render(sigma_vi, c_vi, delta_vi)
    
    # 7. View-Dependent Feature Rendering
    F_vd, _ = volume_render(sigma_vd, f_vd, delta_gl)
    
    # 8. Decode and Blend
    C_vd = decoder(F_vd)
    alpha = gate(F_vd)
    
    C = C_vi + alpha * C_vd
    
    return C

def train_step(images, poses, glass_net, decomp_net, decoder, gate, optimizer, epsilon=1e-5):
    # Sample rays from 1024 pixels
    rays = sample_rays(images, poses, num_pixels=1024)
    
    # Render
    preds = []
    for r in rays:
        pred = render_ray(r.origin, r.direction, glass_net, decomp_net, decoder, gate)
        preds.append(pred)
    preds = torch.stack(preds)
    
    # Loss
    L_render = torch.mean((preds - images) ** 2)
    
    # Regularization on glass network offsets
    # Need to forward glass net on all points to get delta_x
    # This requires re-running the glass network part or storing it.
    # Assuming we have access to delta_x from render step or re-compute.
    # For simplicity, assume we have delta_x_batch from a forward pass.
    # L_offset = mean(delta_x^2)
    # L_offset = torch.mean(glass_net_outputs_delta_x ** 2) 
    
    L = L_render + epsilon * L_offset
    
    optimizer.zero_grad()
    L.backward()
    optimizer.step()
    
    return L.item()
```

## Operating characteristics
*   **Time Complexity:** Training takes approximately 10 hours per scene on an Nvidia RTX 4080 GPU for 200,000 iterations [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Accurately models both glass refraction and overall scene, separating direct and reflected light components [S1].
*   **Strengths:** Handles multiple transparent objects (e.g., glass showcases) and reciprocal reflections better than explicit-space methods like MS-NeRF [S1]. Simultaneously estimates refraction surfaces and offsets [S1].
*   **Limitations:** Requires predetermined camera poses, which are difficult to acquire in scenes with glass objects (necessitating robotic arms) [S1].
*   **When to use:** Scenes with glass cases or multiple transparent objects where both refraction and reflection need to be modeled [S1].
*   **When not to use:** Scenes without transparent objects or where camera poses cannot be precisely determined [S1].

## Background
REF2-NeRF was proposed to address the challenge of modeling scenes with multiple glass objects, such as objects in a glass showcase, where existing NeRF-based methods struggled with simultaneous reflection and refraction effects [S1]. It builds on LB-NeRF, which simplified refraction as an offset, but improves upon it by introducing view-independent density and view-dependent offset fields [S1]. It also addresses limitations of MS-NeRF, which used multiple explicit spaces for mirrors, by using a single view-dependent feature field to handle complex reciprocal reflections [S1]. The method was implemented based on NeRFPytorch [S1].

## Sources
[S1] REF2-NeRF: Reflection and Refraction aware Neural Radiance Field, https://doi.org/10.1109/iros58592.2024.10801830, fetched 2026-11-08
