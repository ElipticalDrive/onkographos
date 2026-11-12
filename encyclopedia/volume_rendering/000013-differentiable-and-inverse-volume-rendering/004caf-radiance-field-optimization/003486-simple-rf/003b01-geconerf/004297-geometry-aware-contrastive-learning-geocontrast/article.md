# Geometry Aware Contrastive Learning (GeoContrast)

Also called: GeoContrast

## Overview
Geometry Aware Contrastive Learning (GeoContrast) is a regularization technique for few-shot NeRF that enforces multi-view geometric consistency by distinguishing surface points from non-surface points using depth-guided warping. It sets itself apart from sibling methods like random pixel sampling by explicitly constructing positive and negative pairs based on 3D geometric projections, thereby preventing the model from learning incorrect depth-geometry mappings in under-constrained scenarios.

## How it works
GeoContrast operates on pairs of source views to extract geometrically enhanced features. It constructs contrastive pairs by projecting 2D pixels into 3D space and re-projecting them into other views, using ground truth depth to identify surface correspondences.

### Inputs and Outputs
*   **Inputs:**
    *   A set of $N_{near}$ source views $\{I_i, K_i, E_i\}_{i=1}^{N_{near}}$, where $I_i$ is the image, $K_i$ is the intrinsic matrix, and $E_i$ is the extrinsic matrix.
    *   Ground truth depth maps $\delta$ for the source views.
    *   A shared CNN feature extractor $M(\cdot)$ (from GeCoNeRF) that produces feature maps $F_i$ from images $I_i$.
    *   Hyperparameters: $N_c$ (number of sampled pixels per view), $N_{neg}$ (number of negative samples per pixel), $\tau'$ (temperature, default 10000).
*   **Outputs:**
    *   A scalar contrastive loss value $L_{contrast}$ used to update the feature extractor weights.

### Data Structures
*   **Pixel Coordinate:** $p = (u, v)^\top$ representing a 2D coordinate in a source view.
*   **Homogeneous Coordinate:** $\tilde{p}$ representing $p$ in homogeneous coordinates.
*   **3D Point:** $p_3$ representing a point in world coordinates.
*   **Feature Vector:** $f_p$ representing the feature vector extracted at pixel $p$.

### Numbered Steps

1.  **Feature Extraction:** For each source view $i$, extract feature maps $F_i$ using the shared CNN $M(\cdot)$.
2.  **Pixel Sampling:** For a pair of source views $i$ and $j$, randomly sample a batch of $N_c$ pixels $\{p\}$ from view $i$. Each pixel is denoted $p = (u, v)$.
3.  **Positive Pair Construction:**
    *   For each sampled pixel $p$ in view $i$, retrieve its depth $\delta_p$ from the ground truth depth map.
    *   Compute the 3D point $p_3$ corresponding to $p$ using the camera parameters:
        $$ p_3 = t_i + \delta_p R_i K_i^{-1} \tilde{p} $$
        where $t_i$ and $R_i$ are the translation and rotation components of the extrinsic matrix $E_i$ for view $i$.
    *   Project $p_3$ into view $j$ to obtain the 2D coordinate $q^+$:
        $$ q^+ = K_j (R_j^{-1} (p_3 - t_j)) $$
        (Note: The exact projection formula depends on the coordinate system convention, but generally involves inverse extrinsics and intrinsics).
    *   **Occlusion Check:** Verify that the region around $q^+$ in view $j$ is unoccluded. If occluded, discard $(p, q^+)$ as a positive pair. If unoccluded, $(p, q^+)$ is a positive pair.
4.  **Negative Pair Construction:**
    *   For the same pixel $p$ in view $i$, generate $N_{neg}$ negative samples $\{q^-\}_{k=1}^{N_{neg}}$ in view $j$.
    *   These are obtained by projecting $p$ to view $j$ using *different* depth values or by sampling pixels that do not correspond to the surface point $p_3$. The source text states: "We first project $p$ to the j-th source view to get a collection of projections $\{q^-\}$... by taking Eq.(3) and Eq.(4)." This implies using the same 2D coordinate $p$ but potentially different depth assumptions or simply treating non-corresponding projections as negatives. Specifically, the text notes that negative pairs are "projections of non-surface 3D points" or pixels sampled randomly in previous methods, but GeoContrast uses geometric constraints. The most precise interpretation from the text is that $q^-$ are projections derived from the same 2D ray but associated with non-surface depths or simply other pixels in view $j$ that are not the true correspondence. However, the text explicitly says "project p to the j-th source view to get a collection of projections...". This likely means using the same 2D coordinate $p$ but interpreting it via different depth hypotheses or simply treating the set of all other pixels in view $j$ as negatives, weighted by geometry. Given the formula involves distance to $q^+$, $q^-$ are likely other pixels in view $j$.
    *   *Clarification from text:* "Here, each negative pair is the projection of 3D point under different source views...". This phrasing is slightly ambiguous. However, the loss function compares $q^+$ and $q^-$. The standard contrastive setup implies $q^-$ are pixels in view $j$ that are *not* the correct projection of $p_3$.
