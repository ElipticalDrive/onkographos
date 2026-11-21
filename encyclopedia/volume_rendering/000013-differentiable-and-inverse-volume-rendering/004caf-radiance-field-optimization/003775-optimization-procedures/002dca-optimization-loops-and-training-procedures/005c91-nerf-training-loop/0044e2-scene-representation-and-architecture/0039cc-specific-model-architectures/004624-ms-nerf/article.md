# MS-NeRF

Also called: MS-Mip-NeRF, MS-NeRF MLP-based Module, Multi-Space Neural Radiation Field

## Overview
MS-NeRF (Multi-Space Neural Radiance Field) is a scene representation method that extends standard MLP-based NeRF architectures to handle unbounded virtual scenes containing reflective and refractive surfaces [S1, S2]. It solves the problem of rendering distortion and blurring caused by reflection or refraction, where standard NeRF fails to maintain multi-view consistency due to the geometric complexity of virtual images [S2]. Unlike sibling methods like Ref-NeRF, which require manually labeled masks for reflective surfaces and are limited to forward-facing scenes, MS-NeRF automatically handles 360-degree scenes without extra supervision [S1]. It sets itself apart by decomposing the radiance field into $K$ parallel sub-spaces, each representing a characteristic field, and integrating them via a learned gating mechanism [S2].

## How it works
MS-NeRF modifies the canonical NeRF architecture by replacing the single density-color prediction with a multi-space feature prediction [S2]. The method operates by rendering multiple feature maps from parallel sub-spaces and decoding them into RGB images using a gating network [S2].

### Inputs and Outputs
*   **Inputs:** Camera ray parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), scene images (for training supervision) [S1, S2].
*   **Outputs:** Final rendered pixel color $\mathbf{C}(\mathbf{r})$ [S2].

### Data Structures and Layouts
The core data structure is the set of $K$ sub-spaces [S2]. For each sub-space $k \in \{1, \dots, K\}$, the network outputs a density $\sigma_k$ and a feature vector $\mathbf{f}_k$ [S2].
1.  **Sub-space Feature Vector:** $\mathbf{f}_k \in \mathbb{R}^d$, where $d$ is the feature dimension [S1, S2].
2.  **Density:** $\sigma_k \in \mathbb{R}$, scalar density for sub-space $k$ [S2].
3.  **Gate Weights:** $\mathbf{w} \in \mathbb{R}^K$, pixel-level weights for each sub-space [S2].

### Parameters
*   $K$: Number of sub-spaces. Typical values: 2, 4, 6, 8, 16 [S1].
*   $d$: Dimension of output features. Typical values: 24, 48, 64, 128. $d=24$ encodes sufficient information; $d=48$ offers better stability [S1].
*   $h$: Hidden layer dimension of the Decoder MLP and Gate MLP. Typical values: 24, 48, 64, 128 [S1].

### Algorithm Steps

1.  **Positional Encoding:** Encode the 3D position $\mathbf{x}$ along the ray using the standard NeRF positional encoding $\gamma(\mathbf{x})$ [S1, G].
2.  **Multi-Space Feature Extraction:** Pass $\gamma(\mathbf{x})$ through the modified density network [S2]. Instead of outputting a single density and color, the network outputs $K$ pairs of $(\sigma_k, \mathbf{f}_k)$ for $k=1 \dots K$ [S2]. The density network consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer [Parent, S1]. The feature vector $\mathbf{f}_k$ replaces the intermediate feature vector used for color prediction in standard NeRF [S2].
3.  **Volumetric Rendering of Features:** Perform standard volumetric rendering for each sub-space independently to generate a feature map $\mathbf{F}_k$ and an accumulated transparency $T_{\infty, k}$ [S2].
    $$
    \mathbf{F}_k = \sum_{i=1}^{N} T_{i,k} \alpha_{i,k} \mathbf{f}_{i,k}, \quad T_{\infty, k} = \prod_{i=1}^{N} (1 - \alpha_{i,k})
    $$
    where $\alpha_{i,k} = 1 - \exp(-\sigma_{i,k} \delta_i)$ and $T_{i,k} = \prod_{j=1}^{i-1} (1 - \alpha_{j,k})$ [S2].
4.  **Gating Weight Calculation:** Compute the gate weights $\mathbf{w}$ for the current pixel [S2]. This is done by passing the accumulated transparency $T_{\infty, k}$ through a Gate MLP [S2]. Let $G(\cdot)$ be the Gate MLP with hidden dimension $h$ [S1].
    $$
    \mathbf{w} = \text{Softmax}(G(T_{\infty, 1}, \dots, T_{\infty, K}))
    $$
