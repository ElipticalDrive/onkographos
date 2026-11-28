# Tri-Plane Positional Encoding

Also called: PET-NeuS, PET-NeuS derivation, Positional Encoding Tri-Planes, Positional Encoding Tri-Planes (PET) Derivation, Tri-plane positional encoding expansion

## Overview
Tri-Plane Positional Encoding (PET) is a method for neural surface reconstruction that integrates frequency inductive biases into tri-plane representations to suppress high-frequency noise caused by the discrete nature of grid-based features [S1]. Unlike standard tri-planes which lack positional constraints, PET expands the 3D implicit function into a sum of trigonometric terms multiplied by 2D plane features, allowing an MLP to learn the coefficients directly [S1]. It sets itself apart from siblings like [Fourier Feature Networks](2765-fourier-feature-networks.md) by operating on 2D planar projections rather than 3D coordinates, and from [Off-Axis Positional Encoding](3503-off-axis-positional-encoding.md) by using orthogonal projections with specific windowed convolutions to avoid depth bleeding.

## How it works
**Inputs and Outputs:**
- **Input:** 3D coordinates $(x, y, z)$ normalized to a bounding box of size $L=3.0$ [S1].
- **Output:** A scalar value representing the Signed Distance Function (SDF) or color, computed by an MLP [S1].

**Data Structures:**
The representation consists of three tri-planes $T_{xy}, T_{yz}, T_{xz}$ [S1].
- Each plane has spatial resolution $R \times R$ (default $512 \times 512$) [S1].
- Each plane has $n_f$ feature channels (default $n_f = 24$) [S1].
- The final input to the MLP is a flattened vector formed by concatenating modulated features from 4 frequency bands (original + 3 window sizes) [S1].

