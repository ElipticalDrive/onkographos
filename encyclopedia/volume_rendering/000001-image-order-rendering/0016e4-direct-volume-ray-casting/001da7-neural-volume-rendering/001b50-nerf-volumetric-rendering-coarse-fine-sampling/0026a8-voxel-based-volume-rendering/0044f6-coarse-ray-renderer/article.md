# Coarse Ray Renderer


## Overview
The Coarse Ray Renderer (CRR) is a fast rendering technique that provides coarse density estimation and ray features for downstream processes in neural volume rendering, specifically serving the Fine Voxel Renderer (FVR) and Progressive Blur Estimation (PBE). It differs from standard per-voxel ray casting by aggregating features along a ray and inferring color once per ray rather than querying every voxel, which significantly reduces computational cost at the expense of per-voxel accuracy [S1][S2, unofficial]. It builds on [Voxel-based Volume Rendering](3845-voxel-based-volume-rendering.md) by utilizing a decomposed explicit feature volume.

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $r_x$ defined by origin and direction, and the ray direction vector $d$.
*   **Output:** Coarse color $\hat{C}_{coar}(r_x)$ and coarse density $\sigma_{coar}$ (used by FVR).

**Data Structures:**
*   **Explicit Feature Volume ($G_{coar}$):** A decomposed tensor representing 17 million voxels [S1][S2, unofficial].
    *   **Layout:** Tensor decomposition with channel dimensions $\{64, 16, 16\}$ for the $\{X, Y, Z\}$ axes respectively [S1][S2, unofficial].
    *   **Implementation:** Implemented using tensor decomposition (e.g., TensoRF-style) [S1][S2, unofficial].
*   **MLPs:**
    *   $M_{\sigma}^{coar}$: Two-layer MLP for density estimation. Hidden layer channel size: 64 [S1][S2, unofficial].
    *   $M_{r}^{coar}$: Three-layer MLP for color rendering. Hidden layer channel size: 64 [S1][S2, unofficial].

**Parameters:**
*   **Number of coarse samples ($N_c$):** 64 equidistant points along the ray [S1][S2, unofficial].
*   **Gamma correction:** $\hat{C}(r_x)^{1/2.2}$ is applied as the camera response function [S1][S2, unofficial].

**Steps:**
1.  **Ray Tracing:** Trace $N_c$ equidistant points $X_i$ along the ray $r_x$ [S1][S2, unofficial].
2.  **Feature Aggregation:** For each point $X_i$, query the explicit feature volume $G_{coar}$ to obtain local features. Aggregate these features into a single ray feature vector $f_{ray}(r_x)$ [S1][S2, unofficial]. The aggregation method is not explicitly detailed in the provided notes, but standard practice in decomposed volumes involves interpolation and concatenation or summation of the queried features [G].
3.  **Density Estimation:** Compute the density $\sigma_i^{coar}$ for each sample point $X_i$ using the density MLP:
    $$ \sigma_i^{coar} = M_{\sigma}^{coar}(\gamma_L(X_i), G_{coar}(X_i)) $$
    where $\gamma_L$ is a positional encoding function and $G_{coar}(X_i)$ are the features retrieved from the volume [S1][S2, unofficial].
4.  **Color Rendering:** Compute the coarse color $\hat{C}_{coar}(r_x)$ by passing the aggregated ray feature $f_{ray}(r_x)$ and the ray direction $d$ (optionally encoded via $\gamma_L$) through the color MLP:
    $$ \hat{C}_{coar}(r_x) = M_{r}^{coar}(f_{ray}(r_x), \gamma_L(d)) $$
    [S1][S2, unofficial]. Note: This inference happens once per ray, not per voxel [S1].