5.  **Decoding and Integration:** For each sub-space, decode the rendered feature map $\mathbf{F}_k$ into an RGB color $\mathbf{C}_k$ using a Decoder MLP $D_k(\cdot)$ with hidden dimension $h$ [S2].
    $$
    \mathbf{C}_k = D_k(\mathbf{F}_k)
    $$
    Finally, integrate the colors from all sub-spaces using the gate weights:
    $$
    \mathbf{C}(\mathbf{r}) = \sum_{k=1}^{K} w_k \mathbf{C}_k
    $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class MSNeRF(nn.Module):
    def __init__(self, K=8, d=48, h=48, L=8, freq_bands=10):
        """
        K: Number of sub-spaces
        d: Feature dimension
        h: Hidden dimension for Decoder and Gate MLPs
        L: Number of hidden layers in the main network
        freq_bands: Number of frequency bands for positional encoding
        """
        super().__init__()
        self.K = K
        self.d = d
        self.freq_bands = freq_bands
        
        # Backbone: 256 hidden layers, ReLU, skip connections from input to every 4th layer [Parent]
        self.backbone_layers = nn.ModuleList()
        self.backbone_layers.append(nn.Linear(2 * freq_bands + 1, 256))
        for i in range(1, L):
            if i % 4 == 0:
                self.backbone_layers.append(nn.Sequential(nn.Linear(256 + 2 * freq_bands + 1, 256), nn.ReLU()))
            else:
                self.backbone_layers.append(nn.Sequential(nn.Linear(256, 256), nn.ReLU()))
        
        # Heads for each sub-space
        self.density_heads = nn.ModuleList([nn.Linear(256, 1) for _ in range(K)])
        self.feature_heads = nn.ModuleList([nn.Linear(256, d) for _ in range(K)])
        
        # Decoder MLPs for each sub-space
        self.decoders = nn.ModuleList([
            nn.Sequential(nn.Linear(d, h), nn.ReLU(), nn.Linear(h, 3)) for _ in range(K)
        ])
        
        # Gate MLP
        self.gate_mlp = nn.Sequential(nn.Linear(K, h), nn.ReLU(), nn.Linear(h, K))

    def forward(self, ray_origins, ray_directions, near, far, num_samples):
        # 1. Sample points along the ray
        t_vals = torch.linspace(0., 1., num_samples, device=ray_origins.device)
        z_vals = near.unsqueeze(1) * (1. - t_vals.unsqueeze(0)) + far.unsqueeze(1) * t_vals.unsqueeze(0) # (B, N)
        points = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * z_vals.unsqueeze(2) # (B, N, 3)
        
        # 2. Positional Encoding
        encoded_points = positional_encoding(points, self.freq_bands) # (B, N, 2*freq+1)
        
        # 3. Forward through backbone with skip connections
        B, N, _ = points.shape
        encoded_flat = encoded_points.view(-1, encoded_points.shape[-1])
        x = encoded_flat
        for layer in self.backbone_layers:
            h = layer(x)
            x = h + x if hasattr(layer, 'in_features') and layer.in_features == x.shape[-1] else h
            # Explicit skip connection handling for every 4th layer as per parent spec
            if len(self.backbone_layers) > 4 and (self.backbone_layers.index(layer) + 1) % 4 == 0:
                x = torch.cat([x, encoded_flat], dim=-1)
                encoded_flat = x # Update for next layer if needed, but standard NeRF skips to input
            x = h # Standard residual flow
        # Simplified backbone flow matching parent: input -> layers with skips to every 4th
        x = encoded_flat
        for i, layer in enumerate(self.backbone_layers):
            h = layer(x)
            if i > 0 and i % 4 == 0:
                x = torch.cat([x, encoded_flat], dim=-1)
            x = h
        
        # 4. Get K densities and K feature vectors
        densities = []
        features = []
        for k in range(self.K):
            sigma_k = self.density_heads[k](x).squeeze(-1) # (B, N)
            f_k = self.feature_heads[k](x) # (B, N, d)
            densities.append(sigma_k)
            features.append(f_k)
            
        # 5. Volumetric Rendering for each sub-space
        rendered_features = []
        transparencies = []
        
        # Compute deltas between samples
        dists = z_vals[:, 1:] - z_vals[:, :-1] # (B, N-1)
        dists = torch.cat([dists, torch.full((B, 1), 1e10, device=z_vals.device)], dim=1) # (B, N)
        
        for k in range(self.K):
            sigma_k = densities[k]
            f_k = features[k]
            
            # Compute alpha
            alpha = 1.0 - torch.exp(-sigma_k.unsqueeze(-1) * dists) # (B, N)
            
            # Transmittance
            T = torch.cumprod(1.0 - alpha + 1e-10, dim=1) # (B, N)
            T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1) # (B, N)
            
            # Weighted sum for feature map
            weights = T * alpha # (B, N)
            F_k = torch.sum(weights.unsqueeze(-1) * f_k, dim=1) # (B, d)
            
            # Accumulated transparency
            T_inf = torch.prod(1.0 - alpha + 1e-10, dim=1) # (B,)
            
            rendered_features.append(F_k)
            transparencies.append(T_inf)
            
        # 6. Gate MLP
        T_stack = torch.stack(transparencies, dim=1) # (B, K)
        gate_weights = F.softmax(self.gate_mlp(T_stack), dim=1) # (B, K)
        
        # 7. Decode and Integrate
        final_colors = []
        for k in range(self.K):
            C_k = self.decoders[k](rendered_features[k]) # (B, 3)
            final_colors.append(C_k)
            
        final_colors = torch.stack(final_colors, dim=1) # (B, K, 3)
        
        # Weighted sum
        output = torch.sum(gate_weights.unsqueeze(-1) * final_colors, dim=1) # (B, 3)
        
        return output

