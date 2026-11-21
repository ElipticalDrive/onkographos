# Spatial-Ratio Neural Field Configuration

Also called: Gaussian Transform Field, Multi-Head Predictor, Spatial-Ratio Neural Field

## Overview
The Spatial-Ratio Neural Field is a module that predicts position and shape transformations for selected 3D Gaussians to patch missing areas caused by masking, mapping them from an "all-selected" state to an adapted state under a given elastic ratio $e$ [S1]. It serves as the core component of the Gaussian Transform Field, enabling flexible 3D Gaussian Splatting where the number of Gaussians can be adjusted dynamically without degrading rendering quality [S1]. It is distinguished from sibling methods like [Multi-MLP Linear Layer](15116-multi-mlp-linear-layer.md) by its use of a multi-plane implicit representation conditioned on a scalar elastic ratio, rather than static latent codes or simple linear projections [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D position $X \in \mathbb{R}^3$ and a scalar elastic ratio $e \in [0, 1]$.
*   **Output:** Displacement vectors for position ($T_X$), scaling ($T_s$), and rotation ($T_q$).

**Data Structures:**
The Spatial-Ratio Neural Field $\psi$ is a multi-resolution 4D volume $\psi = \{\psi_l \in \mathbb{R}^4\}_{l \in \{1,2\}}$ [S1]. Each vertex in the volume stores multi-channel features [S1]. To ensure efficiency, each resolution plane $\psi_l$ is factorized into 6 planes $\psi_l(k_1, k_2) \in \mathbb{R}^{d_{k1} \times d_{k2}}$ [S1].
*   **Plane Pairs:** $(k_1, k_2) \in \{(x, y), (x, z), (x, e), (y, z), (y, e), (z, e)\}$ [S1]. Note: The source text initially lists $(x,t), (y,t), (z,t)$ but clarifies in the implementation details that the dimensions are $(x, y, z, e)$, implying $t$ corresponds to the elastic ratio $e$ [S1].
*   **Resolutions:** The resolutions across the four dimensions $(x, y, z, e)$ are configured as $\{64, 64, 64, 100\}$ respectively [S1].
*   **Feature Dimension:** The hidden feature dimension $D$ for the Multi-Head Predictor is set to 64 [S1].

**Numbered Steps:**
1.  **Feature Querying:** For a given input $(X, e)$, query the multi-resolution volume $\psi$. For each resolution level $l$, perform bilinear interpolation ($BiInterp$) on the 6 factorized planes at the coordinates defined by the spatial components of $X$ and the ratio $e$ [S1].
2.  **Feature Fusion:** Aggregate the interpolated features $f_m$ from all resolution levels and pass them through a fusion MLP $F_m$ to obtain the final spatial-ratio feature $f$ [S1].
    $$ f = F_m(f_m), \quad f_m = BiInterp(\psi_l(X, e)) $$
    [S1]
3.  **Displacement Prediction:** Pass the feature $f$ through three separate Multi-Layer Perceptrons (MLPs), denoted as $\phi_X$, $\phi_s$, and $\phi_q$, to predict displacement vectors for position, scaling, and rotation respectively [S1].
    $$ T_X = \phi_X(f), \quad T_s = \phi_s(f), \quad T_q = \phi_q(f) $$
    [S1]
4.  **Transformation Application:** Apply the predicted displacements to the original Gaussian attributes to obtain the transformed state [S1].
    $$ X' = X + T_X, \quad s' = s + T_s, \quad q' = q + T_q $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bilinear_interpolate(plane, u, v):
    """
    Performs bilinear interpolation on a 2D plane at normalized coordinates (u, v).
    plane: 2D numpy array of shape (H, W)
    u, v: float in [0, 1]
    Returns: interpolated value
    """
    H, W = plane.shape
    # Map [0,1] to pixel coordinates
    u_px = u * (W - 1)
    v_px = v * (H - 1)
    
    u0, u1 = int(np.floor(u_px)), int(np.ceil(u_px))
    v0, v1 = int(np.floor(v_px)), int(np.ceil(v_px))
    
    # Clamp indices
    u0 = np.clip(u0, 0, W - 1)
    u1 = np.clip(u1, 0, W - 1)
    v0 = np.clip(v0, 0, H - 1)
    v1 = np.clip(v1, 0, H - 1)
    
    # Interpolate
    val_v0 = plane[v0, u0] * (u1 - u_px) * (v1 - v_px) + \
             plane[v0, u1] * (u_px - u0) * (v1 - v_px)
    val_v1 = plane[v1, u0] * (u1 - u_px) * (v_px - v0) + \
             plane[v1, u1] * (u_px - u0) * (v_px - v0)
             
    return val_v0 + val_v1

def query_spatial_ratio_neural_field(X, e, planes, resolutions, fusion_mlp):
    """
    Queries the Spatial-Ratio Neural Field.
    X: numpy array of shape (3,) representing (x, y, z)
    e: float representing elastic ratio
    planes: list of 6 lists, each containing 2 resolution planes (numpy arrays)
            Structure: [
                [(xy_l1, xy_l2), (xz_l1, xz_l2), (xe_l1, xe_l2), 
                 (yz_l1, yz_l2), (ye_l1, ye_l2), (ze_l1, ze_l2)]
            ]
            Note: planes[l][k] contains the two planes for resolution l and pair k.
    resolutions: list of tuples (H, W) for each plane pair.
                 Order: (x,y), (x,z), (x,e), (y,z), (y,e), (z,e)
    fusion_mlp: callable function taking concatenated features and returning f
    """
    x, y, z = X
    features = []
    
    # Pair indices mapping: 0:(x,y), 1:(x,z), 2:(x,e), 3:(y,z), 4:(y,e), 5:(z,e)
    # Coordinates for interpolation:
    # (x,y): u=x, v=y
    # (x,z): u=x, v=z
    # (x,e): u=x, v=e
    # (y,z): u=y, v=z
    # (y,e): u=y, v=e
    # (z,e): u=z, v=e
    
    coord_pairs = [
        (x, y), (x, z), (x, e), (y, z), (y, e), (z, e)
    ]
    
    for l in range(2): # Two resolutions
        l_features = []
        for k in range(6):
            u, v = coord_pairs[k]
            # Get the two planes for this resolution and pair
            plane_low, plane_high = planes[l][k]
            res_h, res_w = resolutions[k]
            
            # Interpolate both planes and average? Or concatenate?
            # Source says "BiInterp of psi_l at (X, e) and fusing across resolutions via MLP"
            # Usually, features from different resolutions are concatenated or summed.
            # Given "f_m = BiInterp(...)", it implies a single feature vector per resolution level.
            # We assume concatenation of the 6 interpolated values or a specific fusion.
            # Standard practice in Plenoxels/Instant-NGP is concatenation.
            # However, Eq 12 suggests f_m is the result of BiInterp. 
            # Let's assume we interpolate each plane and concatenate them for the resolution level.
            
            vals = []
            for plane in [plane_low, plane_high]: # Assuming planes list contains both resolutions? 
                # Re-reading layout: "psi = {psi_l}". "factorize psi_l into 6 planes".
                # So planes structure should be: planes[l][k] is the plane for res l, pair k.
                # But layout says "Resolutions: {64, 64, 64, 100}".
                # Let's assume planes is a list of 2 elements (resolutions), each is a list of 6 planes.
                pass 
            
            # Correct structure assumption based on "psi_l ... factorize into 6 planes":
            # planes[l][k] is a single plane array.
            val = bilinear_interpolate(planes[l][k], u, v)
            vals.append(val)
            
        # Concatenate features from 6 planes for this resolution
        f_l = np.concatenate(vals)
        features.append(f_l)
        
    # Fuse across resolutions
    f_m = np.concatenate(features)
    f = fusion_mlp(f_m)
    return f

def predict_transformations(X, e, planes, resolutions, fusion_mlp, phi_X, phi_s, phi_q):
    """
    Full pipeline to predict Gaussian transformations.
    """
    f = query_spatial_ratio_neural_field(X, e, planes, resolutions, fusion_mlp)
    
    T_X = phi_X(f)
    T_s = phi_s(f)
    T_q = phi_q(f)
    
    return T_X, T_s, T_q
```

**Parameters:**
*   **Hidden Feature Dimension ($D$):** 64 [S1].
*   **Plane Resolutions:** $\{64, 64, 64, 100\}$ for dimensions $\{x, y, z, e\}$ [S1].
*   **Elastic Ratio ($e$):** Sampled randomly from a set $\mathcal{E}$ during training [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Bilinear interpolation requires coordinates to be within $[0, 1]$. If $X$ or $e$ falls outside this range, clamping or normalization is required [G].
*   **Ratio Accuracy:** During inference, the number of activated entries may not exactly match the target ratio (e.g., target 0.20 results in ~19.5%) [S1]. Post-selection or thresholding adjustments may be needed to achieve the exact elastic ratio [S1].
*   **Opacity Handling:** During training, unselected Gaussians are rendered transparent by updating opacity $\alpha_i = \alpha_i \cdot \hat{M}_i$ [S1]. During inference, unselected Gaussians are directly discarded [S1].

## Subtopics

### Multi-MLP Linear Layer
Uses multiple linear layers to predict transformations. Simpler but less expressive than the neural field. Choose when computational resources are extremely limited. [Link](15116-multi-mlp-linear-layer.md)

### Latent Code Conditioning
Conditions transformations on a global latent code. Better for global style changes but less flexible for local spatial adjustments. Choose when global consistency is more important than local detail. [Link](16345-latent-code-conditioning.md)

## Operating characteristics
*   **Time Complexity:** Querying the field involves bilinear interpolation on 12 planes (6 pairs $\times$ 2 resolutions) per Gaussian, followed by three small MLP evaluations. This is $O(1)$ per Gaussian relative to scene size, but depends on the number of planes and hidden dimensions.
*   **Memory Complexity:** The field stores $2 \times 6$ planes. With resolutions $\{64, 64, 64, 100\}$, the memory footprint is small. Specifically, for dimensions $x,y,z$ at 64 and $e$ at 100, the plane sizes are $64 \times 64$, $64 \times 64$, $64 \times 100$, etc. Total parameters are manageable [S1].
*   **Strengths:** Enables dynamic adjustment of Gaussian count (elastic ratio) while maintaining rendering quality by intelligently redistributing Gaussian positions and shapes [S1].
*   **Limitations:** Requires training with multiple elastic ratios. Performance may degrade if the elastic ratio is extremely low, as the field must extrapolate transformations for sparse Gaussians [S1].
*   **When to Use:** When deploying 3D Gaussian Splatting models on devices with varying computational capabilities, allowing for a single model to serve multiple quality levels [S1].

## Background
The Spatial-Ratio Neural Field was introduced in the context of FlexGS, a method for flexible 3D Gaussian Splatting [S1]. It builds on previous work in dynamic scene representation, specifically referencing [45] and [12] for the multi-plane implicit representation technique [S1]. The method addresses the issue of "holes and missing details" that occur when simply masking out Gaussians for sparsity [S1].

## Sources
[S1] FlexGS: Train Once, Deploy Everywhere with Many-in-One Flexible 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.01523, fetched 2026-10-29
