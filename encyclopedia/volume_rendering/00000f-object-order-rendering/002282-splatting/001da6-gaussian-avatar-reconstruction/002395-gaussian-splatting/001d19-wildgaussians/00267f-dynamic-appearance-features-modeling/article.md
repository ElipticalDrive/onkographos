# Dynamic Appearance Features Modeling

Also called: Adaptive Sampling with K Feature Maps

## Overview
Dynamic Appearance Features Modeling addresses the limitation of global feature embeddings in 3D Gaussian Splatting, which assume uniform environmental information (lighting, weather) across a scene [S1]. By extracting dynamic appearance features from a reference image and allowing each Gaussian point to sample from multiple feature maps, the method captures local details such as highlights and shadows that global embeddings miss [S1]. It sets itself apart from methods like NeRF-W and Ha-NeRF, which use global scene embeddings, and CR-NeRF, which integrates features at the 2D level without dynamic 3D environmental information [S1].

## How it works
**Inputs:**
- Reference image $I_{gt} \in \mathbb{R}^{3 \times H \times W}$ [S1].
- Known camera pose for the reference image, defining the projection matrix $P$ [S1].
- Set of $N$ 3D Gaussian points, where each point $i$ has a position coordinate $X_i$ [S1].

**Outputs:**
- Dynamic appearance feature vector $df_i \in \mathbb{R}^{16(K+1)}$ for each Gaussian point $i$ [S1].

**Data Structures:**
- **Feature Map $F$**: A 2D tensor of shape $\mathbb{R}^{16(K+1) \times H \times W}$ [S1].
- **Projection Feature Map $F^P$**: The first 16 channels of $F$, shape $\mathbb{R}^{16 \times H \times W}$ [S1].
- **K Feature Maps $(F^1, \dots, F^K)$**: The remaining $16K$ channels of $F$, evenly divided into $K$ maps, each of shape $\mathbb{R}^{16 \times H \times W}$ [S1].
- **Sampling Coordinates $sc_i$**: For each Gaussian point $i$, a set of $K$ learnable 2D coordinates $(sc_i^1, \dots, sc_i^K)$, where each $sc_i^k \in [-1, 1]^2$ [S1].

**Parameters:**
- $K$: The number of additional feature maps for adaptive sampling. Typical value: 3 [S1].

**Steps:**
1. **Feature Extraction**: Pass the reference image $I_{gt}$ through a U-Net model with a ResNet backbone to generate the combined feature map $F \in \mathbb{R}^{16(K+1) \times H \times W}$ [S1].
2. **Map Division**: Split $F$ along the channel dimension into $K+1$ feature maps: the projection feature map $F^P$ (first 16 channels) and $K$ additional feature maps $F^1, \dots, F^K$ (remaining $16K$ channels) [S1].
3. **Projection Feature Sampling**: For each Gaussian point $i$:
   - Project its 3D position $X_i$ onto the 2D image plane using the projection matrix $P$ derived from the camera pose, resulting in coordinates $\Pi_P(X_i)$ [S1].
   - Sample the projection feature $f_i^P \in \mathbb{R}^{16}$ from $F^P$ at $\Pi_P(X_i)$ using bilinear interpolation [S1].
   $$f_i^P = BL(\Pi_P(X_i), F^P) \quad (6)$$
   where $BL$ denotes bilinear interpolation sampling [S1].
4. **Adaptive Feature Sampling**: For each Gaussian point $i$:
   - Use the learnable sampling coordinates $(sc_i^1, \dots, sc_i^K)$ to sample features from the $K$ additional feature maps [S1].
   - Sample features $(f_i^1, \dots, f_i^K) \in \mathbb{R}^{K \times 16}$ from $(F^1, \dots, F^K)$ using bilinear interpolation at the respective coordinates [S1].
   $$(f_i^1, f_i^2, \dots, f_i^K) = BL((sc_i^1, sc_i^2, \dots, sc_i^K), (F^1, F^2, \dots, F^K)) \quad (7)$$
5. **Concatenation**: Concatenate the projection feature and the $K$ sampled features to form the dynamic appearance feature $df_i$ [S1].
   $$df_i = f_i^P \oplus f_i^1 \oplus f_i^2 \oplus \dots \oplus f_i^K \quad (8)$$
   where $\oplus$ denotes concatenation [S1].
