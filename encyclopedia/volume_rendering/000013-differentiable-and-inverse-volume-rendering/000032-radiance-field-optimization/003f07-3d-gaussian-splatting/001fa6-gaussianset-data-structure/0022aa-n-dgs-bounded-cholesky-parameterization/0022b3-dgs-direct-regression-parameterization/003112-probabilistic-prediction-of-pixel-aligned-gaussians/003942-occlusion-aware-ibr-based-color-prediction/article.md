# Occlusion-Aware IBR-based Color Prediction

Also called: IBR Color Prediction, Occlusion-Aware Color Prediction

## Overview
This method predicts the color of a 3D Gaussian primitive by aggregating information from multiple nearby reference frames, using monocular depth priors to filter out occluded regions. It addresses the problem of retrieving inaccurate color information from 2D reference images due to occlusions, which degrades rendering quality in generalizable Gaussian Splatting. It sets itself apart from direct regression methods by leveraging Inter-View Consistency and explicit visibility estimation to distinguish visible from invisible features, allowing the model to focus on visible views without manual thresholding.

## How it works
**Inputs:**
*   3D Gaussian primitive $G_i$ with center $\mu_i$.
*   Set of $K$ nearby reference frames with camera centers $t_k$.
*   Monocular depth maps $D_r$ for the reference frames.
*   Reference frame images.

**Outputs:**
*   Spherical Harmonics (SH) coefficients representing the color $c_i$ of the Gaussian.

**Data Structures:**
*   Local color window $c_{ik} \in \mathbb{R}^{W \times W \times 3}$: A 3D tensor of size $W \times W \times 3$ containing RGB values sampled from the reference frame $k$.
*   Visibility map $v_{ik} \in \mathbb{R}^{W \times W}$: A 2D tensor of size $W \times W$ containing visibility weights.
*   Distance scalar $\delta_{ik} \in \mathbb{R}$.
*   Relative direction vector $\overrightarrow{\mathbf{\theta}}_{ik} \in \mathbb{R}^3$.

**Parameters:**
*   $K$: Number of nearby reference frames. Not given in the sources.
*   $W$: Window size for local color sampling. Effective choice is $W=3$ [S1].

**Steps:**
1.  **Projection:** For each reference frame $k \in \{1, \dots, K\}$, project the 3D Gaussian center $\mu_i$ to the 2D image plane of the reference frame.
2.  **Local Color Sampling:** Sample a local color window $c_{ik}$ of size $W \times W \times 3$ from the reference frame image $k$, centered at the 2D projection of $\mu_i$. This integrates 2D texture information and enhances robustness against noisy 3D Gaussian locations [S1].
3.  **Depth Retrieval:** Bilinearly retrieve the monocular depth value $d_{ik}$ from the reference frame's depth map $D_r$ at the 2D projection location. This serves as pseudo ground truth [S1].
4.  **Distance Calculation:** Compute the Euclidean distance $\delta_{ik}$ between the Gaussian center $\mu_i$ and the reference camera center $t_k$:
    $$ \delta_{ik} = \|\mu_i - t_k\|_2 $$
    [S1]
5.  **Visibility Estimation:** Estimate the visibility map $v_{ik}$ for the window. The visibility is derived from the difference between the projected depth (distance $\delta_{ik}$) and the retrieved monocular depth $d_{ik}$. The formula is:
    $$ v_{ik} = \frac{\delta_{ik} - d_{ik}}{\delta_{ik}} $$
    [S1]
    Note: $d_{ik}$ in the formula refers to the depth window corresponding to $c_{ik}$. This step eliminates invisibility projections based on depth priors [S1].
6.  **Relative Direction Calculation:** Compute the normalized relative direction vector $\overrightarrow{\mathbf{\theta}}_{ik}$ from the camera to the Gaussian:
    $$ \overrightarrow{\mathbf{\theta}}_{ik} = \frac{\mu_i - t_k}{\|\mu_i - t_k\|_2} $$
    [S1]
