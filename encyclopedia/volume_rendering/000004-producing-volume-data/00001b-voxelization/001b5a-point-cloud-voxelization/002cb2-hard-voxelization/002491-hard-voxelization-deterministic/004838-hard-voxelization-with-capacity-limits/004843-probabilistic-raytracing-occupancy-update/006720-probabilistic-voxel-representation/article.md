# Probabilistic Voxel Representation

Also called: OpenVox, OpenVox Probabilistic Voxel Representation

## Overview
Probabilistic Voxel Representation is a real-time, instance-level open-vocabulary mapping method that enhances robustness in incremental 3D reconstruction by modeling voxel occupancy and semantic identity through probabilistic distributions [S1]. Unlike deterministic hard-voxel approaches that suffer from compounding errors due to segmentation inaccuracies, this method uses Maximum Likelihood Estimation (MLE) for instance association and Maximum A Posteriori (MAP) estimation for map evolution, allowing it to handle under-segmentation and poor viewing angles gracefully [S1]. It builds on [Probabilistic Raytracing Occupancy Update](12215-probabilistic-raytracing-occupancy-update.md) by extending probabilistic modeling from binary occupancy to multi-instance semantic vectors and codebook features.

## How it works
The method operates in a loop of **Instance Association** and **Live Map Evolution** for each incoming frame observation $Q_t = \{ \{m_{ti}, f_{ti}\}, D_t, P_t \}$, where $m_{ti}$ is a 2D instance mask, $f_{ti}$ is its feature embedding, $D_t$ is the depth image, and $P_t$ is the camera pose [S1].

### Inputs and Outputs
*   **Inputs**: Current frame masks $\{m_{ti}\}$, mask features $\{f_{ti}\}$, depth map $D_t$, camera pose $P_t$, camera intrinsics $K$. Existing map $M_{t-1} = \{V_{t-1}, B_{t-1}\}$, where $V_{t-1}$ is the voxel map and $B_{t-1}$ is the embedding codebook.
*   **Outputs**: Updated map $M_t = \{V_t, B_t\}$ and instance assignments $I_t$.

### Data Structures
1.  **Voxel Map $V$**: A 3D grid with resolution $0.04\text{m}$ [S1]. Each voxel $v_j$ stores a **probabilistic instance ID vector** $\theta_j = \{ \theta_{j,\gamma} \mid \gamma \in \Gamma \}$, where $\Gamma$ is the set of known instance IDs.
    *   $\theta_{j,\gamma} > 0$
    *   $\sum_{\gamma \in \Gamma} \theta_{j,\gamma} = 1$
    *   $\theta_{j,\gamma}$ represents the probability that voxel $v_j$ belongs to instance $\gamma$ [S1].
2.  **Embedding Codebook $B$**: A dictionary or array associating each instance ID $\gamma$ with a fused caption feature vector $f_\gamma$ and a weight $W_\gamma$ [S1].
    *   $f_\gamma$: Feature vector for instance $\gamma$.
    *   $W_\gamma$: Cumulative weight (credibility) for instance $\gamma$ [S1].

### Parameters
*   **Voxel Resolution**: $0.04\text{m}$ [S1].
*   **Similarity Threshold**: A fixed threshold for determining if an association is valid (value not specified in sources) [S1].
*   **Prior Probability**: Assumed uniform across all existing instances $\Gamma_{t-1}$ for MLE [S1].

### Algorithm Steps

1.  **Voxel Region Projection**:
    For each mask $m_{ti}$ with pixel coordinates $[u, v]$, project the mask into the voxel map using the depth $D_t$ and camera intrinsics $K$:
    $$ V_{m_{ti}} = \text{Vox}( D_t[u,v] \cdot P_t \cdot K^{-1} \cdot [u, v]^T \mid [u, v] \in m_{ti} ) $$
    where $\text{Vox}(\cdot)$ transforms 3D points to voxel indices [S1].