5.  **Blur Aggregation (for PBE):** If used for Progressive Blur Estimation, the coarse color is aggregated with blur kernel weights $h(x_k^*)$ at refined kernel locations $x_k^*$:
    $$ \tilde{\hat{C}}_{coar}(r_x) = \sum_{x_k^*} \hat{C}_{coar}(r_{x_k^*}) h(x_k^*) $$
    [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_ray_renderer(ray_origin, ray_direction, G_coar, M_sigma_coar, M_r_coar, Nc=64):
    """
    Renders a coarse color and density for a single ray.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        G_coar: Decomposed feature volume object supporting query(X).
        M_sigma_coar: Two-layer MLP for density (hidden size 64).
        M_r_coar: Three-layer MLP for color (hidden size 64).
        Nc: Number of coarse samples (64).
        
    Returns:
        color: 3D vector, coarse color of the ray.
        densities: List of Nc floats, densities for importance sampling in FVR.
    """
    # 1. Sample Nc equidistant points along the ray
    # Assuming a standard ray segment [t_min, t_max] is defined by the scene bounds
    # or passed externally. Here we assume a helper `sample_equidistant` exists.
    points = sample_equidistant(ray_origin, ray_direction, Nc)
    
    densities = []
    ray_features = []
    
    for point in points:
        # 2. Query feature volume
        # G_coar.query returns features based on tensor decomposition interpolation
        features = G_coar.query(point)
        
        # 3. Estimate density
        # gamma_L is positional encoding
        sigma = M_sigma_coar(concat(positional_encode(point), features))
        densities.append(sigma)
        
        # 4. Aggregate features for color rendering
        # Standard aggregation in such volumes is often concatenation or sum.
        # Given M_r_coar takes ray features, we accumulate them.
        # Note: The source implies a single ray feature fray(rx) is used.
        ray_features.append(features)
        
    # Aggregate ray features into a single vector fray(rx)
    # Implementation detail: Typically concatenation of all features or a learned pooling.
    # Given the MLP input size, we assume concatenation or a fixed-size aggregation.
    # If Nc=64 and channel is 64, concatenation would be large. 
    # However, source says "fray(rx)" is the input. 
    # We assume a standard aggregation like mean or concatenation followed by projection.
    # For strict adherence to "no placeholders", we assume a simple mean or concatenation 
    # if the MLP input dimension matches. 
    # Let's assume concatenation of features from all Nc points is too large for a 64-channel MLP 
    # unless projected. The source doesn't specify the aggregation operator. 
    # [G] We assume a simple concatenation of the features from the Nc points, 
    # or that the MLP handles the variable length via a specific architecture not detailed.
    # To be implementable: We assume the features are aggregated via concatenation 
    # and passed to the MLP, or the MLP is designed for the aggregated size.
    # Given M_r_coar is 3-layer, 64 hidden, it likely expects a fixed input size.
    # Let's assume the "ray feature" is the concatenation of features from the Nc points.
    # If the channel dims are {64,16,16}, the feature dimension at a point is 64*16*16? 
    # No, TensoRF decomposes into factors. The feature vector at a point is the product/sum of factors.
    # Usually, the feature dimension is the rank sum or product. 
    # Source says channel dimensions for axes are {64, 16, 16}. 
    # This usually means the feature vector size is 64+16+16 = 96 or 64*16*16.
    # Let's assume the feature vector size is 64 (matching the MLP hidden size hint) or derived from the decomposition.
    # We will use a placeholder comment for the specific aggregation math if not given, 
    # but the source says "fray(rx)" is the input.
    
    # [G] Assuming standard TensoRF aggregation: feature = sum of factor products.
    # Let's assume the resulting feature vector size is F.
    # M_r_coar takes fray(rx) and direction.
    
    aggregated_feature = aggregate_features(ray_features) # [G] Specific aggregation not detailed, assumed concatenation or sum.
    
    # 5. Render Color
    color = M_r_coar(concat(aggregated_feature, positional_encode(ray_direction)))
    
    return color, densities

def aggregate_features(features_list):
    # [G] The source does not specify how Nc features are combined into one fray(rx).
    # Common approaches: Concatenation, Mean, or Max pooling.
    # Given the MLP size, Mean pooling is a safe default for "ray feature".
    return mean(features_list, axis=0)
```

**Edge Cases and Pitfalls:**
*   **Accuracy vs. Speed:** CRR infers color once per ray, which is faster but less accurate than per-voxel rendering [S1]. Multi-view observations constrain the ray features to coarsely represent surface features [S1][S2, unofficial].
*   **Blur Handling:** When used with PBE, CRR renders color based on refined kernel locations. The photometric loss combines terms for both coarse and refined blur estimates [S1][S2, unofficial].

## Subtopics

### Fine Voxel Renderer
A renderer that uses importance sampling based on CRR's density PDF to sample additional points in high-density regions for high-quality view synthesis [S2, unofficial]. It differs from CRR by performing per-voxel rendering with a larger MLP and feature volume [S2, unofficial]. Choose it when high-quality rendering is required after the coarse stage [S2, unofficial]. Link: [Fine Voxel Renderer](3845-voxel-based-volume-rendering.md#fine-voxel-renderer) (Note: No separate article exists, so it is described here).
*   **Definition:** Uses importance sampling based on CRR's density PDF to sample additional points in high-density regions for high-quality view synthesis [S2, unofficial].
*   **Steps:**
    1.  Calculate weights $w_i = T_i^{coar} (1 - \exp(-\sigma_i^{coar} \delta_i))$ for $N_c$ equidistant points from CRR [S2, unofficial].
    2.  Normalize these weights to produce a PDF of ray density [S2, unofficial].
    3.  Use importance sampling to select $N_f$ additional points on high-density regions for finer tracing [S2, unofficial].
    4.  Compute per-voxel color $c_i$ using MLP $M_{fine\_c}$ with feature $f_{X_i}$ and direction $d$ [S2, unofficial].
*   **Formula:** $\tilde{\hat{C}}_{fine}(r_x) = \sum_{x_i} \hat{C}_{fine}(r_{x_i}) h(x_i)$, summing over refined kernel locations $x_i$ with weights $h$ [S2, unofficial].
*   **Parameters:**
    *   Number of fine samples $N_f$: 64 [S2, unofficial].
    *   Explicit feature volume $G_{fine}$: Represents 134 million voxels using tensor decomposition with channel dimensions $\{64, 16, 16\}$ for $\{X, Y, Z\}$ axes [S2, unofficial].
    *   MLP $M_{fine\_\sigma}$: Two-layer MLP with channel size 256 for hidden layers [S2, unofficial].
    *   MLP $M_{fine\_c}$: Three-layer MLP with channel size 256 for hidden layers [S2, unofficial].
*   **Pitfalls:** Voxel pruning is avoided because PBE dynamically generates rays based on gradients; pruning eliminates gradients on many points and forces PBE to optimize based on small ray segments, and high-density voxels may be incorrectly estimated from blurry observations [S2, unofficial].

### Progressive Blur Estimation
(No separate article, described here as it interacts with CRR).
*   **Definition:** A module that models blur from 2D coordinates to obtain kernel locations, using CRR to gather ray features [S2, unofficial].
*   **Role:** Uses CRR to gather ray features $f_{ray}(r_{x_i^*})$ for refined blur estimation [S2, unofficial].
*   **Loss:** Combines photometric loss for both coarse and refined blur estimates: $L_{CRR\_photo} = \sum_x (\|\tilde{\hat{C}}_{coar}^*(r_x) - \tilde{C}(r_x)\|^2 + \|\tilde{\hat{C}}_{coar}(r_x) - \tilde{C}(r_x)\|^2)$ [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Very fast. Color inference is performed once per ray instead of per voxel [S1].
*   **Memory Complexity:** Uses a decomposed feature tensor $G_{coar}$ representing 17 million voxels with channel dimensions $\{64, 16, 16\}$ [S1][S2, unofficial].
*   **Measured Performance:** In PDRF, the model size for a single scene is 160MB [S2, unofficial]. Trained with a batch size of 1024 rays, each with $N_c=64$ coarse samples and $N_f=64$ fine samples [S1][S2, unofficial].
*   **Strengths:** Fast computation; balances speed and performance by coarsely representing surface features [S1][S2, unofficial].
*   **Limitations:** Less accurate than per-voxel rendering; relies on multi-view observations to constrain features [S1].
*   **When to use:** As a coarse approximation in a two-stage rendering pipeline (e.g., PDRF) to provide initial density and features for fine rendering and blur estimation [S1][S2, unofficial].
*   **When not to use:** When high-fidelity per-voxel detail is required without subsequent refinement stages [S1].

## Background
The Coarse Ray Renderer was introduced in the PDRF (Progressively Deblurring Radiance Field) paper by Chen et al. in 2023 [S1][S2, unofficial]. It is part of a two-stage design that first estimates blur and then refines the radiance field, leveraging explicit voxel representations [S1][S2, unofficial].

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast Scene Reconstruction from Blurry Images, https://doi.org/10.1609/aaai.v37i2.25295, fetched 2026-10-26
[S2] PDRF: Progressively Deblurring Radiance Field for Fast and Robust Scene Reconstruction from Blurry Images, https://doi.org/10.48550/arxiv.2208.08049, fetched 2026-10-26 (unofficial)
