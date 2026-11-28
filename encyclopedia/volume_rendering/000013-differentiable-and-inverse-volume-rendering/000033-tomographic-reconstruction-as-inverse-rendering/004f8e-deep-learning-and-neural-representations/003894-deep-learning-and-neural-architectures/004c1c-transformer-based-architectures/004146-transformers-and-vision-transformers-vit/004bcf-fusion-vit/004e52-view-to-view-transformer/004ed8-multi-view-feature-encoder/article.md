# Multi-View Feature Encoder

Also called: MuRF Feature Encoder

## Overview
The Multi-View Feature Encoder aggregates information from $K$ input images to create a compact volumetric representation that encodes both appearance and geometry for subsequent radiance field prediction [S1]. It solves the problem of multi-view inconsistency and information loss by extracting features via a weight-sharing 2D CNN, quantifying consistency through pairwise cosine similarities, and aggregating these features using learned visibility-aware weights [S1]. It sets itself apart from standard feature extractors by explicitly modeling multi-view geometric cues via cosine similarity and using a window-based sampling strategy to mitigate resolution loss from downsampling [S1].

## How it works
**Inputs:** $K$ input images $\{I_k\}_{k=1}^K$.
**Outputs:** A target volume $z \in \mathbb{R}^{\frac{H}{8} \times \frac{W}{8} \times D \times C}$, where $H \times W$ is the image resolution, $D$ is the depth dimension, and $C$ is the channel dimension [S1].

**Data Structures:**
1.  **Feature Maps:** Extracted at 1/8 resolution.
2.  **Cosine Similarity Matrix:** Aggregated scalar $\hat{s}$ per 3D point.
3.  **Aggregated Vectors:** $\hat{c}$ (color) and $\hat{f}$ (feature) per 3D point.

**Steps:**

1.  **2D Feature Extraction:**
    Extract features $\{F_k\}_{k=1}^K$ from the input images using a weight-sharing 2D CNN [S1]. The CNN consists of 6 residual blocks [12]. Every 2 residual blocks include a stride-2 convolutional layer, achieving 2× downsampling. Features are obtained at 1/2, 1/4, and 1/8 resolutions [S1].

2.  **Multi-View Transformer:**
    Feed the 1/8 resolution convolutional features into a joint multi-view Transformer to refine features at the 1/8 resolution [S1]. The Transformer is built on the GMFlow 2-view architecture, extended to $K \ge 2$ views [S1].

3.  **Window-Based Sampling:**
    For a 3D point projected to 2D coordinates on the source views, sample a $9 \times 9$ window centered at the projection from the feature maps of each view $k$ [S1]. This yields a color vector $\tilde{c}_k$ for each view, where $\tilde{c}_k$ is the concatenation of all colors within the window [S1]. Sample image features $f_i$ from the $K$ feature maps [S1].

4.  **Cosine Similarity Computation:**
    Compute pairwise cosine similarities to quantify multi-view consistency [S1].
    Normalize features to unit vectors and collect them into a matrix $F \in \mathbb{R}^{K \times M}$, where $M$ is the feature dimension [S1].
    Compute the matrix product $F F^T$. Extract the upper triangle of this product to obtain all $K(K-1)/2$ pairwise cosine similarities $\cos(f_i, f_j)$ for $i < j$ [S1].
    *Note: Group-wise cosine similarities may be used for improved expressiveness, following the same computation structure [S1].*

5.  **Aggregation of Similarities:**
    Compute the aggregated cosine similarity $\hat{s}$ using learned visibility weights $w_{ij}$ [S1]:
    $$
    \hat{s} = \sum_{i<j} w_{ij} \cos(f_i, f_j)
    $$
    where $i, j = 1, \dots, K$ [S1]. The weights $w_{ij}$ are normalized visibility weights learned from the entropy of the depth distribution on each ray [S1].

6.  **Aggregation of Colors and Features:**
    Compute aggregated color $\hat{c}$ and aggregated feature $\hat{f}$ using learned weights $w_i$ [S1]:
    $$
    \hat{c} = \sum_{i=1}^K w_i \tilde{c}_i, \quad \hat{f} = \sum_{i=1}^K w_i f_i
    $$
    The weights $w_i$ are normalized learned weights [S1]. A small MLP network predicts $w_i$ from the concatenation of the sampled features and the view direction difference between source and target views [S1].

