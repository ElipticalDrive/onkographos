# ReSplat Initialization

Also called: 3D Context Aggregation, Initial Gaussian Prediction

## Overview
ReSplat Initialization predicts an initial set of 3D Gaussians from a sparse point cloud by aggregating local and global 3D context to compensate for information loss incurred by downsampling the input point cloud by 16x [S1]. Unlike naive prediction methods that directly map point features to Gaussian parameters, this method uses a transformer-based encoder to enrich features with neighborhood context before decoding [S1]. It sets the stage for the subsequent Recurrent Gaussian Update by providing a structured initial state $\mathcal{G}^0$ [S1].

## How it works
**Inputs:** $N$ input views with intrinsics $K_i$, extrinsics $R_i, t_i$, and images $I_i$ [S1].
**Outputs:** $M$ initial 3D Gaussians, where $M = N \times H \times W / 16$ [S1]. Each Gaussian $j$ is defined by parameters $\mathbf{g}_j^0$ (concatenated mean $\boldsymbol{\mu}$, opacity $\alpha$, covariance $\boldsymbol{\Sigma}$, spherical harmonics coefficients $\mathbf{sh}$) and an initial hidden state $\mathbf{z}_j^0$ [S1].

**Data Structures:**
1.  **Point Cloud:** A set of $M$ points $\{\mathbf{p}_j\}_{j=1}^M$ with associated feature vectors $\{\mathbf{f}_j\}_{j=1}^M \in \mathbb{R}^{C_0}$ [S1].
2.  **Gaussian Parameters:** $\mathbf{g}_j^0 \in \mathbb{R}^{C_1}$, where $C_1 = 59$ [S1]. This vector concatenates $\boldsymbol{\mu}_j$ (3), $\alpha_j$ (1), $\boldsymbol{\Sigma}_j$ (6, using lower-triangular or quaternion representation as per standard GS, though the source specifies the total count), and $\mathbf{sh}_j$ (49 for 3rd degree SH, or adjusted based on implementation; the source fixes $C_1=59$) [S1].
3.  **Hidden State:** $\mathbf{z}_j^0 \in \mathbb{R}^{C_0}$, initialized to the context-aggregated feature $\mathbf{f}_j^*$ [S1].

**Parameters:**
*   $C_0$: Dimension of the hidden state and context-aggregated features. Values: 256 (small model) or 512 (base model) [S1].
*   $C_1$: Total number of parameters per Gaussian. Value: 59 [S1].
*   $k$: Number of neighbors for kNN attention. Value: 16 [S1].
*   $T$: Total number of recurrent iterations (used in subsequent update, not initialization itself, but defines the context of $\mathcal{G}^0$) [S1].

**Steps:**

1.  **Downsampling:**
    Downsample the input data to $M$ points. The source specifies $M = N \times H \times W / 16$ [S1]. This results in a point cloud $\{(\mathbf{p}_j, \mathbf{f}_j)\}_{j=1}^M$ where $\mathbf{f}_j$ are features extracted from input images [S1].

2.  **3D Context Aggregation:**
    Encode 3D context using six alternating blocks of kNN attention and global attention layers [S1].
    *   **kNN Attention:** For each point, identify $k=16$ nearest neighbors in 3D space [S1]. Compute attention weights based on geometric proximity and feature similarity [G].
    *   **Global Attention:** Apply self-attention across all $M$ points to capture long-range dependencies [S1].
    *   **Blocks:** Repeat the kNN and global attention blocks six times [S1].
    *   **Output:** Context-aggregated features $\{\mathbf{f}_j^*\}_{j=1}^M \in \mathbb{R}^{C_0}$ [S1].

3.  **Decoding to Gaussians:**
    Use the point cloud $\{\mathbf{p}_j\}_{j=1}^M$ as the Gaussian centers (mean $\boldsymbol{\mu}_j = \mathbf{p}_j$) [S1]. Decode other Gaussian parameters ($\alpha_j, \boldsymbol{\Sigma}_j, \mathbf{sh}_j$) using a lightweight two-layer Multi-Layer Perceptron (MLP), referred to as the "Gaussian head" [S1].
    *   Input to MLP: $\mathbf{f}_j^*$ [S1].
    *   Output of MLP: Parameters for $\alpha_j, \boldsymbol{\Sigma}_j, \mathbf{sh}_j$ [S1].
    *   Form $\mathbf{g}_j^0$ by concatenating all Gaussian parameters ($\boldsymbol{\mu}_j, \alpha_j, \boldsymbol{\Sigma}_j, \mathbf{sh}_j$) [S1].

4.  **Initialization of Recurrent State:**
    Initialize the hidden state for the recurrent update process: $\mathbf{z}_j^0 = \mathbf{f}_j^*$ [S1].

