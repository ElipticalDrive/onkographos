# Residual Vector Quantization (R-VQ)
Also called: Geometry Codebook (R-VQ), R-VQ, Residual Vector Quantization for Scale and Rotation

## Overview
Residual Vector Quantization (R-VQ) is a lossy compression technique used to compactly represent high-dimensional geometric attributes, specifically scale and rotation, of 3D Gaussians in radiance field representations. It addresses the high memory overhead of storing raw geometric parameters by mapping them to a set of learned codebooks through a cascaded quantization process. Unlike standard Vector Quantization which maps vectors to a single codebook, R-VQ decomposes the quantization error across multiple stages, allowing for higher precision with smaller individual codebooks. It is distinct from [Hierarchical Vector Quantization](827-hierarchical-vector-quantization.md) in its specific application to Gaussian geometric attributes within 3D Gaussian Splatting pipelines, and from [Vector Quantization for Volumetric Radiance Fields](1870-vector-quantization-for-volumetric-radiance-fields.md) by its focus on scale/rotation rather than general feature vectors.

## How it works
### Inputs and Outputs
**Inputs:**
- $N$: Number of Gaussians.
- $L$: Number of R-VQ stages.
- $C$: Codebook size (number of vectors per codebook).
- $\mathbf{r} \in \mathbb{R}^{N \times 4}$: Input rotation vectors (quaternions) for $N$ Gaussians.
- $\mathbf{s} \in \mathbb{R}^{N \times 3}$: Input scale vectors for $N$ Gaussians.

**Outputs:**
- $\tilde{\mathbf{r}} \in \mathbb{R}^{N \times 4}$: Reconstructed rotation vectors.
- $\tilde{\mathbf{s}} \in \mathbb{R}^{N \times 3}$: Reconstructed scale vectors.
- $\mathbf{i}^l \in \{0, \dots, C-1\}^N$: Indices selected at each stage $l \in \{1, \dots, L\}$.
- $\mathbf{Z}^l \in \mathbb{R}^{C \times 4}$: Codebooks for rotation at each stage $l$.
- $\mathbf{Z}^l_s \in \mathbb{R}^{C \times 3}$: Codebooks for scale at each stage $l$.

### Data Structures
- **Codebook ($\mathbf{Z}^l$):** A matrix of shape $C \times 4$ (for rotation) or $C \times 3$ (for scale). Each row is a prototype vector.
- **Indices ($\mathbf{i}^l$):** A vector of length $N$, storing integer indices into the codebook for each Gaussian.

### Algorithm Steps
1. **Initialization:** Initialize codebooks $\mathbf{Z}^l$ and $\mathbf{Z}^l_s$ using K-means clustering on the input data $\mathbf{r}$ and $\mathbf{s}$ respectively.
2. **Training Phase (Last 1K Iterations):**
   - For the final $K=1000$ training iterations, compute the reconstruction loss for rotation ($L_r$) and scale ($L_s$).
   - For all other iterations, set $L_r = 0$ and $L_s = 0$ to avoid heavy computation.
3. **Quantization (Inference/Forward Pass):**
   - For each Gaussian $n$ and each stage $l$ from $1$ to $L$:
     - Compute the residual vector: $\mathbf{residual} = \mathbf{r}_n - \tilde{\mathbf{r}}_n^{l-1}$ (where $\tilde{\mathbf{r}}_n^0 = \mathbf{0}$).
     - Select the index $i_n^l$ that minimizes the squared Euclidean distance to the residual:
       $$ i_n^l = \text{argmin}_k \| \mathbf{Z}^l[k] - (\mathbf{r}_n - \tilde{\mathbf{r}}_n^{l-1}) \|_2^2 $$
     - Update the reconstructed vector using the selected codebook entry:
       $$ \tilde{\mathbf{r}}_n^l = \tilde{\mathbf{r}}_n^{l-1} + \mathbf{Z}^l[i_n^l] $$
   - The final reconstructed rotation is $\tilde{\mathbf{r}}_n = \tilde{\mathbf{r}}_n^L$.
   - Apply the same process to scale $\mathbf{s}$ using scale codebooks $\mathbf{Z}^l_s$.
