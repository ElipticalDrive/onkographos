# Geometry Aware Feature Extraction

Also called: Geometrically Enhanced Feature Aggregation

## Overview
Geometry Aware Feature Extraction is a module that enhances source view features by aggregating information from other source views using geometric constraints (ray projection) and cross-view attention [S1]. It resolves the insufficiency of single-point feature extraction for semantic prediction by approximating dense semantic connections in 3D space [S1]. It sets itself apart from siblings like [3D Alignment via Reprojection](11791-3d-alignment-via-reprojection.md) by explicitly modeling the 3D ray structure to guide feature aggregation, rather than relying solely on 2D reprojection alignment [S1].

## How it works
**Inputs:**
- Set of $N$ source views $\{I_i, K_i, E_i\}_{i=1}^N$, where $I_i$ is the image, $K_i$ is the camera intrinsic matrix, and $E_i = [R_i, t_i]$ contains the rotation matrix $R_i$ and translation vector $t_i$ for the $i$-th view [S1].
- Number of sample points per ray, $N_s$ [S1].

**Outputs:**
- Geometrically enhanced features $\{F'_i\}_{i=1}^N$ [S1].

**Data Structures:**
- $F_i$: Feature map extracted from $I_i$ by a shared CNN. Dimensions depend on the CNN architecture [S1].
- $f_i$: Local feature vector at 2D coordinate $u_i$ in $F_i$ [S1].
- $g_{js}$: Local feature vector at projected coordinate $v_{js}$ in $F_j$ [S1].
- $P(s)$: Position embedding for the $s$-th sample point along the ray [S1].

**Steps:**

1.  **Feature Extraction:** For each source view $i$, use a shared Convolutional Neural Network (CNN) to extract feature map $F_i$ from image $I_i$ [S1].

2.  **Ray Definition:** For each 2D coordinate $u_i = [u, v]^\top$ on the $i$-th source view, compute the corresponding ray $R$ in world coordinates parametrized by $\delta$:
    $$ R(\delta) = t_i + \delta R_i K_i^{-1} \begin{bmatrix} u_i \\ 1 \end{bmatrix} $$
    where $K_i$ is the intrinsic matrix and $[R_i, t_i]$ are the extrinsic parameters of the $i$-th source view [S1].

3.  **Ray Sampling and Projection:** Sample a sequence of points $p_s = R(\delta_s)$ along the ray $R$ for $s = 1, \dots, N_s$. Project each point $p_s$ to the $j$-th source view to obtain 2D coordinates $v_{js}$ and depth $d_{sj}$:
    $$ d_{sj} \begin{bmatrix} v_{js}^\top \\ 1 \end{bmatrix} = K_j R_j^{-1} (p_s - t_j) $$
    where $v_{js}$ is the 2D coordinate in the $j$-th source view and $d_{sj}$ is the corresponding depth [S1]. This yields a collection of projections $\{v_{js}\}_{s=1}^{N_s}$ for the $j$-th source view [S1].

4.  **Stage 1: Intra-View Aggregation:** For a target view $i$ and a neighbor view $j$, let $f_i$ be the local feature of $F_i$ at $u_i$, and $g_{js}$ be the local feature of $F_j$ at $v_{js}$. Aggregate features from the projection points within the $j$-th source view using Multi-Head Attention (MHA):
    $$ g_j = \text{MHA}(f_i, \{g_{js} + P(s)\}_{s=1}^{N_s}) $$
    Here, $f_i$ is the query, $\{g_{js} + P(s)\}_{s=1}^{N_s}$ serve as the keys and values, and $P(s)$ is the position embedding of the sample index $s$ [S1].

5.  **Stage 2: Cross-View Aggregation:** Aggregate features from other source views. Compute the enhanced feature $f'_i$ using MHA:
    $$ f'_i = \text{MHA}(f_i, \{g_j\}_{j=1, j \neq i}^{N}) $$
    Here, $f_i$ is the query and $\{g_j\}_{j=1, j \neq i}^{N}$ are the keys and values [S1].

6.  **Iteration:** Apply the above process for all source views to obtain the final geometrically enhanced features $\{F'_i\}_{i=1}^N$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_aware_feature_extraction(source_views, Ns):
    """
    source_views: List of dicts containing 'image', 'K', 'R', 't' for each view i.
    Ns: Number of sample points along the ray.
    Returns: List of enhanced feature maps F_prime.
    """
    N = len(source_views)
    F = []
    
    # Step 1: Extract features using shared CNN
    for view in source_views:
        F.append(shared_cnn(view['image']))
        
    F_prime = [None] * N
    
    for i in range(N):
        # Initialize enhanced feature map for view i
        F_prime_i = torch.zeros_like(F[i])
        
        # Iterate over all 2D coordinates u_i in the feature map F[i]
        # In practice, this is done via vectorized operations or grid sampling
        for u_i in all_pixel_coordinates(F[i]):
            f_i = get_feature_at(F[i], u_i)
            
            # Step 2: Define Ray R for u_i
            # R(delta) = t_i + delta * R_i * K_i^-1 * [u_i, 1]^T
            # We need to sample delta values. The source does not specify the range.
            # [G] Assuming a standard depth range [delta_min, delta_max] is provided or inferred.
            deltas = sample_depths(Ns) 
            
            rays_points = []
            for delta in deltas:
                # Compute world point p_s
                homogenous_u = torch.tensor([u_i[0], u_i[1], 1.0])
                inv_K_i = torch.linalg.inv(source_views[i]['K'])
                dir_vec = source_views[i]['R'] @ inv_K_i @ homogenous_u
                p_s = source_views[i]['t'] + delta * dir_vec
                rays_points.append(p_s)
            
            view_g_j_list = []
            
            # Step 3 & 4: For each other view j, project and aggregate
            for j in range(N):
                if i == j:
                    continue
                    
                # Step 3: Project points to view j
                g_js = []
                for s, p_s in enumerate(rays_points):
                    # d_sj * [v_js^T, 1]^T = K_j * R_j^-1 * (p_s - t_j)
                    inv_K_j = torch.linalg.inv(source_views[j]['K'])
                    inv_R_j = torch.linalg.inv(source_views[j]['R'])
                    
                    # Compute right hand side
                    rhs = source_views[j]['K'] @ inv_R_j @ (p_s - source_views[j]['t'])
                    
                    # Extract depth and 2D coords
                    d_sj = rhs[2]
                    v_js = rhs[:2] / d_sj
                    
                    # Get feature at v_js from F[j]
                    # [G] Requires bilinear interpolation or grid_sample to handle non-integer coordinates
                    g_js.append(get_feature_at(F[j], v_js))
                
                # Step 4: Stage 1 Aggregation
                # g_j = MHA(f_i, {g_js + P(s)})
                # Prepare keys/values with position embeddings
                keys_values = [g_js[s] + position_embedding(s) for s in range(Ns)]
                g_j = multi_head_attention(query=f_i, keys=keys_values, values=keys_values)
                
                view_g_j_list.append(g_j)
            
            # Step 5: Stage 2 Aggregation
            # f'_i = MHA(f_i, {g_j}_{j!=i})
            # keys_values_stage2 = view_g_j_list
            f_prime_i = multi_head_attention(query=f_i, keys=view_g_j_list, values=view_g_j_list)
            
            F_prime_i[get_pixel_index(u_i)] = f_prime_i
            
        F_prime[i] = F_prime_i
        
    return F_prime
```

**Parameters:**
- $N_s$: Number of sample points along the ray. Not given in the sources.
- Position Embedding $P(s)$: Not given in the sources.

**Edge Cases and Pitfalls:**
- Ray projection may fall outside the image boundaries of source views $j$. The source does not specify handling for out-of-bounds projections [S1]. [G] Standard practice is to clamp or ignore such samples.
- Depth ambiguity: The parametrization $\delta$ requires a range. The source does not specify how $\delta_s$ are chosen (e.g., uniform, learned) [S1]. [G] Implementers must define a depth range $[\delta_{min}, \delta_{max}]$.

## Subtopics

## Operating characteristics
- **Time Complexity:** Not given in the sources. The method involves $N$ views, $N_s$ samples per ray, and Multi-Head Attention operations, suggesting complexity related to $O(N^2 \cdot N_s)$.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Approximates dense semantic connections in 3D space by reprojecting query rays to source views and aggregating features [S1]. Reduces computational cost and GPU memory usage compared to dense multi-view attention while maintaining comparable semantic segmentation performance [S1].
- **Limitations:** Relies on accurate camera parameters and geometry. May fail if the geometry prediction is inaccurate, leading to artifacts [S1].
- **When to use:** Synthetic-to-real generalization of NeRF where semantic consistency across views is critical [S1].
- **When not to use:** When computational resources are extremely limited and $N_s$ or $N$ are large, as the attention mechanism adds overhead.

## Background
Originated in the paper "ContraNeRF: Generalizable Neural Radiance Fields for Synthetic-to-real Novel View Synthesis via Contrastive Learning" [S1]. The method was introduced to address the problem where existing NeRF models may fail to accurately predict scene geometry, resulting in serious artifacts in rendered images, particularly in synthetic-to-real generalization scenarios [S1]. It is used in the context of training NeRF models on synthetic data to generalize to real data [S1].

## Sources
[S1] ContraNeRF: Generalizable Neural Radiance Fields for Synthetic-to-real Novel View Synthesis via Contrastive Learning, https://doi.org/10.1109/cvpr52729.2023.01584, fetched 2026-10-29