5.  **Form Initial Set:**
    Construct the initial Gaussian set:
    $$ \mathcal{G}^0 = \{(\mathbf{g}_j^0, \mathbf{z}_j^0)\}_{j=1}^M \quad (3) $$
    [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def resplat_initialization(images, intrinsics, extrinsics, H, W):
    """
    Predicts initial 3D Gaussians from input views.
    
    Args:
        images: List of N images.
        intrinsics: List of N intrinsic matrices K.
        extrinsics: List of N extrinsic matrices [R|t].
        H, W: Height and width of input images.
        
    Returns:
        G0: List of M initial Gaussians, each tuple (g0, z0).
    """
    N = len(images)
    M = (N * H * W) // 16
    
    # Step 1: Downsampling and Feature Extraction
    # [G] Projects image pixels into 3D space using camera parameters,
    # samples M points, and extracts initial features.
    points, features = _downsample_and_extract_features(images, intrinsics, extrinsics, M)

    # Step 2: 3D Context Aggregation
    k = 16
    C0 = 256  # or 512 depending on model size
    aggregated_features = features
    for _ in range(6):
        # kNN Attention Block
        # [G] Standard kNN attention: compute Q, K, V from features,
        # restrict attention to k neighbors, apply softmax, and aggregate.
        # Implementation details are not provided in [S1].
        aggregated_features = _knn_attention(aggregated_features, points, k)
        # Global Attention Block
        # [G] Standard self-attention: compute Q, K, V from all features,
        # apply softmax to all pairs, and aggregate.
        # Implementation details are not provided in [S1].
        aggregated_features = _global_attention(aggregated_features)

    # Step 3: Decoding to Gaussians
    C1 = 59
    gaussian_head = _TwoLayerMLP(input_dim=C0, output_dim=C1 - 3)  # 3 for mean which is points

    initial_gaussians = []
    for j in range(M):
        p_j = points[j]
        f_star_j = aggregated_features[j]

        # Mean is the point position
        mu_j = p_j

        # Decode other parameters from context-aggregated feature
        decoded_params = gaussian_head(f_star_j)

        # Concatenate to form g0_j: mu, alpha, Sigma, sh
        g0_j = np.concatenate([mu_j, decoded_params])

        # Initialize hidden state z0_j
        z0_j = f_star_j

        initial_gaussians.append((g0_j, z0_j))

    return initial_gaussians

def _downsample_and_extract_features(images, intrinsics, extrinsics, M):
    # [G] Standard multi-view geometry projection and sampling to obtain M points and features.
    # Implementation details are not provided in [S1].
    pass

def _knn_attention(features, points, k):
    # [G] Standard kNN attention: compute Q, K, V from features,
    # restrict attention to k neighbors, apply softmax, and aggregate.
    # Implementation details are not provided in [S1].
    pass

def _global_attention(features):
    # [G] Standard self-attention: compute Q, K, V from all features,
    # apply softmax to all pairs, and aggregate.
    # Implementation details are not provided in [S1].
    pass
```

**Edge Cases and Pitfalls:**
*   **Downsampling Loss:** Naïve prediction from downsampled points leads to considerable performance loss; context aggregation is critical [S1].
*   **Precision:** The source does not specify floating-point precision requirements, but standard PyTorch implementations typically use FP32 or FP16 [G].
*   **Boundary Conditions:** The kNN search must handle points near the boundary of the point cloud correctly. [G] Standard kNN implementations handle this by considering only existing points.

## Operating characteristics
*   **Time Complexity:** Dominated by the six blocks of kNN and global attention. kNN attention is $O(M \cdot k \cdot C_0)$ per block, and global attention is $O(M^2 \cdot C_0)$ per block [G]. With $M$ being $1/16$ of the total pixel count, this is significantly faster than full-resolution attention but still quadratic in $M$ for the global attention component [G].
*   **Memory Complexity:** $O(M \cdot C_0)$ to store features and hidden states [G]. The Gaussian head is lightweight [S1].
*   **Performance:** The method compensates for 16x downsampling, allowing for efficient initialization while maintaining high-quality results [S1].
*   **Strengths:** Efficiently initializes Gaussians from sparse data; leverages 3D context for better feature representation [S1].
*   **Limitations:** Requires a pre-computed point cloud (e.g., from COLMAP); the global attention component can be memory-intensive for very large scenes [G].

## Background
ReSplat Initialization was introduced in "ReSplat: Learning Recurrent Gaussian Splatting" [S1]. It addresses the challenge of initializing 3D Gaussian Splatting from sparse point clouds, which is common in multi-view stereo pipelines [S1]. The method builds on the idea that local and global 3D context can be aggregated to recover information lost during downsampling [S1]. It is designed to work with the Recurrent Gaussian Update module of the ReSplat framework [S1].

## Sources
[S1] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