2.  **Instance Association**:
    *   **Case A: Unobserved Region**: If $V_{m_{ti}}$ contains no voxels with existing instance information (or is empty), add a new instance ID $\gamma_{new}$ to $\Gamma_{t-1}$ and initialize its embedding in $B_{t-1}$ using $f_{ti}$ [S1].
    *   **Case B: Observed Region**: If $V_{m_{ti}}$ contains voxels with instance information, determine the best matching existing instance $\gamma \in \Gamma_{t-1}$:
        1.  **Geometric Similarity ($SI_{geo}$)**: Compute the average likelihood probability of the voxels in $V_{m_{ti}}$ for instance $\gamma$. To avoid numerical underflow from multiplying probabilities, use the law of large numbers by averaging:
            $$ SI_{geo}^{i=\gamma} = \mathbb{E}(\{ \theta_{m_i}^j [\gamma] \}) = \frac{1}{|V_{m_{ti}}|} \sum_{v_j \in V_{m_{ti}}} \theta_{j,\gamma} $$
            where $\theta_{j,\gamma}$ is the probability of voxel $v_j$ belonging to instance $\gamma$ from the previous map [S1].
        2.  **Feature Similarity ($SI_{fea}$)**: Compute the cosine similarity between the mask feature $f_{ti}$ and the codebook feature $f_\gamma$ of instance $\gamma$ [S1].
        3.  **Association Probability ($A$)**: Fuse the similarities. The sources mention "weighted fusion" but do not provide the specific weights or formula for combining $SI_{geo}$ and $SI_{fea}$ into $A_{I_{ti}=\gamma}$ [S1].
        4.  **Decision**: If $\max_{\gamma} (A_{I_{ti}=\gamma})$ exceeds the similarity threshold, associate $m_{ti}$ with the corresponding instance ID $I_{ti}$. Otherwise, initialize a new instance [S1].

3.  **Live Map Evolution (Voxel Update)**:
    Update the probabilistic instance vector $\theta_j$ for each voxel $v_j \in V_{m_{ti}}$ using MAP estimation with a Dirichlet conjugate prior [S1].
    *   Let $\alpha_{t-1, j, \gamma}$ be the prior parameter for voxel $v_j$ and instance $\gamma$.
    *   Let $y_{tj, \gamma}$ be a one-hot encoded measurement tuple indicating the association of the current frame's instance $I_{ti}$ to instance $\gamma$ (i.e., $y_{tj, I_{ti}} = 1$, others 0) [S1].
    *   Update the parameter $\alpha$:
        $$ \alpha_{t, j, \gamma} = \alpha_{t-1, j, \gamma} + y_{tj, \gamma} $$
    *   Compute the new probabilistic instance vector $\theta_{t, j}$ as the expected value of the posterior Dirichlet distribution:
        $$ \theta_{t, j, \gamma} = \frac{\alpha_{t, j, \gamma}}{\sum_{\tau \in \Gamma} \alpha_{t, j, \tau}} $$
    This model is referred to as the "instance counting sensor model" [S1].

