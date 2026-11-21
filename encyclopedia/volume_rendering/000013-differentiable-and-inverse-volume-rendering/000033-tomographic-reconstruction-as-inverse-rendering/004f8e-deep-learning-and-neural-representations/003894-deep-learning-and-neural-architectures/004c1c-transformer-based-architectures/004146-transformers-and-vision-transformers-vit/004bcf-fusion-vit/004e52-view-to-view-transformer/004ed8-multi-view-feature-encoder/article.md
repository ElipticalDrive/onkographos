# Multi-View Feature Encoder

Also called: MuRF Multi-View Feature Encoder

## Overview
The Multi-View Feature Encoder aggregates essential appearance and geometric information from $K$ input images to serve as the input for feed-forward Neural Radiance Field (NeRF) models [S1]. It solves the problem of multi-view inconsistency and information loss during downsampling by extracting features via a weight-sharing 2D CNN and a joint multi-view Transformer, then aggregating them using visibility-weighted cosine similarities [S1]. It sets itself apart from sibling methods by explicitly modeling multi-view consistency through pairwise cosine similarities and using a learned weighted average for aggregation, rather than simple averaging or attention [S1].

## How it works
**Inputs:** $K$ input images $\{I_k\}_{k=1}^K$, camera poses for each view, and a target 3D point cloud or volume grid defined by spatial dimensions $H/8 \times W/8 \times D$.
**Outputs:** A target volume $z \in \mathbb{R}^{\frac{H}{8} \times \frac{W}{8} \times D \times C}$ encoding appearance and geometry [S1].

**Data Structures:**
1.  **Feature Maps:** Extracted at 1/2, 1/4, and 1/8 resolutions. The encoder uses the 1/8 resolution features for the Transformer stage [S1].
2.  **Windowed Color Vector:** For each 3D point, a $9 \times 9$ window is sampled from each of the $K$ feature maps centered at the 2D projection of the point. The colors within this window are concatenated to form $\tilde{c}_k$ for view $k$ [S1].
3.  **Feature Matrix:** Sampled features $\{f_i\}_{i=1}^K$ are collected into a matrix of shape $K \times M$, where $M$ is the feature dimension [S1].

**Steps:**

1.  **2D Feature Extraction:**
    Pass each input image $I_k$ through a weight-sharing 2D CNN. The CNN consists of 6 residual blocks [12]. Every 2 residual blocks include a stride-2 convolutional layer to achieve 2× downsampling. This produces feature maps at 1/2, 1/4, and 1/8 resolutions [S1].

2.  **Multi-View Transformer:**
    Feed the 1/8 resolution convolutional features into a joint multi-view Transformer. This Transformer extends the 2-view cross-attention architecture (from GMFlow [34, 35]) to $K$ input views. The output is the refined feature maps at 1/8 resolution [S1].

3.  **Sampling:**
    For each 3D point in the target volume:
    *   Project the 3D point to 2D coordinates in each of the $K$ input views.
    *   Sample a $9 \times 9$ window centered at these 2D coordinates from the 1/8 resolution feature maps.
    *   Concatenate the pixel values within the window to obtain the color vector $\tilde{c}_k$ for each view $k$ [S1].
    *   Sample the feature vector $f_k$ at the center of the window for each view $k$ [S1].

4.  **Compute Pair-wise Cosine Similarities:**
    Quantify multi-view consistency by computing pair-wise cosine similarities between the sampled features $\{f_i\}_{i=1}^K$.
    *   Normalize all feature vectors $f_i$ to unit length.
    *   Collect them into a matrix $F \in \mathbb{R}^{K \times M}$.
    *   Compute the Gram matrix $G = F F^T$.
    *   Extract the upper triangle of $G$ to obtain all $K(K-1)/2$ pair-wise cosine similarities $\cos(f_i, f_j)$ for $i < j$ [S1].
    *   *Note:* In practice, group-wise cosine similarities are used for improved expressiveness, but the matrix multiplication approach remains valid [S1].

5.  **Aggregate Cosine Similarities:**
    Compute the aggregated cosine similarity $\hat{s}$ using a learned weighted average based on visibility:
    $$ \hat{s} = \sum_{i<j} w_{ij} \cos(f_i, f_j) $$
    where $w_{ij}$ are normalized visibility weights learned from the entropy of the depth distribution on each ray [S1].

6.  **Aggregate Colors and Features:**
    Compute the aggregated color $\hat{c}$ and aggregated feature $\hat{f}$ using a learned weighted average:
    $$ \hat{c} = \sum_{i=1}^K w_i \tilde{c}_i $$
    $$ \hat{f} = \sum_{i=1}^K w_i f_i $$
    The weights $w_i$ are normalized learned weights. A small MLP network predicts $w_i$ from the concatenation of the sampled features and the view direction difference between the source and target views [S1].