4. **Masking (Post-Quantization):**
   - Generate binary masks $\mathbf{M} \in \{0, 1\}^N$ based on a mask parameter $\mathbf{m}$ and threshold $\epsilon$ using a straight-through estimator:
     $$ M_n = \text{sg}(\mathbb{1}[\sigma(m_n) > \epsilon] - \sigma(m_n)) + \sigma(m_n) $$
   - Apply masks to reconstructed scale and opacity:
     $$ \hat{s}_n = M_n \tilde{s}_n $$
     $$ \hat{o}_n = M_n \tilde{o}_n $$

### Formulas
**Reconstruction:**
$$ \tilde{\mathbf{r}}_n^l = \sum_{k=1}^{l} \mathbf{Z}^k[\mathbf{i}_n^k], \quad l \in \{1, \dots, L\} $$
$$ \tilde{\mathbf{r}}_n^0 = \mathbf{0} $$

**Index Selection:**
$$ \mathbf{i}_n^l = \text{argmin}_k \| \mathbf{Z}^l[k] - (\mathbf{r}_n - \tilde{\mathbf{r}}_n^{l-1}) \|_2^2 $$

**Training Objective (Rotation):**
$$ L_r = \frac{1}{NC} \sum_{n=1}^{N} \sum_{k=1}^{K} \| \text{sg}[\mathbf{r}_n - \tilde{\mathbf{r}}_n^{k-1}] - \mathbf{Z}^k[\mathbf{i}_n^k] \|_2^2 $$
Where $\text{sg}[\cdot]$ is the stop-gradient operator. The objective for scale $L_s$ is analogous.

**Total Loss:**
$$ L = L_{ren} + \lambda_m L_m + L_r + L_s $$
Where $L_{ren}$ is the rendering loss (L1 + SSIM), $L_m$ is the masking loss, and $\lambda_m$ is a regularization hyperparameter.

### Parameters
- $L$: Number of stages. Not given in the sources.
- $C$: Codebook size. Not given in the sources.
- $K$: Number of training iterations for codebook learning. $1000$ (last 1K iterations).
- $\epsilon$: Masking threshold. Not given in the sources.
- $\lambda_m$: Hyperparameter for masking regularization. Not given in the sources.