**Mathematical Derivation:**
The method derives from the Fourier series expansion of a 3D function $f(x, y, z)$ [S1]:
$$
f(x, y, z) = \sum_{m=-M}^{M} \sum_{n=-N}^{N} \sum_{k=-K}^{K} a_{mnk} \Theta_{xm} \Theta_{yn} \Theta_{zk}
$$
where $\Theta_{vt}$ is an auxiliary variable defined as [S1]:
$$
\Theta_{vt} = \begin{cases} \cos(tv) & t > 0 \\ 1 & t=0 \\ \sin(tv) & t < 0 \end{cases}
$$
The expansion is grouped into terms involving $\cos(mx)$, $\sin(mx)$, and residual terms. By defining 2D plane features as coefficients of these trigonometric functions, the 3D function is rewritten as [S1]:
$$
f(x, y, z) = \text{MLP}(\text{flatten}([g_m(y,z)\cos(mx), g'_m(y,z)\sin(mx), h_n(x,z)\cos(ny), h'_n(x,z)\sin(ny), w_k(x,y)\cos(kz), w'_k(x,y)\sin(kz)]))
$$
where $g, h, w$ are the tri-plane features [S1].

**Algorithm Steps:**
1. **Initialize Tri-Planes:** Create three planes $T_{xy}, T_{yz}, T_{xz}$ of size $R \times R \times n_f$ [S1].
2. **Generate Multi-Frequency Features:** Apply Self-Attention Convolution (SAC) with non-overlapping patches of window sizes $W \in \{4, 8, 16\}$ to each tri-plane to generate features for different frequency bands [S1]. Let $T^i$ be the output of the $i$-th SAC convolution ($i=0$ for original, $i=1,2,3$ for windows) [S1].
3. **Concatenate Bands:** Form the final tri-plane representation $T$ by concatenating the features from all bands along the channel dimension [S1]:
   $$ T = \text{concat}[T^0, T^1, T^2, T^3] $$
   where $T^i = [T^i_{xy}, T^i_{yz}, T^i_{xz}]$ [S1].
4. **Modulate with Positional Encoding:** For a query point $(x, y, z)$, compute the positional encoding terms $\cos(mx), \sin(mx)$, etc., for $m \in \{1, \dots, M\}$, $n \in \{1, \dots, N\}$, $k \in \{1, \dots, K\}$ [S1].
5. **Sample and Multiply:** Sample the tri-plane features at $(x,y), (y,z), (x,z)$ and multiply them element-wise with the corresponding trigonometric terms [S1].
6. **MLP Prediction:** Flatten the resulting vector and pass it through a 3-layer MLP to predict the SDF and color [S1].

**Parameters:**
- Number of positional encoding scales: $M=N=K=8$ [S1].
- Feature channels per tri-plane: $n_f = 24$ [S1].
- Window sizes for SAC: 4, 8, 16 [S1].
- Scene normalization: Bounding box size $L=3.0$ [S1].
- Optimizer: Adam with learning rate $5 \times 10^{-4}$ [S1].
- Eikonal loss weight: $\lambda = 0.1$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class TriPlanePositionalEncoding:
    def __init__(self, R=512, n_f=24, M=8, N=8, K=8, window_sizes=[4, 8, 16]):
        self.R = R
        self.n_f = n_f
        self.M = M
        self.N = N
        self.K = K
        self.window_sizes = window_sizes
        
        # Initialize three tri-planes
        self.T_xy = nn.Parameter(torch.randn(1, n_f, R, R))
        self.T_yz = nn.Parameter(torch.randn(1, n_f, R, R))
        self.T_xz = nn.Parameter(torch.randn(1, n_f, R, R))
        
        # Self-Attention Convolution layers for different window sizes
        # Note: SAC implementation details are abstracted here as per source [S1]
        self.sac_layers = nn.ModuleList([
            SelfAttentionConv(n_f, window_size) for window_size in window_sizes
        ])
        
        # MLP for SDF and Color
        self.mlp = nn.Sequential(
            nn.Linear(6 * (M + N + K) * (len(window_sizes) + 1) * n_f, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 2) # SDF and Color
        )

    def forward(self, xyz):
        """
        xyz: Tensor of shape (N, 3) containing normalized coordinates (x, y, z)
        """
        x, y, z = xyz[:, 0], xyz[:, 1], xyz[:, 2]
        
        # 1. Generate multi-frequency features using SAC
        # T_base is the concatenation of original and SAC features
        T_base_xy = self.T_xy
        T_base_yz = self.T_yz
        T_base_xz = self.T_xz
        
        T_all_xy = [T_base_xy]
        T_all_yz = [T_base_yz]
        T_all_xz = [T_base_xz]
        
        for sac_layer in self.sac_layers:
            T_all_xy.append(sac_layer(T_base_xy))
            T_all_yz.append(sac_layer(T_base_yz))
            T_all_xz.append(sac_layer(T_base_xz))
            
        # Concatenate along channel dimension
        # Shape: (1, n_f * (1 + len(window_sizes)), R, R)
        T_xy_final = torch.cat(T_all_xy, dim=1)
        T_yz_final = torch.cat(T_all_yz, dim=1)
        T_xz_final = torch.cat(T_all_xz, dim=1)
        
        # 2. Sample features at query points
        # Bilinear interpolation
        # xy coordinates for T_xy are (x, y)
        # yz coordinates for T_yz are (y, z)
        # xz coordinates for T_xz are (x, z)
        
        # Normalize coordinates to [-1, 1] for grid_sample if not already
        # Assuming xyz is already in [-1, 1] or normalized appropriately
        
        # Sample T_xy at (x, y)
        # grid_sample expects (N, C, H, W) and grid of (N, H_out, W_out, 2)
        # We need to reshape xyz for sampling
        xy_grid = torch.stack([x, y], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, N, 2)
        features_xy = F.grid_sample(T_xy_final, xy_grid, align_corners=True).squeeze() # (C, N)
        
        yz_grid = torch.stack([y, z], dim=-1).unsqueeze(0).unsqueeze(0)
        features_yz = F.grid_sample(T_yz_final, yz_grid, align_corners=True).squeeze()
        
        xz_grid = torch.stack([x, z], dim=-1).unsqueeze(0).unsqueeze(0)
        features_xz = F.grid_sample(T_xz_final, xz_grid, align_corners=True).squeeze()
        
        # 3. Modulate with positional encoding
        # Flatten features to (N, C)
        features_xy = features_xy.permute(1, 0) # (N, C)
        features_yz = features_yz.permute(1, 0)
        features_xz = features_xz.permute(1, 0)
        
        C = features_xy.shape[1]
        num_windows = len(self.window_sizes) + 1
        n_f_per_window = self.n_f
        
        input_vector = []
        
        # X-axis terms: cos(mx), sin(mx) modulated by T_yz and T_xz features
        # The source indicates g_m(y,z) and h_n(x,z) etc.
        # We iterate over m, n, k and window indices
        
        for w_idx in range(num_windows):
            # Features for this window
            feat_xy_w = features_xy[:, w_idx*n_f_per_window : (w_idx+1)*n_f_per_window]
            feat_yz_w = features_yz[:, w_idx*n_f_per_window : (w_idx+1)*n_f_per_window]
            feat_xz_w = features_xz[:, w_idx*n_f_per_window : (w_idx+1)*n_f_per_window]
            
            for m in range(1, self.M + 1):
                cos_mx = torch.cos(m * x)
                sin_mx = torch.sin(m * x)
                # Modulate yz and xz features with x-frequency terms
                # g_m(y,z) corresponds to features on yz plane
                # h_n(x,z) corresponds to features on xz plane
                # The source formula groups them. We flatten the modulation.
                term_g_cos = feat_yz_w * cos_mx.unsqueeze(1)
                term_g_sin = feat_yz_w * sin_mx.unsqueeze(1)
                input_vector.append(term_g_cos.flatten(1))
                input_vector.append(term_g_sin.flatten(1))
                
            for n in range(1, self.N + 1):
                cos_ny = torch.cos(n * y)
                sin_ny = torch.sin(n * y)
                term_h_cos = feat_xz_w * cos_ny.unsqueeze(1)
                term_h_sin = feat_xz_w * sin_ny.unsqueeze(1)
                input_vector.append(term_h_cos.flatten(1))
                input_vector.append(term_h_sin.flatten(1))
                
            for k in range(1, self.K + 1):
                cos_kz = torch.cos(k * z)
                sin_kz = torch.sin(k * z)
                term_w_cos = feat_xy_w * cos_kz.unsqueeze(1)
                term_w_sin = feat_xy_w * sin_kz.unsqueeze(1)
                input_vector.append(term_w_cos.flatten(1))
                input_vector.append(term_w_sin.flatten(1))
                
        # Concatenate all terms
        final_input = torch.cat(input_vector, dim=1)
        
        # 4. MLP Prediction
        output = self.mlp(final_input)
        return output

class SelfAttentionConv(nn.Module):
    def __init__(self, channels, window_size):
        super().__init__()
        self.window_size = window_size
        # Simplified SAC: Just a placeholder for the operation described in [S1]
        # In practice, this involves dividing the plane into non-overlapping patches
        # and applying self-attention within each patch.
        self.conv = nn.Conv2d(channels, channels, kernel_size=1)
        
    def forward(self, x):
        # x: (1, C, R, R)
        # Apply self-attention convolution with window_size
        # This is a complex operation not fully detailed in pseudocode form in source
        # We assume a standard convolution as a proxy for the feature mixing
        # [G] The source specifies non-overlapping patches and self-attention.
        # A full implementation requires patching and attention mechanism.
        # Here we use a simple conv to represent the feature transformation.
        return self.conv(x)
```

**Edge Cases and Pitfalls:**
- **Depth Bleeding:** Standard Gaussian kernel smoothing on tri-planes causes features to bleed across depth discontinuities (e.g., foreground/background) because planes are orthogonal projections [S1]. This method avoids that by using Self-Attention Convolution with non-overlapping patches [S1].
- **Precision:** The method relies on normalized coordinates $L=3.0$ [S1].

## Operating characteristics
- **Time Complexity:** Training time on DTU dataset is 9 hours on a single NVIDIA TITAN V100 [S1]. This is faster than VolSDF (12h), NeuS (16h), and HF-NeuS (20h) [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Mesh extraction time is 20 seconds for 512 grid resolution. Rendering time is ~100 seconds for a 1600x1200 resolution image [S1].
- **Strengths:** Faster training than competitors; suppresses high-frequency noise in grid-based features; avoids depth bleeding associated with Gaussian smoothing [S1].
- **Limitations:** Requires careful tuning of window sizes for SAC; relies on MLP capacity to approximate the error caused by finite truncation of Fourier series [S1].
- **When to use:** When high-frequency details are needed in neural surface reconstruction and standard tri-planes exhibit noise or depth bleeding [S1].
- **When not to use:** When computational resources are extremely limited and training time is critical, though it is faster than some alternatives [S1].

## Background
Tri-Plane Positional Encoding was introduced in the paper "PET-NeuS: Positional Encoding Tri-Planes for Neural Surfaces" [S1]. It builds on the concept of tri-plane representations and integrates positional encoding to provide frequency inductive biases. The method was developed to address the issue of high-frequency noise in discrete grid-based features and the depth bleeding caused by simple smoothing techniques [S1].

## Sources
[S1] PET-NeuS: Positional Encoding Tri-Planes for Neural Surfaces, http://arxiv.org/abs/2305.05594v1, fetched 2026-10-27