7.  **Volume Projection:**
    Concatenate the aggregated cosine similarity $\hat{s}$, the aggregated color $\hat{c}$, and the aggregated feature $\hat{f}$ [S1]. Project this concatenation to a $C$-dimensional vector using a linear layer [S1]. This produces the target volume $z \in \mathbb{R}^{\frac{H}{8} \times \frac{W}{8} \times D \times C}$ for all 3D points [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MultiViewFeatureEncoder(nn.Module):
    def __init__(self, K, feature_dim, hidden_dim, num_res_blocks=6):
        super().__init__()
        self.K = K
        # 2D CNN with weight sharing across views
        # Consists of 6 residual blocks, stride-2 conv every 2 blocks
        self.cnn = self._build_cnn(feature_dim, num_res_blocks)
        
        # Joint multi-view Transformer (based on GMFlow extension)
        self.transformer = JointMultiViewTransformer(feature_dim, K)
        
        # MLP to predict aggregation weights w_i
        self.weight_mlp = nn.Sequential(
            nn.Linear(feature_dim * 2 + 3, hidden_dim), # features + view diff
            nn.ReLU(),
            nn.Linear(hidden_dim, 1)
        )
        
        # Linear projection for final volume channels
        # Input dim: 1 (similarity) + 81*3 (color window) + feature_dim
        color_window_dim = 81 * 3 
        self.projection = nn.Linear(1 + color_window_dim + feature_dim, C)

    def _build_cnn(self, out_dim, num_blocks):
        # Simplified CNN structure: 6 res blocks, stride-2 every 2 blocks
        # Results in 1/8 resolution features
        layers = []
        # Initial conv to match input channels
        layers.append(nn.Conv2d(3, 64, 7, padding=3))
        layers.append(nn.ReLU())
        
        current_dim = 64
        for i in range(num_blocks):
            layers.append(ResBlock(current_dim))
            if (i + 1) % 2 == 0:
                layers.append(nn.Conv2d(current_dim, current_dim * 2, 3, stride=2, padding=1))
                layers.append(nn.ReLU())
                current_dim *= 2
        
        # Final conv to match feature_dim
        layers.append(nn.Conv2d(current_dim, out_dim, 1))
        return nn.Sequential(*layers)

    def forward(self, images):
        # images: (B, K, 3, H, W)
        B, K, C_in, H, W = images.shape
        
        # Step 1: Extract features using weight-sharing CNN
        # Reshape to process all views through the same CNN
        images_flat = images.view(B * K, C_in, H, W)
        features_raw = self.cnn(images_flat) # (B*K, feature_dim, H/8, W/8)
        features_raw = features_raw.view(B, K, -1, H//8, W//8)
        
        # Step 2: Joint Multi-View Transformer
        features_tf = self.transformer(features_raw) # (B, K, feature_dim, H/8, W/8)
        
        # Step 3: Window Sampling
        # Sample 9x9 window centered at 2D projection. 
        # Assuming we have 3D points projected to 2D (u, v) for each view.
        # For implementation, we assume a grid of 3D points is provided or 
        # we are computing this for a frustum volume.
        # Here we demonstrate the aggregation logic for a single 3D point's projection
        # across K views.
        
        # Let's assume we have projected coordinates (u, v) for the target volume grid
        # Shape: (H/8, W/8, D, 2)
        # We need to sample features and colors from features_tf
        
        # For simplicity, assume we are aggregating for a specific set of 3D points
        # represented by their 2D projections in each view.
        # In practice, this is done via grid_sample or similar for the whole volume.
        
        # Step 4 & 5: Cosine Similarity
        # Normalize features
        features_norm = F.normalize(features_tf, p=2, dim=2) # (B, K, M, H/8, W/8)
        
        # Compute pairwise cosine similarities in parallel
        # Matrix multiplication: (B, K, M) x (B, M, K) -> (B, K, K)
        # We need to handle the spatial dimensions. 
        # Let's flatten spatial dims for similarity computation
        B, K, M, H_s, W_s = features_norm.shape
        features_flat = features_norm.view(B, K, M, -1) # (B, K, M, H_s*W_s)
        
        # Transpose for matmul: (B, K, M, H_s*W_s) -> (B, K, H_s*W_s, M)
        # Actually, we want (K, M) x (K, M)^T for each spatial location
        # So we permute to (B, H_s*W_s, K, M)
        features_perm = features_flat.permute(0, 3, 1, 2) # (B, H_s*W_s, K, M)
        
        # Cosine similarity matrix: (B, H_s*W_s, K, K)
        sim_matrix = torch.bmm(features_perm, features_perm.transpose(1, 2))
        
        # Extract upper triangle (excluding diagonal)
        # Indices i < j
        idx_i, idx_j = torch.triu_indices(K, K, offset=1)
        # Gather similarities
        # sim_matrix shape: (B, H_s*W_s, K, K)
        # We want to aggregate these with weights w_ij
        # w_ij depends on depth entropy, assumed available or learned
        
        # Step 6: Aggregation of Colors and Features
        # Sample colors from original images (downsampled to 1/8)
        images_down = F.interpolate(images, scale_factor=0.125, mode='bilinear') # (B, K, 3, H/8, W/8)
        
        # Sample 9x9 windows for colors and features
        # This requires grid_sample with padding or manual window extraction
        # Assuming a function `sample_window` exists that extracts 9x9 patches
        # and returns flattened vectors
        
        # For the sake of the algorithm description, we assume:
        # c_tilde_k: (B, H_s*W_s, 81*3)
        # f_i: (B, H_s*W_s, M)
        
        # Compute weights w_i using MLP
        # Input: concatenated features and view direction difference
        # View direction difference is not explicitly defined in the snippet 
        # but mentioned as input to MLP. We assume it's available as `view_diff`
        # Shape: (B, K, 3) or similar.
        
        # Predict w_i
        # w_i_raw = MLP(concat(f_i, view_diff))
        # w_i = softmax(w_i_raw)
        
        # Aggregate
        # c_hat = sum(w_i * c_tilde_i)
        # f_hat = sum(w_i * f_i)
        
        # Step 7: Projection
        # Concatenate s_hat, c_hat, f_hat
        # Project to C dimensions
        
        return volume_z
```

**Parameters:**
-   $K$: Number of input views.
-   $C$: Output channel dimension of the volume.
-   Window size: $9 \times 9$.
-   CNN: 6 residual blocks with stride-2 convolutions every 2 blocks [S1].

**Edge Cases:**
-   **Resolution Loss:** The 8× subsampling risks losing information. The $9 \times 9$ window sampling remedies this by capturing local context [S1].
-   **Small vs Large Baseline:** The method is designed to work on both small and large baseline input views by modeling 3D context [S1].

## Subtopics

### [3D Transformer Network Refinement](14648-3d-transformer-network-refinement.md)
The 3D Transformer Network Refinement (also known as the Point Transformer Network) is a module used to optimize the appearance and geometric attributes of selected 3D Gaussian Splatting (GS) tokens. It builds on [Multi-View Feature Encoder](13802-multi-view-feature-encoder.md) by processing encoded features through a hierarchical point transformer architecture. What sets it apart from global attention mechanisms is its use of local attention, which allows each Gaussian to refine its features based on spatial neighbors, preserving fine-grained structural details while preventing influence from distant, unrelated Gaussians.

### [Temporal Mutual Self Attention (TMSA)](20785-temporal-mutual-self-attention-tmsa.md)
Temporal Mutual Self Attention (TMSA) is a transformer module designed for feature alignment, fusion, and refinement across multi-view perspectives or adjacent video frames. It serves as the core component of the Video Restoration Transformer (VRT) network, addressing the problem of aggregating spatio-temporal information by bidirectionally aligning features from adjacent frames and extracting intra-frame context. Unlike standard self-attention which processes a single sequence, TMSA explicitly models inter-frame dependencies through Mutual Attention (MMA) while preserving intra-frame structure via Multi-Head Self-Attention (MSA).

## Operating characteristics
-   **Time Complexity:** The pairwise cosine similarity computation is optimized to a single matrix multiplication $F F^T$, making it efficient for parallelization [S1]. The MLP for weight prediction is lightweight [S1].
-   **Memory Complexity:** The method uses a target volume at 1/8 resolution, which is more compact than full-resolution volumes. The channel dimension is reduced (e.g., from 128 to 16 in the fine stage) for efficiency [S1].
-   **Strengths:** Explicitly models multi-view geometric cues via cosine similarity. Window-based sampling mitigates resolution loss. Efficient aggregation using learned weights [S1].
-   **Limitations:** Requires $K$ input images. The quality depends on the accuracy of the depth distribution entropy for visibility weights [S1].

## Background
The Multi-View Feature Encoder is a key component of the MuRF (Multi-Baseline Radiance Fields) framework [S1]. It was introduced to aggregate essential information for learning feed-forward NeRF models, addressing the challenges of multi-view inconsistency and the need for geometric cues in radiance field prediction [S1]. The method builds upon GMFlow's 2-view Transformer architecture, extending it to handle $K$ views [S1].

## Sources
[S1] MuRF: Multi-Baseline Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.01894, fetched 2026-10-27