7.  **Aggregation:** Pass the collected features from all $K$ frames through an aggregation module $\mathcal{M}$ to derive the final Gaussian color $c_i$. The module $\mathcal{M}$ is implemented as a three-layer Multi-Layer Perceptron (MLP) with a width of 64 [S1]. The input to $\mathcal{M}$ is the set of tuples $\{c_{ik}, v_{ik}, \delta_{ik}, \overrightarrow{\mathbf{\theta}}_{ik}\}_{k=1}^{K}$ [S1].
8.  **SH Representation:** The output of $\mathcal{M}$ is represented as Spherical Harmonics (SH) coefficients to model view-dependent color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def occlusion_aware_color_prediction(mu_i, K_ref_frames, D_r, W=3):
    """
    Predicts Gaussian color using occlusion-aware IBR.
    
    Args:
        mu_i: 3D center of the Gaussian (3,)
        K_ref_frames: List of K reference frame data structures.
                      Each frame has:
                      - 'image': H x W x 3 tensor
                      - 'depth': H x W tensor (monocular depth map D_r)
                      - 'camera_center': 3D position t_k (3,)
        D_r: Monocular depth maps for reference frames.
        W: Window size for sampling (default 3).
        
    Returns:
        sh_coeffs: Spherical Harmonics coefficients for the Gaussian color.
    """
    features = []
    
    for k in range(len(K_ref_frames)):
        frame = K_ref_frames[k]
        t_k = frame['camera_center']
        img_k = frame['image']
        depth_k = frame['depth']
        
        # 1. Project mu_i to 2D (Assuming standard projection function exists)
        # Note: The source assumes projection is possible. 
        # In a full implementation, this requires camera intrinsics/extrinsics.
        u_k, v_k = project_2d(mu_i, frame['intrinsics'], frame['extrinsics'])
        
        # 2. Sample local color window c_ik
        # Sample W x W x 3 centered at (u_k, v_k)
        c_ik = sample_window(img_k, u_k, v_k, W) # Shape: W x W x 3
        
        # 3. Retrieve monocular depth d_ik
        # Bilinearly retrieve depth at (u_k, v_k)
        d_ik = bilinear_retrieve(depth_k, u_k, v_k) # Shape: W x W (or scalar if single point)
        # Source says "depth window corresponding to c_ik", implying d_ik might be a map
        # If d_ik is a scalar, broadcasting applies. If map, element-wise op.
        # Assuming d_ik is retrieved for the window center or as a map.
        # Source formula: v_ik = (delta_ik - d_ik) / delta_ik
        # If d_ik is a window, v_ik is a window.
        
        # 4. Calculate distance delta_ik
        delta_ik = np.linalg.norm(mu_i - t_k)
        
        # 5. Estimate visibility map v_ik
        # v_ik = (delta_ik - d_ik) / delta_ik
        # If d_ik is a scalar, v_ik is a scalar. If d_ik is a map, v_ik is a map.
        # Source notation v_ik \in R^{WxW} implies d_ik is likely a map or sampled per pixel.
        # Let's assume d_ik is sampled for the window to produce a visibility map.
        d_ik_map = bilinear_retrieve_window(depth_k, u_k, v_k, W) # Shape: W x W
        v_ik = (delta_ik - d_ik_map) / delta_ik
        
        # 6. Calculate relative direction theta_ik
        theta_ik = (mu_i - t_k) / np.linalg.norm(mu_i - t_k)
        
        features.append({
            'c_ik': c_ik,
            'v_ik': v_ik,
            'delta_ik': delta_ik,
            'theta_ik': theta_ik
        })
        
    # 7. Aggregation via MLP M
    # M is a 3-layer MLP of width 64.
    # Input: concatenation of features from all K frames.
    # The exact concatenation strategy (channel-wise, spatial flattening) is not detailed.
    # We assume a standard aggregation where features are processed.
    
    # Placeholder for MLP implementation details not fully specified in source
    # The source states: c_i = M({c_ik, v_ik, delta_ik, theta_ik}_{k=1}^K)
    # We assume the MLP takes these as inputs and outputs SH coefficients.
    
    sh_coeffs = aggregation_mlp(features)
    
    return sh_coeffs

def aggregation_mlp(features):
    """
    Aggregation module M: 3-layer MLP, width 64.
    """
    # Flatten and concatenate features from all K frames
    # This is a simplification as the exact tensor shape handling is not in source.
    # In practice, this would involve reshaping c_ik (WxWx3) and v_ik (WxW)
    # and combining with scalars delta_ik and vector theta_ik.
    
    # Example structure (not executable without full context):
    # inputs = []
    # for f in features:
    #     inputs.append(f['c_ik'].flatten())
    #     inputs.append(f['v_ik'].flatten())
    #     inputs.append(f['delta_ik'])
    #     inputs.append(f['theta_ik'])
    # x = torch.cat(inputs, dim=-1)
    # x = torch.relu(Linear(64, 64)(x))
    # x = torch.relu(Linear(64, 64)(x))
    # sh_coeffs = Linear(64, SH_DIM)(x)
    
    # Since the source does not give the exact MLP architecture beyond "3-layer, width 64",
    # we cannot provide a complete executable implementation.
    # [G] The aggregation module is a 3-layer MLP with hidden width 64.
    # The input is the combination of local colors, visibility maps, distances, and directions.
    # The output is SH coefficients.
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** Bilinear retrieval of depth and color requires careful handling of boundary conditions if the projection falls outside the image frame.
*   **Occlusion Ambiguity:** The visibility formula $v_{ik} = (\delta_{ik} - d_{ik})/\delta_{ik}$ assumes that if the monocular depth $d_{ik}$ is significantly smaller than the projected depth $\delta_{ik}$, the point is occluded. If $d_{ik} > \delta_{ik}$, the visibility might become negative, which should be handled (e.g., clamped to 0) depending on implementation, though the source does not explicitly specify clamping.
*   **Noisy Depth:** The method relies on monocular depth as pseudo ground truth. Inaccurate depth priors may lead to incorrect visibility estimation.

## Operating characteristics
*   **Time Complexity:** Depends on $K$ (number of reference frames) and $W$ (window size). For each Gaussian, $K$ projections and window samplings are performed, followed by an MLP inference.
*   **Memory Complexity:** Stores local color windows and visibility maps for $K$ frames per Gaussian during the forward pass.
*   **Strengths:** Improves robustness against noisy 3D Gaussian locations by integrating 2D texture information. Effectively distinguishes occlusion-caused features using depth priors, enhancing rendering quality.
*   **Limitations:** Relies on the quality of monocular depth estimates. Requires access to reference frames and their depth maps.
*   **When to use:** When generalizable Gaussian Splatting suffers from occlusion artifacts and inaccurate color retrieval from 2D references.
*   **When not to use:** If monocular depth priors are unavailable or highly unreliable.

## Background
This method is part of the EVolSplat framework, introduced in "EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis" [S1]. It addresses issues in generalizable Gaussian Splatting where occlusions lead to inaccurate color information. The method extends projection locations into local sample windows and leverages depth priors to eliminate invisibility projections.

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
