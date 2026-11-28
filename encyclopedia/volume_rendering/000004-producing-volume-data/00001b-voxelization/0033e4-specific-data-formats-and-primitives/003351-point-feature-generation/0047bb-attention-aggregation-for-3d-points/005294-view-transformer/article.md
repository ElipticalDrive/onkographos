# View Transformer


## Overview
The View Transformer is a module that aggregates multi-view features into a single projection feature to handle occlusions and ensure view consistency without assuming an order of source views [S1][S2, unofficial]. It addresses the limitation of mean/variance aggregation, which treats all views equally and fails to account for visibility and occlusion [S2, unofficial]. By using self-attention, the model reasons about consistency across multiple views to produce a robust feature representation for a query point [S2, unofficial]. It sets itself apart from simple statistical aggregation by learning to weight views based on visual consistency rather than fixed rules [S2, unofficial].

## How it works
**Inputs:**
- A set of $N$ source views [S1][S2, unofficial].
- For a specific pixel in a reference view, a set of $M$ 3D points along the emitted ray: $\{p(t) = o + tv, t \ge 0\}$ [S1][S2, unofficial].
- Feature maps for each source view, containing color and feature channels [S1][S2, unofficial].

**Outputs:**
- A single projection feature $\mathbf{f}_p$ [S1][S2, unofficial].
- Updated multi-view features $\{\mathbf{f}'_i\}_{i=1}^N$ [S1][S2, unofficial].

**Data Structures:**
- $\mathbf{f}_0$: A learnable aggregation token, a vector of the same dimension as the view features [S1][S2, unofficial].
- $\{\mathbf{c}_i\}_{i=1}^N$: Extracted colors from source views [S1][S2, unofficial].
- $\{\mathbf{f}_i\}_{i=1}^N$: Extracted features from source views [S1][S2, unofficial].

**Steps:**
1. **Feature Extraction:** For each point $p$ on the ray and each source view $i \in \{1, \dots, N\}$, project $p$ onto the feature map of view $i$ and extract the color $\mathbf{c}_i$ and feature vector $\mathbf{f}_i$ using bilinear interpolation [S1][S2, unofficial].
2. **Token Initialization:** Initialize a learnable aggregation token $\mathbf{f}_0$ [S1][S2, unofficial].
3. **Attention Aggregation:** Apply a self-attention transformer with linear attention to the set $\{\mathbf{f}_0, \mathbf{f}_1, \dots, \mathbf{f}_N\}$ [S1][S2, unofficial].
   - **Constraint:** Do not use positional encoding, as no order of source views is assumed [S1][S2, unofficial].
4. **Output Generation:** The transformer computes the projection feature $\mathbf{f}_p$ (derived from the updated token) and the updated multi-view features $\{\mathbf{f}'_i\}_{i=1}^N$ [S1][S2, unofficial].

**Formula:**
$$
\mathbf{f}_p, \{\mathbf{f}'_i\}_{i=1}^N = \text{ViewTrans}(\mathbf{f}_0, \{\mathbf{f}_i\}_{i=1}^N)
$$
Where $\text{ViewTrans}$ denotes the self-attention transformer with linear attention [S1][S2, unofficial].

**Parameters:**
- $N$: Number of source views [S1][S2, unofficial].
- $M$: Number of points on the ray (used for context, but the transformer operates on the $N$ views per point) [S1][S2, unofficial].
- Positional Encoding: None [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def view_transformer(uv_coords, feature_maps, num_views, token_dim):
    """
    Aggregates multi-view features for points on a ray.
    
    Args:
        uv_coords: Array of shape (M, N, 2) with normalized (u, v) coordinates for each point and view.
        feature_maps: List of N feature maps, each of shape (H, W, C).
        num_views: Number of source views N.
        token_dim: Dimension of the aggregation token.
        
    Returns:
        projection_features: Array of shape (M, C), the aggregated feature fp for each point.
        updated_features: Array of shape (M, N, C), the updated multi-view features f'i.
    """
    M, N, _ = uv_coords.shape
    C = feature_maps[0].shape[2]
    
    # [G] In practice, f0 is a learnable parameter vector initialized randomly or with zeros.
    f0 = np.zeros(token_dim)
    
    projection_features = np.zeros((M, C))
    updated_features = np.zeros((M, N, C))
    
    for m in range(M):
        view_features = []
        
        for i in range(N):
            u, v = uv_coords[m, i]
            # Extract feature fi using bilinear interpolation
            fi = bilinear_interpolate(feature_maps[i], u, v)
            view_features.append(fi)
            
        # Concatenate f0 and all fi into sequence of shape (N+1, C)
        sequence = np.concatenate([f0.reshape(1, -1)] + [f.reshape(1, -1) for f in view_features], axis=0)
        
        # Apply self-attention transformer with linear attention, no positional encoding
        # [G] Source specifies linear attention [19] but does not provide the exact kernel formula.
        # Standard scaled dot-product attention is shown as a functional equivalent for aggregation.
        output_sequence = self_attention_no_pos_enc(sequence)
        
        # First output corresponds to updated token -> projection feature fp
        projection_features[m] = output_sequence[0]
        # Remaining outputs correspond to updated multi-view features f'i
        for i in range(N):
            updated_features[m, i] = output_sequence[i + 1]
            
    return projection_features, updated_features

def bilinear_interpolate(feature_map, u, v):
    # [G] Standard bilinear interpolation on feature map at normalized coordinates (u, v)
    h, w, c = feature_map.shape
    u_idx = u * (w - 1)
    v_idx = v * (h - 1)
    u0, u1 = int(math.floor(u_idx)), min(int(math.ceil(u_idx)), w - 1)
    v0, v1 = int(math.floor(v_idx)), min(int(math.ceil(v_idx)), h - 1)
    w_u = u_idx - u0
    w_v = v_idx - v0
    top = feature_map[v0, u0] * (1 - w_u) + feature_map[v0, u1] * w_u
    bottom = feature_map[v1, u0] * (1 - w_u) + feature_map[v1, u1] * w_u
    return top * (1 - w_v) + bottom * w_v

def self_attention_no_pos_enc(sequence):
    # sequence shape: (L, C)
    L, C = sequence.shape
    d = C
    # [G] Wq, Wk, Wv are learnable weight matrices of shape (C, C)
    Wq = np.random.randn(C, C)
    Wk = np.random.randn(C, C)
    Wv = np.random.randn(C, C)
    
    Q = sequence @ Wq
    K = sequence @ Wk
    V = sequence @ Wv
    
    scores = Q @ K.T / math.sqrt(d)
    # Softmax over last dimension
    exp_scores = np.exp(scores - np.max(scores, axis=-1, keepdims=True))
    attn_weights = exp_scores / np.sum(exp_scores, axis=-1, keepdims=True)
    
    out = attn_weights @ V
    return out
```

**Edge Cases and Pitfalls:**
- **Occlusion:** The transformer handles occlusions where some views are invalid for a point by learning to down-weight inconsistent views [S2, unofficial].
- **Order Independence:** Positional encoding must not be used, as the source views are unordered [S1][S2, unofficial].
- **Precision:** Bilinear interpolation is used for feature extraction [S1][S2, unofficial].

## Subtopics

### [Field-Aggregation Transformer](15112-field-aggregation-transformer.md)
The Field-Aggregation Transformer (FAT) is a shared attention mechanism that co-aggregates radiance and semantic-embedding fields by combining features from multiple reference views for a 3D point. It builds on [View Transformer](14756-view-transformer.md) by explicitly handling two distinct feature types (RGB and semantic) within a single aggregation step, ensuring that the aggregation weights derived from low-level visual features are consistently applied to high-level semantic features. Unlike standard view transformers that may treat views independently or sequentially, FAT enforces joint optimization of geometry (via RGB) and semantics (via semantic embeddings) through a unified attention structure.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1][S2, unofficial].
- **Memory Complexity:** Not given in the sources [S1][S2, unofficial].
- **Strengths:** Handles occlusions robustly by learning view consistency; does not assume an order of views; generalizable to arbitrary input counts [S1][S2, unofficial].
- **Limitations:** Requires learning the aggregation weights; may be computationally more expensive than simple mean/variance aggregation [S2, unofficial].
- **When to use:** When multi-view features need to be aggregated for a 3D point in the presence of occlusions and without a fixed view order [S1][S2, unofficial].
- **When not to use:** When computational resources are extremely limited and occlusion handling is not critical [S2, unofficial].

## Background
The View Transformer was introduced in the VolRecon paper for generalizable multi-view reconstruction [S1][S2, unofficial]. It builds on previous work using learnable tokens for aggregation [S1][S2, unofficial]. The use of linear attention is cited from [19] (Transformers are RNNs) [S1][S2, unofficial].

## Sources
[S1] VolRecon: Volume Rendering of Signed Ray Distance Functions for Generalizable Multi-View Reconstruction, https://doi.org/10.1109/cvpr52729.2023.01601, fetched 2026-10-26
[S2] [PDF] VolRecon: Volume Rendering of Signed Ray Distance Functions for ..., https://openaccess.thecvf.com/content/CVPR2023/papers/Ren_VolRecon_Volume_Rendering_of_Signed_Ray_Distance_Functions_for_Generalizable_CVPR_2023_paper.pdf, fetched 2026-10-26 (unofficial)