7.  **Volume Projection:**
    Concatenate the aggregated cosine similarity $\hat{s}$, the aggregated color $\hat{c}$, and the aggregated feature $\hat{f}$.
    Project this concatenated vector to a $C$-dimensional vector using a linear layer.
    The result is the target volume $z \in \mathbb{R}^{\frac{H}{8} \times \frac{W}{8} \times D \times C}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MultiViewFeatureEncoder(nn.Module):
    def __init__(self, num_views, feature_dim, hidden_dim, num_residual_blocks=6):
        super().__init__()
        self.num_views = num_views
        self.feature_dim = feature_dim
        
        # 2D CNN for per-image feature extraction
        # Consists of 6 residual blocks, with stride-2 conv every 2 blocks
        self.cnn = self._build_cnn(num_residual_blocks, feature_dim)
        
        # Joint Multi-View Transformer (simplified as a placeholder for GMFlow-based architecture)
        # Extends 2-view cross-attention to K views
        self.transformer = nn.TransformerEncoder(
            nn.TransformerEncoderLayer(d_model=feature_dim, nhead=8),
            num_layers=2
        )
        
        # MLP for predicting aggregation weights
        self.weight_mlp = nn.Sequential(
            nn.Linear(feature_dim * 2 + 3, hidden_dim), # features + view direction diff
            nn.ReLU(),
            nn.Linear(hidden_dim, 1)
        )
        
        # Linear projection for final volume
        self.projection = nn.Linear(feature_dim * 2 + 1, feature_dim) # s, c, f concatenated

    def _build_cnn(self, num_blocks, out_dim):
        # Simplified CNN structure
        layers = []
        # Initial conv
        layers.append(nn.Conv2d(3, 64, 7, stride=2, padding=3))
        layers.append(nn.ReLU())
        
        for i in range(num_blocks):
            # Residual block logic
            # Every 2 blocks, add stride-2 conv for downsampling
            if i % 2 == 0 and i > 0:
                layers.append(nn.Conv2d(64, 64, 3, stride=2, padding=1))
            layers.append(nn.Conv2d(64, 64, 3, padding=1))
            layers.append(nn.ReLU())
            
        layers.append(nn.Conv2d(64, out_dim, 1))
        return nn.Sequential(*layers)

    def forward(self, images, camera_poses, target_points_3d):
        """
        images: (K, H, W, 3)
        camera_poses: (K, 4, 4) or similar projection matrix
        target_points_3d: (N, 3) in world coordinates
        """
        K = self.num_views
        
        # Step 1: Extract features using weight-sharing 2D CNN
        # images shape: (K, H, W, 3) -> permute to (K, 3, H, W)
        img_tensor = images.permute(0, 3, 1, 2)
        features_1_8 = self.cnn(img_tensor) # (K, C, H/8, W/8)
        
        # Step 2: Joint Multi-View Transformer
        # Reshape for transformer: (K, H/8 * W/8, C)
        B, C, H_f, W_f = features_1_8.shape
        features_flat = features_1_8.permute(0, 2, 3, 1).reshape(K, -1, C)
        
        # Apply transformer (simplified attention mechanism)
        # In reality, this involves cross-attention between views
        transformed_features = self.transformer(features_flat) # (K, H/8*W/8, C)
        transformed_features = transformed_features.reshape(K, H_f, W_f, C)
        
        # Step 3: Sampling
        # Project 3D points to 2D for each view
        # Assuming a function to project 3D points to 2D image coordinates
        projected_points = self.project_3d_to_2d(target_points_3d, camera_poses) # (N, K, 2)
        
        # Sample 9x9 windows for colors and features
        # This requires gathering patches from the feature maps
        # For simplicity, we assume a function that returns windowed features
        windowed_colors, sampled_features = self.sample_windows(
            transformed_features, projected_points, window_size=9
        )
        # windowed_colors: (N, K, 9*9*3)
        # sampled_features: (N, K, C)
        
        # Step 4: Compute Pair-wise Cosine Similarities
        # Normalize features
        norm_features = F.normalize(sampled_features, p=2, dim=-1) # (N, K, C)
        
        # Compute Gram matrix for each point
        # (N, K, C) @ (N, C, K) -> (N, K, K)
        gram_matrix = torch.bmm(norm_features, norm_features.transpose(1, 2))
        
        # Extract upper triangle
        # Create mask for upper triangle
        mask = torch.triu(torch.ones(K, K), diagonal=1).bool()
        # Apply mask to get similarities
        similarities = gram_matrix.masked_select(mask.unsqueeze(0).expand(N, -1, -1)) # (N, K*(K-1)/2)
        
        # Step 5: Aggregate Cosine Similarities
        # Compute visibility weights w_ij (simplified as uniform for demonstration)
        # In practice, w_ij comes from depth distribution entropy
        w_ij = torch.ones_like(similarities) / similarities.shape[1]
        aggregated_similarity = torch.sum(similarities * w_ij, dim=-1, keepdim=True) # (N, 1)
        
        # Step 6: Aggregate Colors and Features
        # Compute view direction differences (simplified)
        view_diff = self.compute_view_direction_diff(projected_points, camera_poses) # (N, K, 3)
        
        # Predict weights w_i
        # Concatenate sampled features and view direction difference
        # sampled_features: (N, K, C), view_diff: (N, K, 3)
        # Repeat view_diff to match feature dim? Or concat along feature dim?
        # Source says: "concatenation of sampled features and view direction difference"
        # Likely per-view: (N, K, C+3)
        input_for_mlp = torch.cat([sampled_features, view_diff], dim=-1)
        
        # MLP predicts scalar weight per view
        raw_weights = self.weight_mlp(input_for_mlp) # (N, K, 1)
        w_i = F.softmax(raw_weights, dim=1) # (N, K, 1)
        
        # Aggregate
        windowed_colors_flat = windowed_colors.reshape(N, K, -1)
        aggregated_color = torch.sum(windowed_colors_flat * w_i, dim=1) # (N, 9*9*3)
        aggregated_feature = torch.sum(sampled_features * w_i, dim=1) # (N, C)
        
        # Step 7: Volume Projection
        # Concatenate s, c, f
        # aggregated_similarity: (N, 1), aggregated_color: (N, 81*3), aggregated_feature: (N, C)
        combined = torch.cat([aggregated_similarity, aggregated_color, aggregated_feature], dim=-1)
        
        # Project to C-dimensional vector
        volume_features = self.projection(combined) # (N, C)
        
        # Reshape to volume grid (N, H/8, W/8, D, C)
        # Assuming target_points_3d are structured in a grid
        volume = volume_features.reshape(-1, H_f, W_f, D, C)
        
        return volume

    def project_3d_to_2d(self, points_3d, poses):
        # Placeholder for projection logic
        pass

    def sample_windows(self, features, projected_points, window_size):
        # Placeholder for window sampling logic
        pass

    def compute_view_direction_diff(self, projected_points, poses):
        # Placeholder for view direction difference calculation
        pass