### Edge Cases and Pitfalls
- **Gradient Flow:** The straight-through estimator is required for binary masks because gradients cannot flow through binarization directly [S1].
- **Training Stability:** Codebooks are only trained in the last 1K iterations to prevent heavy computation and ensure optimal training speed [S1].
- **Dimensionality:** Rotation vectors are 4-dimensional (quaternions), while scale vectors are 3-dimensional [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def residual_vector_quantization(r, s, Z_r, Z_s, L, C):
    """
    Performs Residual Vector Quantization on rotation and scale vectors.
    
    Args:
        r: numpy array of shape (N, 4), input rotation vectors.
        s: numpy array of shape (N, 3), input scale vectors.
        Z_r: list of L numpy arrays, each shape (C, 4), rotation codebooks.
        Z_s: list of L numpy arrays, each shape (C, 3), scale codebooks.
        L: int, number of stages.
        C: int, codebook size.
        
    Returns:
        r_hat: numpy array of shape (N, 4), reconstructed rotations.
        s_hat: numpy array of shape (N, 3), reconstructed scales.
        i_r: list of L numpy arrays, indices for rotation.
        i_s: list of L numpy arrays, indices for scale.
    """
    N = r.shape[0]
    
    # Initialize reconstructed vectors
    r_hat = np.zeros_like(r)
    s_hat = np.zeros_like(s)
    
    i_r = []
    i_s = []
    
    # Quantize Rotation
    for l in range(L):
        # Compute residual
        residual_r = r - r_hat
        
        # Find best codebook entry for each Gaussian
        # Z_r[l] is (C, 4), residual_r is (N, 4)
        # We want argmin_k || Z_r[l][k] - residual_r[n] ||^2
        # This is equivalent to finding the closest vector in Z_r[l] to each row in residual_r
        
        # Compute distances: (N, C)
        # dist[n, k] = || residual_r[n] - Z_r[l][k] ||^2
        # Using broadcasting: residual_r[:, None, :] - Z_r[l][None, :, :]
        diff_r = residual_r[:, np.newaxis, :] - Z_r[l][np.newaxis, :, :]
        dist_r = np.sum(diff_r ** 2, axis=2)
        
        # Select indices
        idx_r = np.argmin(dist_r, axis=1)
        i_r.append(idx_r)
        
        # Update reconstructed vector
        # Gather the selected codebook vectors
        selected_codes_r = Z_r[l][idx_r] # Shape (N, 4)
        r_hat = r_hat + selected_codes_r
        
    # Quantize Scale
    for l in range(L):
        residual_s = s - s_hat
        
        diff_s = residual_s[:, np.newaxis, :] - Z_s[l][np.newaxis, :, :]
        dist_s = np.sum(diff_s ** 2, axis=2)
        
        idx_s = np.argmin(dist_s, axis=1)
        i_s.append(idx_s)
        
        selected_codes_s = Z_s[l][idx_s] # Shape (N, 3)
        s_hat = s_hat + selected_codes_s
        
    return r_hat, s_hat, i_r, i_s

def compute_rotation_loss(r, Z_r, i_r, L):
    """
    Computes the training loss for rotation codebooks.
    """
    N = r.shape[0]
    total_loss = 0.0
    
    # Reconstruct r_hat step-by-step to get intermediate residuals
    r_hat = np.zeros_like(r)
    
    for k in range(L):
        # Residual before stage k
        residual = r - r_hat
        
        # Stop gradient on residual
        residual_sg = residual.detach() if hasattr(residual, 'detach') else residual
        
        # Selected code
        code = Z_r[k][i_r[k]]
        
        # Loss term
        loss_term = np.sum((residual_sg - code) ** 2)
        total_loss += loss_term
        
        # Update r_hat
        r_hat = r_hat + code
        
    # Normalize by N * C (as per formula, though C is not explicitly in the sum normalization in the text, 
    # the formula says 1/(NC) sum over N and K. Here we sum over K stages inside the loop, 
    # so we divide by N*C at the end if we summed all K. 
    # The formula: Lr = 1/(NC) sum_n sum_k ||...||^2
    # Our loop sums over k. So we divide by N*C.
    avg_loss = total_loss / (N * L) # Note: The formula uses K stages, here L is stages.
    
    return avg_loss
```

## Subtopics

### [Hierarchical Vector Quantization](827-hierarchical-vector-quantization.md)
Uses a tree structure for codebooks. Choose when hierarchical clustering benefits are needed.

### [LBG Algorithm](840-lbg-algorithm.md)
A method for designing codebooks. Choose when training codebooks from scratch is required.

### [Progressive Encoding of Time-Varying Sequences](959-progressive-encoding-of-time-varying-sequences.md)
Focuses on temporal data. Choose for video or dynamic scenes.

### [Vector Quantization for Volumetric Radiance Fields](1870-vector-quantization-for-volumetric-radiance-fields.md)
Applies VQ to general radiance fields. Choose for general neural radiance fields.

### [Fortune's Palette Compression](4988-fortune-s-palette-compression.md)
Uses palette-based compression. Choose for color-heavy data.

### [Compact3D / C3DGS / EAGLES](6139-compact3d-c3dgs-eagles.md)
Related compact 3D Gaussian methods. Choose for overall scene compression.

### [Codebook Sharing](7563-codebook-sharing.md)
Shares codebooks across attributes. Choose to reduce memory further.

### [Brood-based Vector Quantization](7595-brood-based-vector-quantization.md)
Uses brood-based clustering. Choose for specific clustering needs.

## Operating characteristics
- **Time Complexity:** Quantization is $O(N \cdot L \cdot C \cdot D)$ where $D$ is dimension (4 for rotation, 3 for scale). Training is limited to the last 1K iterations to reduce cost [S1].
- **Memory Complexity:** Storage is reduced significantly. On Mip-NeRF 360, storage dropped from 746 MB to 48.8 MB (or 29.1 MB with progressive packing) [S1]. On Tanks&Temples, from 411 MB to 39.4 MB (or 20.9 MB) [S1].
- **Strengths:** High compression ratio for geometric attributes; maintains rendering quality comparable to original 3DGS [S1].
- **Limitations:** Requires additional training steps for codebooks; not suitable for real-time codebook updates during inference without pre-computation.
- **When to Use:** When storage and bandwidth are critical constraints for 3D Gaussian Splatting models.
- **When Not to Use:** When real-time codebook training is required or when geometric attributes have high variance that cannot be captured by small codebooks.

## Background
Residual Vector Quantization was proposed in "Compact 3D Gaussian Representation for Radiance Field" (CVPR 2024) to address the high memory footprint of 3D Gaussian Splatting. The method leverages the limited variations in geometric attributes (scale and rotation) to learn compact codebooks. It builds on standard Vector Quantization but cascades multiple stages to reduce quantization error [S1].

## Sources
[S1] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