5.  **Weight Calculation:** Calculate the weight $\lambda_{q^-}$ for each negative sample $q^-$ based on its geometric distance to the positive sample $q^+$:
    $$ \lambda_{q^-} = \frac{\exp(\|q^+ - q^-\|^2 / \tau')}{\sum_{r \in R} \exp(\|q^+ - r\|^2 / \tau')} $$
    where $R$ is the set of negative samples (or potentially all pixels in the batch, but the text says "set of rays in each batch" which is likely a typo for "set of negative samples" or "set of projected points"). The denominator sums over the negative samples $\{q^-\}$. $\tau' = 10000$.
6.  **Loss Computation:** Compute the contrastive loss for pixel $p$:
    $$ L_{contrast}^p = -\log \frac{\exp(\|q^+ - q^+\|^2 / \tau')}{\exp(\|q^+ - q^+\|^2 / \tau') + \sum_{k=1}^{N_{neg}} \lambda_{q_k^-} \exp(\|q^+ - q_k^-\|^2 / \tau')} $$
    *Note: The source text provides a garbled formula for $\lambda$ and the final loss. The standard InfoNCE loss is implied. The weight $\lambda_{q^-}$ acts as a geometric prior. The final loss is averaged over all samples $p$ and all source view pairs.*
    $$ L_{contrast} = \frac{1}{N_c} \sum_{p} L_{contrast}^p $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_aware_contrastive_loss(
    features_i, features_j, 
    depth_i, depth_j,
    K_i, K_j, E_i, E_j,
    N_c, N_neg, tau_prime=10000
):
    """
    Computes the GeoContrast loss for a pair of views i and j.
    
    Args:
        features_i: Feature map for view i, shape (H, W, C)
        features_j: Feature map for view j, shape (H, W, C)
        depth_i: Depth map for view i, shape (H, W)
        depth_j: Depth map for view j, shape (H, W)
        K_i, K_j: Intrinsic matrices (3x3)
        E_i, E_j: Extrinsic matrices (4x4) or (R, t)
        N_c: Number of pixels to sample
        N_neg: Number of negative samples per pixel
        tau_prime: Temperature parameter
        
    Returns:
        loss: Scalar contrastive loss
    """
    H, W, C = features_i.shape
    
    # 1. Sample N_c pixels from view i
    # Randomly select indices
    sampled_u = random.randint(0, H-1, N_c)
    sampled_v = random.randint(0, W-1, N_c)
    
    total_loss = 0.0
    valid_count = 0
    
    for k in range(N_c):
        u, v = sampled_u[k], sampled_v[k]
        
        # 2. Get 3D point p3 from view i
        # Homogeneous coordinate of pixel p
        p_hom = np.array([u, v, 1.0])
        
        # Get depth
        delta_p = depth_i[u, v]
        
        # Camera matrices
        R_i, t_i = decompose_extrinsic(E_i)
        R_j, t_j = decompose_extrinsic(E_j)
        
        # Project to 3D: p3 = t_i + delta_p * R_i * K_i^-1 * p_hom
        # Note: K_i^-1 * p_hom gives normalized camera coordinates
        cam_coords_i = np.linalg.inv(K_i) @ p_hom
        p3 = t_i + delta_p * (R_i @ cam_coords_i)
        
        # 3. Project p3 to view j to get q+
        # p3 in camera j frame: R_j^-1 * (p3 - t_j)
        cam_coords_j = np.linalg.inv(R_j) @ (p3 - t_j)
        # Project to 2D
        q_plus_hom = K_j @ cam_coords_j
        q_plus = q_plus_hom[:2] / q_plus_hom[2]
        
        # 4. Occlusion Check
        # Check if q+ is within bounds and if the depth at q+ in view j
        # is consistent with the projected depth of p3.
        # Simple check: is q+ inside image bounds?
        if not is_inside_image(q_plus, H, W):
            continue
            
        # Check occlusion: compare projected depth with ground truth depth at q+
        # Project p3 back to depth in view j
        projected_depth_j = get_depth_from_3d(p3, K_j, E_j)
        gt_depth_q_plus = depth_j[int(round(q_plus[1])), int(round(q_plus[0]))]
        
        # Allow some tolerance for depth consistency
        if abs(projected_depth_j - gt_depth_q_plus) > tolerance:
            continue # Occluded or inconsistent
            
        # 5. Get Negative Samples
        # Sample N_neg pixels from view j as negatives
        neg_u = random.randint(0, H-1, N_neg)
        neg_v = random.randint(0, W-1, N_neg)
        q_negs = []
        for m in range(N_neg):
            q_negs.append(np.array([neg_u[m], neg_v[m]]))
            
        # 6. Compute Weights and Loss
        # Positive distance (should be 0 or very small if q+ is exact)
        dist_pos = np.linalg.norm(q_plus - q_plus) # 0
        
        # Negative distances
        dist_negs = [np.linalg.norm(q_plus - q_neg) for q_neg in q_negs]
        
        # Calculate weights lambda_q-
        # lambda_q- = exp(||q+ - q-||^2 / tau') / sum(exp(||q+ - r||^2 / tau'))
        # Note: The denominator usually includes the positive term in InfoNCE,
        # but the text defines lambda specifically. We follow the text's structure.
        # Assuming the loss is a weighted cross-entropy or similar.
        
        # Standard InfoNCE with geometric weighting:
        # L = -log( exp(dist_pos/tau) / (exp(dist_pos/tau) + sum(lambda_q- * exp(dist_neg/tau)) ) )
        # However, the text defines lambda as a softmax over negatives.
        # Let's implement the weighted contrastive loss as implied:
        
        exp_pos = np.exp(-dist_pos / tau_prime) # Negative distance for similarity
        exp_negs = [np.exp(-d / tau_prime) for d in dist_negs]
        
        # Calculate lambda for each negative
        sum_exp_negs = sum(exp_negs)
        if sum_exp_negs == 0:
            continue
            
        lambdas = [e / sum_exp_negs for e in exp_negs]
        
        # Weighted sum of negative similarities
        weighted_neg_sim = sum(l * e for l, e in zip(lambdas, exp_negs))
        
        # Loss for this pixel
        # L_p = -log( exp_pos / (exp_pos + weighted_neg_sim) )
        # Note: dist_pos is 0, so exp_pos = 1.
        loss_p = -np.log( exp_pos / (exp_pos + weighted_neg_sim) )
        
        total_loss += loss_p
        valid_count += 1
        
    if valid_count == 0:
        return 0.0
        
    return total_loss / valid_count
```

### Parameters
*   $N_c$: Number of sampled pixels. Not given in the sources.
*   $N_{neg}$: Number of negative samples. Not given in the sources.
*   $\tau'$: Temperature hyper-parameter. Default value is 10000 [S1].

### Edge Cases and Pitfalls
*   **Occlusion:** Positive pairs must be discarded if the projected point $q^+$ is occluded in the target view. Failure to check occlusion leads to incorrect positive pairs [S1].
*   **Depth Precision:** The 3D reconstruction $p_3$ relies on ground truth depth $\delta_p$. Errors in depth lead to incorrect $p_3$ and thus incorrect $q^+$ [S1].
*   **Boundary Conditions:** Projected points $q^+$ or $q^-$ may fall outside the image boundaries. These must be handled or discarded [G].

## Subtopics

### GeCoNeRF
The parent method that provides the feature extraction backbone and depth prediction module. GeoContrast is a regularization term added to GeCoNeRF's training objective. [GeCoNeRF](8905-geconerf.md)

### ContraNeRF
The broader class of methods using contrastive learning for NeRF generalization. GeoContrast is a specific variant that uses geometry-aware sampling rather than random sampling. [ContraNeRF](https://doi.org/10.1109/cvpr52729.2023.01584)

## Operating characteristics
*   **Time Complexity:** Depends on the number of sampled pixels $N_c$ and negative samples $N_{neg}$. For each pixel, 3D projection and re-projection are performed.
*   **Memory Complexity:** Requires storing feature maps for source views and depth maps.
*   **Strengths:** Enforces geometric consistency, reducing artifacts in novel view synthesis, especially in few-shot settings.
*   **Limitations:** Relies on ground truth depth for training, which may not be available in real-world scenarios without additional depth estimation. Computationally more expensive than random sampling due to 3D projections.

## Background
GeoContrast was introduced in the paper "ContraNeRF: Generalizable Neural Radiance Fields for Synthetic-to-real Novel View Synthesis via Contrastive Learning" [S1]. It addresses the problem of generalizing NeRF to unseen scenes by leveraging contrastive learning to enforce multi-view consistency. The method builds on the GeCoNeRF framework, enhancing it with geometry-aware contrastive pairs.

## Sources
[S1] ContraNeRF: Generalizable Neural Radiance Fields for Synthetic-to-real Novel View Synthesis via Contrastive Learning, https://doi.org/10.1109/cvpr52729.2023.01584, fetched 2026-10-29
