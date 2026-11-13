# Binary Radiance Fields

Also called: BRF

## Overview
Binary Radiance Fields (BiRF) is a storage-efficient radiance field representation that constrains feature grid parameters to binary values (+1 or -1) to drastically reduce memory overhead compared to real-valued floating-point grids [S1]. It distinguishes itself from post-processing compression methods (e.g., PlenOctrees, Re:NeRF) by performing binarization during training without requiring additional post-optimization steps, and from standard hash-based methods (e.g., Instant-NGP) by using a 2D-3D hybrid grid structure to mitigate hash collisions while maintaining high reconstruction fidelity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D coordinate $\mathbf{x} = (x, y, z)$ and viewing direction $\mathbf{d} = (\theta, \phi)$.
*   **Output:** Volume density $\sigma$ and color $\mathbf{c} = (r, g, b)$.

**Data Structures:**
The representation consists of a multi-resolution 2D-3D hybrid feature grid implemented using hash encoding [S1].
1.  **3D Voxel Grid ($\Phi_{\theta_{xyz}}$):** Contains $L$ resolution levels. Resolutions range from 16 to 1024. Each level holds up to $T_{3D}$ feature vectors.
2.  **2D Plane Grids ($\Phi_{\theta_{xy}}, \Phi_{\theta_{xz}}, \Phi_{\theta_{yz}}$):** Three orthogonal planes. Each contains $M$ resolution levels. Resolutions range from 64 to 512. Each level holds up to $T_{2D}$ feature vectors.
3.  **Feature Values:** All grid parameters are constrained to binary values $\{-1, +1\}$, stored as 1-bit integers [S1].
4.  **MLPs:**
    *   Density MLP: 1 hidden layer, 128 channels, ReLU activation.
    *   Color MLP: 2 hidden layers, 128 channels, ReLU activation.
    *   Directional Encoding: Spherical Harmonics basis functions [S1].

**Parameters:**
*   $L$: Number of 3D grid levels.
*   $M$: Number of 2D grid levels.
*   $T_{3D}$: Max feature vectors per 3D level.
*   $T_{2D}$: Max feature vectors per 2D level.
*   $F$: Feature dimension per level. Variants use $F \in \{2, 4, 8\}$ [S1].
*   $\lambda_{\text{sparsity}}$: Sparsity loss weight. Value: $2.0 \times 10^{-5}$ [S1].
*   Model Variants:
    *   Small (Ours-S): $\log_2(T_{3D}) = 17$, $\log_2(T_{2D}) = 15$ [S1].
    *   Base (Ours-B): $\log_2(T_{3D}) = 19$, $\log_2(T_{2D}) = 17$ [S1].

**Algorithm Steps:**

1.  **Binarization (Training):**
    Real-valued grid parameters $\theta$ are transformed to binary $\theta'$ using the sign function:
    $$ \theta' = \text{sign}(\theta) = \begin{cases} +1 & \text{if } \theta \ge 0 \\ -1 & \text{otherwise} \end{cases} $$
    Gradients are propagated using the Straight-Through Estimator (STE). The gradient is passed if $|\theta| \le 1$, otherwise it is zeroed to prevent scale divergence [S1].

2.  **Feature Encoding:**
    For a given 3D coordinate $\mathbf{x}$:
    *   **3D Features ($f_{xyz}$):** Compute features from the 3D grid by linearly (tri-linearly) interpolating the binary parameters across $L$ resolution levels and concatenating them:
        $$ f_{xyz} = \{ \text{interp}(\mathbf{x}, \text{sign}(\Phi^l_{\theta_{xyz}})) \}_{l=1}^L $$
    *   **2D Features ($f_{xy}, f_{xz}, f_{yz}$):** Project $\mathbf{x}$ to 2D coordinates: $\mathbf{x}_{xy}=(x,y)$, $\mathbf{x}_{xz}=(x,z)$, $\mathbf{x}_{yz}=(y,z)$. Interpolate binary parameters across $M$ resolution levels for each plane and concatenate:
        $$ f_{xy} = \{ \text{interp}(\mathbf{x}_{xy}, \text{sign}(\Phi^m_{\theta_{xy}})) \}_{m=1}^M $$
        Repeat for $f_{xz}$ and $f_{yz}$ [S1].