6. **Regularization**: Apply a regularization loss $L_{sc}$ to prevent sampling coordinates from deviating beyond the effective sampling range $[-1, 1]$ [S1].
   $$L_{sc} = \frac{1}{N} \sum_{i=1}^N \max\{0, |sc_i| - 1\} \quad (9)$$
   where $N$ is the total number of Gaussian points and $| \cdot |$ denotes the absolute value applied to each coordinate component [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def extract_features(reference_image, unet_model):
    """
    Extracts the combined feature map F from the reference image using a U-Net with ResNet backbone.
    Args:
        reference_image: Tensor of shape (3, H, W)
        unet_model: U-Net model instance
    Returns:
        F: Tensor of shape (16*(K+1), H, W)
    """
    F = unet_model(reference_image)
    return F

def split_feature_maps(F, K):
    """
    Splits the combined feature map F into projection feature map F^P and K additional feature maps.
    Args:
        F: Tensor of shape (16*(K+1), H, W)
        K: Number of additional feature maps
    Returns:
        F_P: Tensor of shape (16, H, W)
        F_K: List of K tensors, each of shape (16, H, W)
    """
    # Split along the channel dimension
    # First 16 channels are F^P
    F_P = F[:16, :, :]
    # Remaining 16*K channels are split into K maps
    F_K = []
    for k in range(K):
        start_idx = 16 + k * 16
        end_idx = 16 + (k + 1) * 16
        F_K.append(F[start_idx:end_idx, :, :])
    return F_P, F_K

def sample_projection_feature(X_i, P, F_P):
    """
    Samples the projection feature for a single Gaussian point.
    Args:
        X_i: 3D position of the Gaussian point, shape (3,)
        P: Projection matrix, shape (3, 4) or (4, 4) depending on implementation
        F_P: Projection feature map, shape (16, H, W)
    Returns:
        f_i_P: Sampled feature, shape (16,)
    """
    # Project 3D point to 2D image coordinates
    # Assuming homogeneous coordinates for X_i if needed
    X_i_homogeneous = torch.cat([X_i, torch.tensor(1.0)])
    projected_coords = P @ X_i_homogeneous
    # Normalize to get 2D coordinates (u, v)
    # This step depends on the specific projection matrix definition
    # Typically: u = x/z, v = y/z, then scaled to [-1, 1] or [0, 1]
    # Assuming the projection matrix P handles the conversion to normalized device coordinates
    # or that the bilinear interpolation function expects coordinates in a specific range.
    # Based on the source, BL expects coordinates derived from P.
    # Let's assume projected_coords gives (u, v) in a range suitable for bilinear interpolation.
    # If the range is [0, 1], we might need to adjust.
    # The source says "projected onto the 2D projection feature map F^P using a projection matrix P"
    # and then "bilinear interpolation sampling".
    # Standard bilinear interpolation in PyTorch expects coordinates in [-1, 1] for grid_sample.
    # Let's assume the projection matrix P outputs coordinates in [-1, 1].
    u, v = projected_coords[0], projected_coords[1]
    
    # Reshape for grid_sample: (1, 1, 1, 2) for a single point
    # grid_sample expects input of shape (N, C, H, W) and grid of shape (N, H_out, W_out, 2)
    # Here we sample a single point, so H_out=1, W_out=1
    grid = torch.tensor([[[[u, v]]]]) # Shape (1, 1, 1, 2)
    
    # F_P needs to be (N, C, H, W)
    F_P_batched = F_P.unsqueeze(0) # Shape (1, 16, H, W)
    
    # Sample using bilinear interpolation
    # align_corners=False is standard for grid_sample
    f_i_P = F.grid_sample(F_P_batched, grid, align_corners=False)
    # f_i_P shape: (1, 16, 1, 1)
    f_i_P = f_i_P.squeeze() # Shape (16,)
    
    return f_i_P

def sample_adaptive_features(sc_i, F_K):
    """
    Samples features from K feature maps using learnable sampling coordinates.
    Args:
        sc_i: List of K sampling coordinates, each (2,) in [-1, 1]
        F_K: List of K feature maps, each (16, H, W)
    Returns:
        features: Tensor of shape (K, 16)
    """
    features = []
    for k in range(len(F_K)):
        sc = sc_i[k] # (2,)
        u, v = sc[0], sc[1]
        
        # Reshape for grid_sample
        grid = torch.tensor([[[[u, v]]]]) # Shape (1, 1, 1, 2)
        
        F_k_batched = F_K[k].unsqueeze(0) # Shape (1, 16, H, W)
        
        # Sample using bilinear interpolation
        f_i_k = F.grid_sample(F_k_batched, grid, align_corners=False)
        f_i_k = f_i_k.squeeze() # Shape (16,)
        
        features.append(f_i_k)
    
    # Concatenate along the first dimension
    features = torch.stack(features, dim=0) # Shape (K, 16)
    return features

def compute_dynamic_appearance_feature(X_i, P, F_P, sc_i, F_K):
    """
    Computes the dynamic appearance feature df_i for a single Gaussian point.
    Args:
        X_i: 3D position, shape (3,)
        P: Projection matrix
        F_P: Projection feature map, shape (16, H, W)
        sc_i: List of K sampling coordinates
        F_K: List of K feature maps
    Returns:
        df_i: Dynamic appearance feature, shape (16*(K+1),)
    """
    f_i_P = sample_projection_feature(X_i, P, F_P)
    f_i_K = sample_adaptive_features(sc_i, F_K)
    
    # Concatenate f_i_P and f_i_K
    # f_i_P shape: (16,)
    # f_i_K shape: (K, 16)
    # We need to flatten f_i_K to (16*K,) and concatenate
    f_i_K_flat = f_i_K.flatten()
    df_i = torch.cat([f_i_P, f_i_K_flat])
    
    return df_i

def compute_regularization_loss(sc_i_list, N):
    """
    Computes the regularization loss L_sc for all Gaussian points.
    Args:
        sc_i_list: List of N lists, each containing K sampling coordinates
        N: Total number of Gaussian points
    Returns:
        L_sc: Scalar loss value
    """
    loss = 0.0
    for i in range(N):
        sc_i = sc_i_list[i]
        for sc in sc_i:
            # sc is (2,)
            # Compute max(0, |sc| - 1) for each component
            abs_sc = torch.abs(sc)
            loss_component = torch.clamp(abs_sc - 1, min=0)
            loss += torch.sum(loss_component)
    
    L_sc = loss / N
    return L_sc
```

**Edge Cases and Pitfalls:**
- **Precision**: Bilinear interpolation requires careful handling of coordinate ranges. The projection matrix $P$ must correctly map 3D points to the 2D image plane in a coordinate system compatible with the bilinear interpolation function (typically $[-1, 1]$ for `grid_sample` in PyTorch) [S1].
- **Boundaries**: The regularization loss $L_{sc}$ is designed to keep sampling coordinates within $[-1, 1]$. If coordinates exceed this range, the loss penalizes them, but sampling outside this range may result in undefined behavior or zero-padding depending on the interpolation implementation [S1].
- **Degenerate Input**: If the reference image $I_{gt}$ is not available or the camera pose is unknown, the projection feature map $F^P$ cannot be computed, and the method fails [S1].

## Operating characteristics
- **Time Complexity**: The feature extraction is performed once per reference image using a U-Net, which is a fixed cost per training step. For each Gaussian point, sampling involves $K+1$ bilinear interpolations. With $K=3$, this is 4 interpolations per point. The complexity is linear with respect to the number of Gaussian points $N$ and the number of feature maps $K$ [S1].
- **Memory Complexity**: The feature map $F$ has size $16(K+1) \times H \times W$. For $K=3$, this is $64 \times H \times W$. The sampling coordinates $sc_i$ are stored for each of the $N$ Gaussian points, with $K$ coordinates per point. The memory usage is proportional to $N \times K$ [S1].
- **Measured Performance**: The method was trained on a single Nvidia RTX 3090 GPU for 70k steps, taking approximately 2 hours. Images were downsampled 2 times during training and evaluation [S1].
- **Scaling**: Increasing $K$ beyond 3 brings no performance improvement with meaningless computational cost, suggesting that $K=3$ is an optimal balance [S1].
- **Strengths**: Captures local details such as highlights and shadows, which global embeddings miss. Aligns better with real-world scenarios where environmental influences vary across a scene [S1].
- **Limitations**: Limited by the single-view reference image $I_{gt}$. Features sampled from the projection feature map are identical along the same ray, which is unreasonable under uneven lighting, hence the need for adaptive sampling [S1].

## Background
- **Origin**: Introduced in "Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections" [S1].
- **History and Development**: Previous methods like NeRF-W and Ha-NeRF used global feature embeddings, assuming uniform environmental information, while CR-NeRF integrated features at the 2D level without dynamic 3D environmental information. This method addresses these limitations by allowing each Gaussian point to independently learn its sampling positions [S1].
- **Usage**: Used in the WildGaussians framework for 'in-the-wild' settings, integrating robust DINO features and a lightweight appearance modeling module within the 3DGS framework [Parent].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