4.  **Live Map Evolution (Codebook Update)**:
    Update the embedding codebook $B$ for the associated instance $I_{ti}$ [S1].
    *   **Visibility Ratio ($R_{ti}$)**: Calculate the proportion of the instance's size observed by the current mask relative to the total instance size. This prevents contamination from poor viewing angles (e.g., only a corner visible) [S1].
    *   **Credibility Weight ($w_{ti}$)**:
        $$ w_{ti} = A_{I_{ti}} \cdot R_{ti} $$
        where $A_{I_{ti}}$ is the association probability for the chosen instance [S1].
    *   **Weight Update**:
        $$ W_{t}^{I_{ti}} = W_{t-1}^{I_{ti}} + w_{ti} $$
    *   **Feature Update**:
        $$ f_{t}^{I_{ti}} = \frac{W_{t-1}^{I_{ti}} f_{t-1}^{I_{ti}} + w_{ti} f_{ti}}{W_{t}^{I_{ti}}} $$
    For instances not associated with the current mask, $W$ and $f$ remain unchanged [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_voxel_representation_step(frame_masks, frame_features, depth_map, camera_pose, camera_intrinsics, existing_map):
    """
    Performs one step of probabilistic voxel mapping.
    
    Args:
        frame_masks: List of 2D masks m_ti.
        frame_features: List of feature vectors f_ti corresponding to masks.
        depth_map: Depth image D_t.
        camera_pose: Camera pose P_t.
        camera_intrinsics: Camera intrinsics K.
        existing_map: Tuple (V, B, Gamma, alpha)
            V: List of voxels, each with theta (prob vector).
            B: Dict mapping instance_id -> (feature, weight).
            Gamma: Set of existing instance IDs.
            alpha: Dict mapping (voxel_idx, instance_id) -> alpha_param.
            
    Returns:
        updated_map: Tuple (V, B, Gamma, alpha)
    """
    V, B, Gamma, alpha = existing_map
    new_map_V = [copy(v) for v in V] # Deep copy to avoid in-place mutation issues during read
    new_map_B = copy(B)
    new_map_Gamma = set(Gamma)
    new_map_alpha = copy(alpha)
    
    # Precompute voxel indices for all pixels in all masks
    # Vox() function projects 3D point to voxel index
    # P_t * K^-1 * [u, v, 1]^T gives ray direction/origin, scaled by depth
    
    for i, (mask, feature) in enumerate(zip(frame_masks, frame_features)):
        # 1. Project mask to voxel region
        voxel_indices_in_mask = []
        for u, v in mask.get_pixel_coordinates():
            # Get depth at pixel
            d = depth_map[u, v]
            if d is None or d <= 0:
                continue
            # Project pixel to 3D point
            # Assuming standard pinhole model: X = K^-1 * [u, v, 1] * d
            # Then transform to world: P_t * X
            # Then voxelize
            point_3d = project_pixel_to_world(u, v, d, camera_intrinsics, camera_pose)
            voxel_idx = voxelize_point(point_3d, resolution=0.04)
            voxel_indices_in_mask.append(voxel_idx)
            
        if not voxel_indices_in_mask:
            continue
            
        # 2. Instance Association
        # Check if region is unobserved
        is_unobserved = True
        for v_idx in voxel_indices_in_mask:
            # Check if any instance in Gamma has significant probability in this voxel
            # Simplified: if max(theta) > epsilon for any gamma in Gamma
            if has_instance_info(new_map_V[v_idx], Gamma):
                is_unobserved = False
                break
                
        if is_unobserved:
            # Initialize new instance
            new_instance_id = generate_new_instance_id(Gamma)
            new_map_Gamma.add(new_instance_id)
            new_map_B[new_instance_id] = (copy(feature), 0.0) # Initialize weight 0 or small epsilon? Source says initialize embedding.
            # Note: Source says "initialize the new instance embedding in codebook". 
            # Weight update happens in step 4. We might need to set initial weight to w_ti later or handle separately.
            # For now, assume weight starts at 0 and gets added.
            associated_instance_id = new_instance_id
        else:
            # Solve MLE / Find best match
            best_gamma = None
            best_score = -1
            
            for gamma in Gamma:
                # Geometric Similarity
                # SI_geo = average of theta[v_idx][gamma] for v_idx in voxel_indices_in_mask
                geo_sim = 0.0
                count = 0
                for v_idx in voxel_indices_in_mask:
                    if v_idx in new_map_V:
                        geo_sim += new_map_V[v_idx].theta[gamma]
                        count += 1
                if count > 0:
                    geo_sim /= count
                else:
                    geo_sim = 0.0
                    
                # Feature Similarity
                # Cosine similarity between feature and B[gamma].feature
                if gamma in new_map_B:
                    feat_sim = cosine_similarity(feature, new_map_B[gamma].feature)
                else:
                    feat_sim = 0.0
                    
                # Association Probability (Weighted Fusion)
                # Source does not specify weights. Using equal weights as placeholder [G].
                # In practice, these weights might be learned or tuned.
                score = 0.5 * geo_sim + 0.5 * feat_sim
                
                if score > best_score:
                    best_score = score
                    best_gamma = gamma
                    
            # Check threshold
            if best_score > SIMILARITY_THRESHOLD:
                associated_instance_id = best_gamma
            else:
                # Initialize new instance
                new_instance_id = generate_new_instance_id(Gamma)
                new_map_Gamma.add(new_instance_id)
                new_map_B[new_instance_id] = (copy(feature), 0.0)
                associated_instance_id = new_instance_id
                
        # 3. Live Map Evolution: Voxel Update
        # Update alpha and theta for voxels in the mask region
        for v_idx in voxel_indices_in_mask:
            if v_idx not in new_map_V:
                continue
                
            # One-hot encoded measurement tuple y_tj
            # y_tj[gamma] = 1 if gamma == associated_instance_id else 0
            # Update alpha
            for gamma in Gamma: # Iterate over all known instances to update alpha
                # Note: Gamma might have grown if new instance was created
                if gamma in new_map_alpha:
                    key = (v_idx, gamma)
                    if key not in new_map_alpha:
                        new_map_alpha[key] = 0.0 # Initialize if missing
                    
                    # y_tj, gamma is 1 if gamma is the associated instance, else 0
                    y_tj_gamma = 1.0 if gamma == associated_instance_id else 0.0
                    
                    new_map_alpha[key] = new_map_alpha[key] + y_tj_gamma
                    
            # Recompute theta for this voxel
            # theta_j,gamma = alpha_j,gamma / sum(alpha_j,tau)
            total_alpha = 0.0
            for gamma in new_map_Gamma:
                key = (v_idx, gamma)
                total_alpha += new_map_alpha.get(key, 0.0)
                
            if total_alpha > 0:
                for gamma in new_map_Gamma:
                    key = (v_idx, gamma)
                    new_map_V[v_idx].theta[gamma] = new_map_alpha.get(key, 0.0) / total_alpha
            else:
                # Uniform distribution if no evidence
                for gamma in new_map_Gamma:
                    new_map_V[v_idx].theta[gamma] = 1.0 / len(new_map_Gamma)

        # 4. Live Map Evolution: Codebook Update
        if associated_instance_id in new_map_B:
            # Visibility Ratio R_ti
            # Source: proportion of instance's size observed. 
            # Implementation detail: Not given how to compute total instance size. 
            # Assume R_ti is provided or approximated by fraction of voxels in mask vs total voxels of instance [G].
            # For this pseudocode, we assume a function get_visibility_ratio exists.
            R_ti = get_visibility_ratio(associated_instance_id, voxel_indices_in_mask, new_map_V)
            
            # Credibility weight w_ti
            w_ti = best_score * R_ti # best_score is A_I_ti
            
            # Update Weight
            current_feat, current_weight = new_map_B[associated_instance_id]
            new_weight = current_weight + w_ti
            
            # Update Feature
            # f_t = (W_{t-1} * f_{t-1} + w_ti * f_ti) / W_t
            new_feat = (current_weight * current_feat + w_ti * feature) / new_weight
            
            new_map_B[associated_instance_id] = (new_feat, new_weight)

    return (new_map_V, new_map_B, new_map_Gamma, new_map_alpha)

def project_pixel_to_world(u, v, depth, K, P):
    # Standard projection inverse
    # x = (u - cx) / fx
    # y = (v - cy) / fy
    # z = depth
    # X_cam = [x, y, z, 1]
    # X_world = P * X_cam
    pass

def voxelize_point(point_3d, resolution):
    # Convert world coordinates to voxel indices
    pass

def has_instance_info(voxel, Gamma):
    # Check if any gamma in Gamma has theta > epsilon
    pass

def cosine_similarity(v1, v2):
    pass

def generate_new_instance_id(Gamma):
    pass

def get_visibility_ratio(instance_id, voxel_indices_in_mask, V):
    # Placeholder for visibility ratio calculation
    # Returns a value between 0 and 1
    return 1.0
```

### Edge Cases and Pitfalls
*   **Numerical Underflow**: Multiplying likelihood probabilities of independent voxels causes underflow. Mitigated by averaging likelihood probabilities (geometric similarity) [S1].
*   **Poor Viewing Contamination**: Features from partial views (e.g., only a corner visible) can corrupt the global codebook. Mitigated by weighting updates with the visibility ratio $R_{ti}$ [S1].
*   **Inaccurate Segmentation**: If the front-end segmentation is inaccurate or under-segmented, deterministic methods fail. Probabilistic modeling provides robustness by allowing soft associations [S1].

## Operating characteristics
*   **Time Complexity**: Depends on the number of voxels in the mask region and the number of instances in $\Gamma$. The MLE step iterates over all instances for each mask, and the voxel update iterates over all voxels in the mask region. Real-time performance is claimed on an RTX 4090 [S1].
*   **Memory Complexity**: Proportional to the number of voxels times the number of instances (for $\theta$ vectors) plus the size of the codebook. The voxel resolution of $0.04\text{m}$ balances precision and memory [S1].
*   **Strengths**: Robust to segmentation errors, handles open-vocabulary semantics via LLM captions, real-time capable.
*   **Limitations**: Requires accurate depth and camera poses. Performance degrades if visibility ratio estimation is poor. Computationally heavier than hard-voxel methods due to probabilistic updates [S1].
*   **When to use**: Incremental mapping where robustness to segmentation errors is critical, and open-vocabulary semantic understanding is required.
*   **When not to use**: Resource-constrained environments where memory for large instance vectors is prohibitive, or when high-precision geometric reconstruction is prioritized over semantic robustness.

## Background
Probabilistic Voxel Representation, specifically as implemented in OpenVox, was introduced in 2025 [S1]. It addresses the limitations of previous incremental mapping methods like ConceptGraphs, which are prone to compounding errors from inaccurate segmentation [S1]. The method leverages probabilistic modeling (Dirichlet priors) for instance association and fusion, inspired by semantic counting sensor models [S1]. It is implemented in PyTorch and tested on robotics platforms [S1].

## Sources
[S1] OpenVox: Real-time Instance-level Open-vocabulary Probabilistic Voxel Representation, https://doi.org/10.1109/IROS60139.2025.11246455, fetched 2026-11-04