def positional_encoding(x, freq_bands=10):
    """
    Standard NeRF positional encoding matching gamma(x) = (sin(2^0 pi x), cos(2^0 pi x), ...) [Parent]
    x: (..., 3)
    Returns: (..., 2*freq_bands + 1)
    """
    x = torch.as_tensor(x, dtype=torch.float32)
    scales = 2.0 ** torch.arange(freq_bands, device=x.device).float() * math.pi
    xb = x[..., None, :] * scales[None, None, :] # (..., 3, freq)
    xb = xb.reshape(*x.shape[:-1], 3 * freq_bands)
    pe = torch.cat([torch.sin(xb), torch.cos(xb)], dim=-1)
    pe = torch.cat([x, pe], dim=-1) # Concatenate original x
    return pe
```

### Edge Cases and Pitfalls
*   **Precision:** The accumulation of transparencies and weights can suffer from numerical instability if not handled with care (e.g., using `1e-10` epsilon) [G].
*   **Boundary Conditions:** The method assumes unbounded scenes [S1].
*   **Degenerate Input:** If the scene has no reflective surfaces, the extra sub-spaces may learn redundant features, adding computational cost without benefit. The ablation study shows minimal parameter increase (0.5%) but non-zero overhead [S1].

## Subtopics

### MS-Mip-NeRF
Extends the MS module to the Mip-NeRF baseline [S1]. Sets itself apart by enabling unbounded scene modeling with minimal parameter overhead [S1]. Choose when rendering unbounded scenes with reflections [S1]. [Link](8585-specific-model-architectures.md)

### MS-Mip-NeRF 360
Integrates the MS module with Mip-NeRF 360 [S1]. Sets itself apart by achieving 4.15 dB PSNR improvement on 360-degree scenes with mirrors without masks [S1]. Choose for 360-degree reflective scenes [S1]. [Link](8585-specific-model-architectures.md)

### MS-TensoRF
Hybrid grid-based integration of the MS module [S1]. Sets itself apart by combining multi-space decomposition with grid-based representations for efficiency [S1]. Choose when balancing reflective surface handling with memory constraints [S1]. [Link](8585-specific-model-architectures.md)

## Operating characteristics
*   **Time Complexity:** The method adds $K$ parallel rendering passes and decoding steps [S2]. The computational overhead is proportional to $K$ [S2]. However, since the backbone is shared, the increase is less than $K$ times the original cost [S1].
*   **Memory Complexity:** Memory usage increases by $K$ times the feature storage during rendering and the parameters of $K$ decoders and one gate MLP [S2]. The parameter increase is reported as 0.5% on Mip-NeRF 360 (9.052M vs 9.007M) [S1].
*   **Performance:** On synthetic, real, and RFFR datasets, PSNR is 2-4 points higher than baselines [S2]. On Mip-NeRF 360, it achieves 4.15 dB PSNR improvement [S1].
*   **Strengths:** Handles 360-degree scenes with mirrors automatically; no manual masks required; robust to reflective and refractive surfaces [S1, S2].
*   **Limitations:** Slightly higher computational cost due to multiple sub-spaces; requires tuning of $K, d, h$ [S1].
*   **When to use:** Scenes with reflective/refractive surfaces, especially unbounded 360-degree scenes [S1, S2].
*   **When not to use:** Simple forward-facing scenes without reflections, where standard NeRF or Mip-NeRF suffices [S1].

## Background
MS-NeRF was introduced to address the limitations of standard NeRF in handling reflective and refractive surfaces, which cause rendering distortion and blurring due to the lack of multi-view consistency in such scenarios [S2]. The method was developed as an extension of MLP-based NeRF methods, including NeRF, Mip-NeRF, and Mip-NeRF 360 [S1]. It was proposed to handle unbounded virtual scenes and was shown to outperform Ref-NeRF, which requires manually labeled masks and fails on 360-degree scenes [S1]. The method is part of the broader family of radiance-field optimization procedures that aim to improve novel view synthesis quality [S2].

## Sources
[S1] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
[S2] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