3.  **Feature Concatenation:**
    Combine all encoded features:
    $$ f = \{ f_{xyz}, f_{xy}, f_{xz}, f_{yz} \} $$

4.  **Positional Encoding:**
    Apply sinusoidal positional encoding $\gamma(\mathbf{x})$ to the input coordinate $\mathbf{x}$ [S1].

5.  **Density Prediction:**
    Feed $\gamma(\mathbf{x})$ and $f$ into the density MLP:
    $$ (\sigma, e) = \text{MLP}_{\text{density}}(\gamma(\mathbf{x}), f) $$
    where $\sigma$ is density and $e$ is the embedded feature [S1].

6.  **Color Prediction:**
    Feed $e$ and directional information $\mathbf{d}$ (encoded via Spherical Harmonics) into the color MLP:
    $$ \mathbf{c} = \text{MLP}_{\text{color}}(e, \mathbf{d}) $$

**Loss Function:**
*   **Reconstruction Loss ($L_{\text{recon}}$):** Mean Squared Error between rendered color $\hat{C}(\mathbf{r})$ and ground truth $C(\mathbf{r})$ for sampled rays $\mathbf{r}$ [S1].
    $$ L_{\text{recon}} = \sum_{\mathbf{r} \in \mathcal{R}} ||\hat{C}(\mathbf{r}) - C(\mathbf{r})||_2^2 $$
*   **Sparsity Loss ($L_{\text{sparsity}}$):** Cauchy loss to regularize density sparsity, encouraging empty space to have low density:
    $$ L_{\text{sparsity}} = \sum_{i,k} \log(1 + 2\sigma(\mathbf{r}_i(t_k))^2) $$
*   **Total Loss:**
    $$ L = L_{\text{recon}} + \lambda_{\text{sparsity}} L_{\text{sparsity}} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sign(x):
    """Binarization operator: +1 if x >= 0, else -1."""
    return 1.0 if x >= 0.0 else -1.0

def ste_gradient(loss, theta):
    """
    Straight-Through Estimator for binarization.
    Passes gradient if |theta| <= 1, else 0.
    """
    if abs(theta) <= 1.0:
        return loss
    else:
        return 0.0

def interpolate_3d(grid, x, y, z, levels, max_vectors):
    """
    Tri-linear interpolation of binary 3D grid parameters.
    Returns concatenated features from all levels.
    """
    features = []
    for l in range(levels):
        # Access grid level l
        # Perform tri-linear interpolation at (x, y, z)
        # Note: In implementation, grid is hash-encoded.
        # This function conceptually retrieves and interpolates binary values.
        val = hash_trilinear_interpolate(grid[l], x, y, z)
        features.append(val)
    return concatenate(features)

def interpolate_2d(grid, u, v, levels, max_vectors):
    """
    Bi-linear interpolation of binary 2D grid parameters.
    Returns concatenated features from all levels.
    """
    features = []
    for m in range(levels):
        # Access grid level m
        # Perform bi-linear interpolation at (u, v)
        val = hash_bilinear_interpolate(grid[m], u, v)
        features.append(val)
    return concatenate(features)

def encode_features(x, y, z, grid_3d, grid_xy, grid_xz, grid_yz, L, M):
    """
    Compute binary feature encoding for a 3D coordinate.
    """
    # 3D Grid Encoding
    f_xyz = interpolate_3d(grid_3d, x, y, z, L, T_3D)
    
    # 2D Grid Encodings
    f_xy = interpolate_2d(grid_xy, x, y, M, T_2D)
    f_xz = interpolate_2d(grid_xz, x, z, M, T_2D)
    f_yz = interpolate_2d(grid_yz, y, z, M, T_2D)
    
    # Concatenate
    f = concatenate([f_xyz, f_xy, f_xz, f_yz])
    return f