```

## Subtopics

### [3D Transformer Network Refinement](14648-3d-transformer-network-refinement.md)
The 3D Transformer Network Refinement (also known as the Point Transformer Network) is a module within the PointSplat framework that optimizes the appearance and geometric attributes of selected 3D Gaussians. It addresses the feature imbalance caused by high-dimensional spherical harmonic coefficients overshadowing geometric factors, which leads to biased learning. By leveraging local attention mechanisms on sparse 3D points, it preserves fine-grained structural details and prevents influence from distant, unrelated Gaussians. It builds on [Multi-View Feature Encoder](13802-multi-view-feature-encoder.md) by refining the encoded features through a hierarchical point transformer architecture.

## Operating characteristics
**Time Complexity:** The computation of pair-wise cosine similarities is $O(K^2 \cdot M)$ per point, where $K$ is the number of views and $M$ is the feature dimension. The matrix multiplication approach allows this to be computed in parallel efficiently [S1]. The MLP for weight prediction is $O(K \cdot (M + 3))$ per point [S1].
**Memory Complexity:** The method requires storing feature maps at 1/8 resolution for all $K$ views, and the target volume $z \in \mathbb{R}^{\frac{H}{8} \times \frac{W}{8} \times D \times C}$ [S1]. The use of 1/8 resolution features reduces memory compared to full-resolution sampling [S1].
**Measured Performance:** The method is designed to be memory and parameter efficient compared to 3D CNNs while maintaining similar performance [S1]. The fine stage of the associated MuRF model uses a compact volume with reduced channel dimension (16) and fewer sampling points (16) for efficiency [S1].
**Strengths:** Explicitly models multi-view consistency via cosine similarities, handles small and large baseline views effectively, and is more efficient than 3D CNNs [S1].
**Limitations:** Relies on the quality of the 2D CNN and Transformer features. The windowed sampling might lose some information if the 8× subsampling is too aggressive, though the 9×9 window mitigates this [S1].

## Background
The Multi-View Feature Encoder is a key component of the MuRF (Multi-Baseline Radiance Fields) framework, introduced in 2024 [S1]. It builds on the observation that multi-view features of a surface point tend to exhibit high multi-view consistency, which can be used to provide geometric cues for volume density prediction [S1]. The method extends the 2-view cross-attention architecture from GMFlow to $K$ views [S1].

## Sources
[S1] MuRF: Multi-Baseline Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.01894, fetched 2026-10-27
