# Geometry Aware Feature Extraction

Also called: Geometrically Enhanced Feature Aggregation

## Overview
Geometry Aware Feature Extraction is a module that enhances source view features by aggregating information from other source views using geometric constraints (ray projection) and cross-view attention [S1]. It resolves ambiguous feature assignment by leveraging the 3D geometry of the scene to guide feature aggregation, ensuring that features from different views correspond to the same physical location [S1]. It builds on [Pose-Aware Projection (PAP)](8527-pose-aware-projection-pap.md) by providing the geometrically enhanced features required for robust generalization in synthetic-to-real scenarios [S1].

## How it works
**Inputs:**
- A set of $N$ source views, where each view $i$ consists of an image $I_i$, camera intrinsic matrix $K_i$, and extrinsic parameters $[R_i, t_i]$ [S1].
- A shared Convolutional Neural Network (CNN) [S1].
- Number of sample points along the ray, $N_s$ [S1].

**Outputs:**
- A set of geometrically enhanced features $\{F'_i\}_{i=1}^{N}$, where $F'_i$ has the same spatial dimensions as the input feature map $F_i$ [S1].

**Data Structures:**
- $F_i$: Feature map extracted from image $I_i$ using the shared CNN. Dimensions: $H \times W \times C$ [S1].
- $f_i$: Local feature vector at 2D coordinate $u_i$ in $F_i$. Dimensions: $C$ [S1].
- $v^j_s$: 2D projection coordinates in view $j$ for the $s$-th sample point. Dimensions: $2$ [S1].
- $g^j_s$: Local feature vector at $v^j_s$ in $F_j$. Dimensions: $C$ [S1].
- $P(s)$: Position embedding for the $s$-th sample point. Dimensions: $C$ [S1].
- $R(\delta)$: Ray in world coordinates parametrized by depth $\delta$ [S1].

**Steps:**

1.  **Feature Extraction:** For each source view $i \in \{1, \dots, N\}$, extract features $F_i$ from image $I_i$ using a shared CNN [S1].

2.  **Ray Construction:** For each 2D pixel coordinate $u_i = [u, v]^\top$ on the $i$-th source view, compute the corresponding ray $R$ in world coordinates parametrized by $\delta$ [S1]:
    $$
    R(\delta) = t_i + \delta R_i K_i^{-1} \begin{bmatrix} u_i \\ 1 \end{bmatrix}
    $$
    where $K_i$ is the camera intrinsic matrix and $[R_i, t_i]$ are the extrinsic parameters (rotation and translation) of the $i$-th source view [S1].

3.  **Ray Sampling and Projection:** Sample a sequence of $N_s$ points $p_s = R(\delta_s)$ along the ray $R$. For each sample point $p_s$, project it onto the $j$-th source view ($j \neq i$) to obtain 2D coordinates $v^j_s$ and depth $d^j_s$ [S1]:
    $$
    d^j_s \begin{bmatrix} (v^j_s)^\top \\ 1 \end{bmatrix} = K_j R_j^{-1} (p_s - t_j)
    $$
    where $v^j_s$ is the 2D coordinate in view $j$ and $d^j_s$ is the corresponding depth [S1]. This yields a collection of projections $\{v^j_s\}_{s=1}^{N_s}$ for the $j$-th source view [S1].

4.  **Stage 1: Intra-View Aggregation:** For each target view $j$ (where $j \neq i$), aggregate features from the sampled points. Let $f_i$ be the local feature of $F_i$ at $u_i$, and $g^j_s$ be the local feature of $F_j$ at $v^j_s$. Compute the aggregated feature $g_j$ using Multi-Head Attention (MHA) [S1]:
    $$
    g_j = \text{MHA}(f_i, \{g^j_s + P(s)\}_{s=1}^{N_s})
    $$
    Here, $f_i$ serves as the query, and $\{g^j_s + P(s)\}_{s=1}^{N_s}$ serve as the keys and values. $P(s)$ is a learnable position embedding for the sample index $s$ [S1].

5.  **Stage 2: Cross-View Aggregation:** Aggregate the features from all other source views. Compute the enhanced feature $f'_i$ using MHA [S1]:
    $$
    f'_i = \text{MHA}(f_i, \{g_j\}_{j=1, j \neq i}^{N})
    $$
    Here, $f_i$ is the query and $\{g_j\}_{j=1, j \neq i}^{N}$ are the keys and values [S1].

6.  **Reconstruction:** Apply the above process for all pixels in all source views to obtain the final geometrically enhanced features $\{F'_i\}_{i=1}^{N}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_aware_feature_extraction(source_views, shared_cnn, n_samples):
    """
    source_views: List of dicts, each containing 'image', 'K', 'R', 't'
    shared_cnn: Function to extract features from image
    n_samples: Number of sample points along the ray (Ns)
    """
    n_views = len(source_views)
    enhanced_features = []

    for i in range(n_views):
        view_i = source_views[i]
        # Step 1: Extract features for view i
        F_i = shared_cnn(view_i['image'])
        H, W, C = F_i.shape
        
        # Initialize output feature map
        F_prime_i = np.zeros_like(F_i)
        
        # Iterate over all pixels in view i
        for u in range(H):
            for v in range(W):
                u_i = np.array([u, v])
                f_i = F_i[u, v, :] # Local feature at ui
                
                # Step 2: Construct ray R(delta) = ti + delta * Ri * Ki^-1 * [ui, 1]^T
                # Note: In practice, delta is sampled from a range [delta_min, delta_max]
                # The specific delta values are not given in sources, assumed to be fixed or learned
                # [G] Assuming uniform sampling over a predefined depth range [d_min, d_max]
                deltas = np.linspace(d_min, d_max, n_samples)
                
                # Precompute ray direction
                # [G] Assuming homogeneous coordinate handling
                pixel_hom = np.array([u, v, 1])
                ray_dir = view_i['R'] @ np.linalg.inv(view_i['K']) @ pixel_hom
                ray_origin = view_i['t']
                
                # Step 3: Sample points and project to other views
                # Store aggregated features from other views
                g_j_list = []
                
                for j in range(n_views):
                    if i == j:
                        continue
                    
                    view_j = source_views[j]
                    g_j_accumulator = []
                    
                    for s in range(n_samples):
                        delta_s = deltas[s]
                        # Sample point ps = ti + delta_s * ray_dir
                        p_s = ray_origin + delta_s * ray_dir
                        
                        # Project ps to view j: dsj * [vjs^T, 1]^T = Kj * Rj^-1 * (ps - tj)
                        # [G] Assuming standard perspective projection
                        # Compute homogeneous coordinates in view j
                        p_j_hom = view_j['R'] @ (p_s - view_j['t'])
                        depth_sj = p_j_hom[2]
                        
                        # Avoid division by zero or invalid depths
                        if depth_sj <= 0:
                            continue
                            
                        v_js_hom = view_j['K'] @ p_j_hom
                        v_js = v_js_hom[:2] / v_js_hom[2]
                        
                        # Check if projection is within image bounds
                        if 0 <= v_js[0] < W and 0 <= v_js[1] < H:
                            # Extract feature gjs from Fj at vjs
                            # [G] Assuming bilinear interpolation for non-integer coordinates
                            g_js = bilinear_interpolate(view_j['features'], v_js[0], v_js[1])
                            
                            # Add position embedding P(s)
                            # [G] P(s) is a learnable embedding for index s
                            p_s_emb = position_embedding(s, C)
                            g_js_with_pos = g_js + p_s_emb
                            g_j_accumulator.append(g_js_with_pos)
                    
                    # If no valid projections, handle gracefully (e.g., skip or use zero)
                    if not g_j_accumulator:
                        continue
                        
                    # Step 4: Stage 1 - Aggregate features from projection points in view j
                    # gj = MHA(fi, {gjs + P(s)})
                    # [G] MHA expects query, key, value tensors. Here keys and values are the same list.
                    keys_values = np.array(g_j_accumulator) # Shape: (Ns, C)
                    g_j = multi_head_attention(query=f_i, keys=keys_values, values=keys_values)
                    g_j_list.append(g_j)
                
                # Step 5: Stage 2 - Aggregate features from other source views
                # fi' = MHA(fi, {gj})
                if g_j_list:
                    keys_values_cross = np.array(g_j_list) # Shape: (N-1, C)
                    f_prime_i = multi_head_attention(query=f_i, keys=keys_values_cross, values=keys_values_cross)
                    F_prime_i[u, v, :] = f_prime_i
                else:
                    F_prime_i[u, v, :] = f_i # Fallback to original feature

        enhanced_features.append(F_prime_i)
        
    return enhanced_features
```

**Parameters:**
- $N_s$: Number of sample points along the ray. Typical value: Not given in the sources.
- $d_{min}, d_{min}$: Depth range for sampling. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Invalid Projections:** If a sampled point $p_s$ projects outside the image bounds of view $j$ or has non-positive depth, it should be ignored [S1].
- **Occlusion:** The method assumes that features from other views correspond to the same physical point. Occlusions may lead to incorrect feature aggregation if not handled [S1].
- **Precision:** Floating-point precision in ray projection and interpolation may affect accuracy [G].

## Operating characteristics
- **Time Complexity:** $O(N^2 \cdot H \cdot W \cdot N_s \cdot C^2)$, where $N$ is the number of views, $H \times W$ is the image resolution, $N_s$ is the number of samples, and $C$ is the feature dimension. The MHA operation dominates the complexity [S1].
- **Memory Complexity:** $O(N \cdot H \cdot W \cdot C)$ to store the feature maps [S1].
- **Strengths:** Improves generalization in synthetic-to-real scenarios by leveraging geometric constraints [S1].
- **Limitations:** Computationally expensive due to ray sampling and multi-view attention [S1].

## Background
- **Origin:** Introduced in ContraNeRF (2023) for synthetic-to-real generalization of Neural Radiance Fields [S1].
- **History:** Developed to address artifacts caused by inaccurate geometry prediction in NeRF models [S1].

## Sources
[S1] ContraNeRF: Generalizable Neural Radiance Fields for Synthetic-to-real Novel View Synthesis via Contrastive Learning, https://doi.org/10.1109/cvpr52729.2023.01584, fetched 2026-10-29
