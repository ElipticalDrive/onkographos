# Bilateral Edge-Aware Constraint

Also called: Lbilateral

## Overview
The Bilateral Edge-Aware Constraint is a geometric regularization term used in NeRF optimization to suppress local noise while preserving sharp geometric discontinuities such as building roofs and shadow transitions [S1]. Unlike standard smoothness constraints that blur distinct boundaries, this method dynamically weights geometric regularization based on photometric differences between neighboring rays, ensuring that significant color changes are not penalized as geometric errors [S1]. It addresses the lack of fixed neighborhood connectivity in implicit radiance fields by sampling neighbors via image-plane offsets [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** For a central ray $i$, the method requires the 3D position $P_i$, the unit surface normal $n_i$, and the albedo feature $c_i$. It also requires $K$ neighboring rays $j \in N_i$.
*   **Outputs:** A scalar loss value $L_{bilateral}$ added to the total training loss.

**Data Structures:**
*   **Neighbor Set $N_i$:** A set of $K$ indices corresponding to rays sampled at integer offsets in the image plane.
*   **3D Positions $P$:** Vector of shape $(3)$ for each sampled point.
*   **Albedo Features $c$:** Vector of shape $(C)$ (dimensionality depends on the network's feature output) for each sampled point.
*   **Normals $n$:** Vector of shape $(3)$ for the central point.

**Parameters:**
*   $K$: Number of neighbors sampled per ray. Value: 4 [S1].
*   $\sigma_d$: Adaptive spatial scale. Computed from the mean Euclidean distance to neighbors [S1].
*   $\sigma_w$: Adaptive photometric scale. Computed from the mean Euclidean distance in albedo space to neighbors [S1].
*   $\gamma_{bi}$: Weight for the bilateral term. Selected from $\{0.1, 0.01, 0.001\}$ based on validation performance [S1].

**Steps:**
1.  **Sample Neighbors:** For each ray $i$, draw $K=4$ integer image-plane offsets uniformly from $[-2, 2] \times [-2, 2]$, excluding the zero offset $(0,0)$ [S1].
2.  **Render Neighbors:** Render the corresponding neighbor rays to recover their 3D positions $P_j$, albedo features $c_j$, and normals $n_j$ (though only $n_i$ is used in the final loss formula) [S1].
3.  **Compute Adaptive Scales:**
    *   Compute $\sigma_d$ as the root mean square of the Euclidean distances between $P_i$ and $P_j$ for all $j \in N_i$ [S1]:
        $$ \sigma_d = \sqrt{\frac{1}{K} \sum_{j \in N_i} \| P_i - P_j \|^2} $$
    *   Compute $\sigma_w$ as the root mean square of the Euclidean distances between $c_i$ and $c_j$ for all $j \in N_i$ [S1]:
        $$ \sigma_w = \sqrt{\frac{1}{K} \sum_{j \in N_i} \| c_i - c_j \|^2} $$
4.  **Calculate Bilateral Weights:** For each neighbor $j$, compute the weight $w_{i,j}$ using the exponential terms for spatial and photometric distances [S1]:
    $$ w_{i,j} = \exp\left(-\frac{\| P_i - P_j \|^2}{2\sigma_d^2}\right) \cdot \exp\left(-\frac{\| c_i - c_j \|^2}{2\sigma_w^2}\right) $$
    Points close in both geometry and albedo receive strong weights; points with inconsistent photometric cues receive weak weights [S1].
5.  **Compute Normalization Factor:** Calculate $l_i$, the average edge length for normalization, defined as the mean distance to neighbors [S1]:
    $$ l_i = \frac{1}{K} \sum_{j \in N_i} \| P_i - P_j \| $$
6.  **Calculate Loss:** Minimize the projected distance of neighboring points onto the tangent plane of the central point. The loss for ray $i$ is the sum of weighted squared distances of neighbors from the tangent plane, normalized by $l_i$ [S1]:
    $$ L_{bilateral} = \frac{1}{N} \sum_{i=1}^{N} \frac{1}{l_i} \sum_{j \in N_i, j \neq i} w_{i,j} \left( (P_i - P_j) \cdot n_i \right)^2 $$
    where $N$ is the total number of rays processed in the batch [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilateral_edge_aware_loss(central_ray, neighbor_rays, K=4):
    """
    Computes the Bilateral Edge-Aware Constraint loss.
    
    Args:
        central_ray: Object/Dict containing:
            - P: 3D position (tensor, shape (3,))
            - c: Albedo feature (tensor, shape (C,))
            - n: Unit normal vector (tensor, shape (3,))
        neighbor_rays: List of K objects/Dicts, each containing:
            - P: 3D position (tensor, shape (3,))
            - c: Albedo feature (tensor, shape (C,))
            - (Normals n_j are not used in the final formula, but may be available)
            
    Returns:
        loss: Scalar tensor representing L_bilateral for the central ray.
    """
    import torch
    import torch.nn.functional as F
    
    P_i = central_ray['P']
    c_i = central_ray['c']
    n_i = central_ray['n']
    
    # Stack neighbor data
    P_j_list = [ray['P'] for ray in neighbor_rays]
    c_j_list = [ray['c'] for ray in neighbor_rays]
    
    # Convert to tensors if not already
    P_j = torch.stack(P_j_list) # Shape (K, 3)
    c_j = torch.stack(c_j_list) # Shape (K, C)
    
    # 1. Compute spatial distances and adaptive scale sigma_d
    # dist_sq_d: Shape (K,)
    dist_sq_d = torch.sum((P_i.unsqueeze(0) - P_j) ** 2, dim=1)
    sigma_d_sq = torch.mean(dist_sq_d) # Mean of squared distances
    
    # 2. Compute photometric distances and adaptive scale sigma_w
    # dist_sq_w: Shape (K,)
    dist_sq_w = torch.sum((c_i.unsqueeze(0) - c_j) ** 2, dim=1)
    sigma_w_sq = torch.mean(dist_sq_w) # Mean of squared distances
    
    # 3. Calculate bilateral weights w_i,j
    # Handle potential division by zero or very small sigma if all neighbors are identical
    # Using a small epsilon for stability
    eps = 1e-8
    
    # Spatial weight component
    w_spatial = torch.exp(-dist_sq_d / (2 * sigma_d_sq + eps))
    
    # Photometric weight component
    w_photometric = torch.exp(-dist_sq_w / (2 * sigma_w_sq + eps))
    
    # Combined bilateral weight
    w_ij = w_spatial * w_photometric # Shape (K,)
    
    # 4. Compute normalization factor l_i (average edge length)
    # dist_d: Shape (K,)
    dist_d = torch.sqrt(dist_sq_d + eps)
    l_i = torch.mean(dist_d)
    
    # 5. Compute projected distance onto tangent plane
    # Vector from P_i to P_j
    diff = P_i.unsqueeze(0) - P_j # Shape (K, 3)
    
    # Dot product with normal n_i
    # n_i is shape (3,), diff is (K, 3)
    projected_dist = torch.sum(diff * n_i.unsqueeze(0), dim=1) # Shape (K,)
    
    # Square the projected distance
    squared_proj_dist = projected_dist ** 2
    
    # 6. Weighted sum and normalization
    # Loss for this ray: sum(w_ij * squared_proj_dist) / l_i
    weighted_loss = torch.sum(w_ij * squared_proj_dist)
    loss = weighted_loss / (l_i + eps)
    
    return loss
```

**Edge Cases and Pitfalls:**
*   **Implicit Geometry Connectivity:** NeRF does not provide fixed surface connectivity. This method handles this by sampling neighbors via image-plane offsets and rendering them to find 3D positions, rather than relying on mesh topology [S1].
*   **Degenerate Neighbors:** If neighbors are extremely close or identical, $\sigma_d$ or $\sigma_w$ may approach zero, causing numerical instability in the exponential weights. A small epsilon should be added to the denominators [G].
*   **Photometric Ambiguity:** If photometric differences are due to lighting changes rather than geometry, the constraint might incorrectly preserve discontinuities. The method assumes photometric similarity implies geometric continuity [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of rays and $K$ is the number of neighbors (typically 4). Rendering $K$ additional rays per central ray increases computational cost significantly compared to single-ray evaluation.
*   **Memory Complexity:** $O(N \cdot K)$ to store neighbor positions and features during the backward pass.
*   **Strengths:** Effectively preserves sharp geometric boundaries (roofs, walls) while suppressing high-frequency noise on continuous surfaces [S1]. It is adaptive to local scene structure.
*   **Limitations:** Requires rendering multiple rays per sample, increasing training time. Performance depends on the quality of albedo features and normals.
*   **When to Use:** When reconstructing scenes with distinct geometric boundaries (e.g., urban environments, DSM reconstruction) where standard smoothness constraints cause over-smoothing [S1].
*   **When Not to Use:** For scenes with uniform geometry and no sharp edges, or when computational resources are severely limited.

## Background
The Bilateral Edge-Aware Constraint was proposed in the context of Digital Surface Model (DSM) reconstruction from multi-view satellite images using Neural Radiance Fields (NeRF) [S1]. It was introduced to address the limitation of standard smoothness constraints that blur distinct boundaries. The method is part of a broader geometric structure constraint framework that also includes a Normal Smoothness Constraint and physical shading losses [S1]. The authors selected $\gamma_{bi}$ from $\{0.1, 0.01, 0.001\}$ based on validation performance for each scene [S1].

## Sources
[S1] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
