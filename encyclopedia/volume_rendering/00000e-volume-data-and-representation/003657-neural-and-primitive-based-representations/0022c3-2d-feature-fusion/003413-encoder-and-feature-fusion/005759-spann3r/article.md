# Spann3R


## Overview
Spann3R is a feature fusion mechanism designed to predict dense pointmaps in a global coordinate frame directly, thereby avoiding the time-consuming and memory-intensive post-processing alignment step required by DUSt3R when handling more than two images [S1, unofficial]. It sets itself apart from its sibling [Feature Extraction Network](8151-feature-extraction-network.md) and [Attention-based Multi-View Density Prediction](10674-attention-based-multi-view-density-prediction.md) by explicitly maintaining a spatial feature memory to fuse features from a reference decoder and query features from a target decoder, enabling scalable multi-view reconstruction without pairwise alignment overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of image pairs $(I_i, I_j)$, where $I_i$ is the reference image and $I_j$ is the target image.
*   **Output:** Dense pointmaps $(X_{i,g}, X_{j,g})$ in a global coordinate frame.

**Data Structures:**
*   **Spatial Feature Memory:** Stores encoded features from reference decoders.
*   **Training Buffer:** Stores registered images and poses for incremental reconstruction [S1, unofficial].

**Parameters:**
*   $t$: The current training epoch.
*   $H_{tgt}$: Target projection head.
*   $H_{ref}$: Reference projection head.
*   $V_{mem}$: Memory encoder.

**Steps:**
1.  **Feature Encoding:** Encode input images $I_i$ and $I_j$ using a ViT Encoder to obtain features $f_{id}$ (reference) and $f_{jd}$ (target) [S1, unofficial].
2.  **Projection:** Compute key and query features using projection heads:
    $$f_i^K = H_{ref}(f_{id})$$
    $$f_j^Q = H_{tgt}(f_{jd})$$
    [S1, unofficial]
3.  **Memory Encoding:** Encode the pointmaps from the reference decoder into the spatial feature memory:
    $$f_i^V = V_{mem}(X_{i,g})$$
    [S1, unofficial]
4.  **Feature Fusion:** Compute the fused feature $f_t^G$ in the $t$-th training epoch from the spatial feature memory using the key, query, and value features [S1, unofficial].
5.  **Reconstruction:**
    *   The reference decoder inputs the fused feature $f_t^G$ for reconstruction.
    *   The target decoder produces features for querying the memory.
    *   The network outputs the global pointmaps $(X_{i,g}, X_{j,g})$ [S1, unofficial].
6.  **Next Pair Preparation:** Use two additional projection heads to compute key and query features for reconstructing the next image pairs [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spann3r_step(reference_image, target_image, spatial_feature_memory, training_epoch):
    # 1. Encode images
    # ViT Encoder produces features fid and fjd
    fid = vit_encoder(reference_image)
    fjd = vit_encoder(target_image)
    
    # 2. Project features to Key and Query spaces
    # Href and Htgt are projection heads
    key_features = Href(fid)
    query_features = Htgt(fjd)
    
    # 3. Encode reference pointmaps into memory
    # Note: X_i,g is the global pointmap from the previous step or initialization
    # Vmem is the memory encoder
    value_features = Vmem(spatial_feature_memory.get_reference_pointmaps())
    
    # 4. Compute fused feature
    # This step involves attention or fusion logic using key, query, value
    # The exact fusion operation is not detailed in the source, 
    # but it produces f_t^G for the current epoch
    fused_feature = compute_fused_feature(key_features, query_features, value_features)
    
    # 5. Decode pointmaps
    # Reference decoder uses fused feature
    # Target decoder uses query features to query memory
    global_pointmaps = decode_pointmaps(fused_feature, query_features, spatial_feature_memory)
    
    # 6. Update memory and prepare for next batch
    # Additional projection heads compute features for next pairs
    next_key_features = Href(next_fid)
    next_query_features = Htgt(next_fjd)
    
    return global_pointmaps, next_key_features, next_query_features
```

**Edge Cases and Pitfalls:**
*   **GPU Memory Limitation:** Spann3R reconstructs images individually, which is limited to very short frames due to GPU memory limitations [S1, unofficial].
*   **Accuracy:** The 3D points predicted by Spann3R in out-of-distribution scenes may lack accuracy [S1, unofficial].

## Subtopics

### Scene Regressor Network (fSCR)
A neural scene representation that extends Spann3R to predict per-pixel 3D Gaussians in addition to dense pointmaps in a global coordinate frame [S1, unofficial]. It takes an image pair $(I_i, I_j)$ as input and outputs $(X_i, G_i; X_j, G_j)$ [S1, unofficial]. To ensure the covariance matrix $\Sigma$ is positive semi-definite, it scales the matrix $S$ such that $\Sigma_i = R S S^\top R^\top$ [S1, unofficial]. Instead of directly predicting the mean $\mu$, it predicts an offset $\Delta X$ and applies it to the pointmaps: $\mu = X + \Delta X$ [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The method avoids the post-processing alignment step of DUSt3R, reducing time complexity for multi-view scenarios. However, it is limited by GPU memory when handling long frames due to individual image reconstruction [S1, unofficial].
*   **Performance:** Effective for out-of-distribution scenes but may suffer from accuracy issues in such cases [S1, unofficial].
*   **Strengths:** Eliminates the need for time-consuming pairwise alignment post-processing.
*   **Limitations:** Limited to short frames due to GPU memory constraints; potential accuracy loss in out-of-distribution scenes [S1, unofficial].

## Background
Spann3R was proposed as an improvement over DUSt3R, which requires a post-processing step to align pairwise dense pointmaps to a global coordinate frame when handling more than two images [S1, unofficial]. It is used in the context of pose-free 3D Gaussian Splatting and incremental reconstruction pipelines [S1, unofficial].

## Sources
[S1] ZeroGS: Training 3D Gaussian Splatting from Unposed Images, https://doi.org/10.48550/arxiv.2411.15779, fetched 2026-10-26 (unofficial)
