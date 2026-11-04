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

def extract_features(reference_image: torch.Tensor, unet_model: torch.nn.Module) -> torch.Tensor:
    """Extracts the combined feature map F from the reference image using a U-Net with ResNet backbone."""
    return unet_model(reference_image)

def split_feature_maps(F_map: torch.Tensor, K: int):
    """Splits the combined feature map F into projection feature map F^P and K additional feature maps."""
    F_P = F_map[:16, :, :]
    F_K = [F_map[16 + k * 16 : 16 + (k + 1) * 16, :, :] for k in range(K)]
    return F_P, F_K

def sample_projection_feature(X_i: torch.Tensor, P: torch.Tensor, F_P: torch.Tensor) -> torch.Tensor:
    """Samples the projection feature for a single Gaussian point."""
    X_i_hom = torch.cat([X_i, torch.ones(1, device=X_i.device)])
    projected = P @ X_i_hom
    u, v = projected[0], projected[1]
    grid = torch.tensor([[[[u, v]]]], device=X_i.device)
    F_P_batch = F_P.unsqueeze(0)
    return F.grid_sample(F_P_batch, grid, align_corners=False).squeeze()

def sample_adaptive_features(sc_i: list, F_K: list) -> torch.Tensor:
    """Samples features from K feature maps using learnable sampling coordinates."""
    features = []
    for k in range(len(F_K)):
        u, v = sc_i[k][0], sc_i[k][1]
        grid = torch.tensor([[[[u, v]]]], device=sc_i[k][0].device)
        F_k_batch = F_K[k].unsqueeze(0)
        features.append(F.grid_sample(F_k_batch, grid, align_corners=False).squeeze())
    return torch.stack(features, dim=0)

def compute_dynamic_appearance_feature(X_i: torch.Tensor, P: torch.Tensor, F_P: torch.Tensor, sc_i: list, F_K: list) -> torch.Tensor:
    """Computes the dynamic appearance feature df_i for a single Gaussian point."""
    f_i_P = sample_projection_feature(X_i, P, F_P)
    f_i_K = sample_adaptive_features(sc_i, F_K)
    return torch.cat([f_i_P, f_i_K.flatten()])

def compute_regularization_loss(sc_i_list: list, N: int) -> torch.Tensor:
    """Computes the regularization loss L_sc for all Gaussian points."""
    total_loss = torch.tensor(0.0, device=sc_i_list[0][0].device)
    for sc_i in sc_i_list:
        for sc in sc_i:
            total_loss += torch.sum(torch.clamp(torch.abs(sc) - 1, min=0))
    return total_loss / N
```

**Edge Cases and Pitfalls:**
- **Precision**: The projection matrix $P$ must map 3D points to 2D coordinates compatible with the bilinear interpolation function's valid range [G].
- **Boundaries**: The regularization loss $L_{sc}$ is designed to keep sampling coordinates within $[-1, 1]$. If coordinates exceed this range, the loss penalizes them, but sampling outside this range may result in undefined behavior or zero-padding depending on the interpolation implementation [S1].
- **Degenerate Input**: If the reference image $I_{gt}$ is not available or the camera pose is unknown, the projection feature map $F^P$ cannot be computed, and the method fails [S1].

## Operating characteristics
- **Time Complexity**: Not given in the sources [S1].
- **Memory Complexity**: Not given in the sources [S1].
- **Measured Performance**: The method was trained on a single Nvidia RTX 3090 GPU for 70k steps, taking approximately 2 hours, with images downsampled twice during training and evaluation [S1].
- **Scaling**: Increasing $K$ beyond 3 yields no performance improvement while adding meaningless computational cost, establishing $K=3$ as optimal [S1].
- **Strengths**: Captures local details such as highlights and shadows that global embeddings miss, aligning better with real-world varying environmental influences [S1].
- **Limitations**: Limited by the single-view reference image $I_{gt}$, where features sampled from the projection feature map are identical along the same ray, which is unreasonable under uneven lighting [S1].

## Background
- **Origin**: Introduced in "Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections" [S1].
- **History and Development**: Previous methods like NeRF-W and Ha-NeRF used global feature embeddings assuming uniform environmental information, while CR-NeRF integrated features at the 2D level without dynamic 3D environmental information [S1]. This method addresses these limitations by allowing each Gaussian point to independently learn its sampling positions [S1].
- **Usage**: Used in the WildGaussians framework for 'in-the-wild' settings, integrating robust DINO features and a lightweight appearance modeling module within the 3DGS framework [Parent].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