def positional_encoding(x, num_frequencies=10):
    """
    Sinusoidal positional encoding.
    """
    freq_bands = 2.0 ** torch.linspace(0, num_frequencies - 1, num_frequencies)
    encoded = torch.cat([torch.sin(x * freq_bands), torch.cos(x * freq_bands)], dim=-1)
    return encoded

def predict_density_and_color(x, d, grid_3d, grid_xy, grid_xz, grid_yz, L, M, F):
    """
    Forward pass of BiRF.
    """
    # 1. Feature Encoding
    f = encode_features(x[0], x[1], x[2], grid_3d, grid_xy, grid_xz, grid_yz, L, M)
    
    # 2. Positional Encoding
    gamma_x = positional_encoding(x)
    
    # 3. Density MLP
    # MLP_Density takes [gamma_x, f]
    density_input = concatenate([gamma_x, f])
    sigma, e = mlp_density(density_input)
    
    # 4. Color MLP
    # MLP_Color takes [e, directional_encoding(d)]
    # Directional encoding uses Spherical Harmonics
    d_encoded = spherical_harmonics(d)
    color_input = concatenate([e, d_encoded])
    c = mlp_color(color_input)
    
    return sigma, c

def compute_loss(rendered_color, ground_truth_color, density_samples, lambda_sparsity):
    """
    Compute total loss.
    """
    # Reconstruction Loss (MSE)
    l_recon = mean_squared_error(rendered_color, ground_truth_color)
    
    # Sparsity Loss (Cauchy)
    # Sum over all sampled points (i, k) along rays
    l_sparsity = 0.0
    for sigma in density_samples:
        l_sparsity += math.log(1 + 2 * sigma**2)
    
    total_loss = l_recon + lambda_sparsity * l_sparsity
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** 3D grids suffer from hash collisions at higher resolutions. The 2D grid supplementation alleviates this impact [S1].
*   **Gradient Vanishing:** The derivative of the sign function is zero almost everywhere. The STE is required to pass gradients [S1].
*   **Scale Divergence:** Large values of $\theta$ can cause divergence. The STE stops propagating gradients when $|\theta| > 1$, constraining values to $\{-1, 1\}$ [S1].

## Operating characteristics
*   **Storage Efficiency:** Extremely low. The Ours-S2 variant requires < 0.5 MB for Synthetic-NeRF scenes, outperforming TensoRF-CP (3.7 MB) and Instant-NGP (24.5 MB) on the mic scene [S1].
*   **Training Time:** Approximately 6, 9, and 14 minutes for Synthetic-NeRF scenes for Ours-B2 to Ours-B8 models on a single NVIDIA RTX A6000 [S1].
*   **Performance:** PSNR of 32.03 dB on Synthetic-NeRF, 34.48 dB on Synthetic-NSVF, and 28.20 dB on Tanks and Temples [S1].
*   **Strengths:** Compact storage, no post-optimization required, comparable or better reconstruction quality than larger baselines.
*   **Limitations:** Binary constraints may limit representational capacity compared to full-precision models, though the hybrid 2D-3D structure mitigates this [S1].

## Background
Binary Radiance Fields were proposed by Seungjoo Shin and Jaesik Park from POSTECH and Seoul National University, published in December 2023 [S1]. The method builds on the Instant-NGP architecture [20] and uses the occupancy grid from NerfAcc [42]. It addresses the storage bottleneck of neural radiance fields by introducing binary feature encoding [S1].

## Sources
[S1] Binary Radiance Fields, https://doi.org/10.52202/075280-2438, fetched 2026-10-26
