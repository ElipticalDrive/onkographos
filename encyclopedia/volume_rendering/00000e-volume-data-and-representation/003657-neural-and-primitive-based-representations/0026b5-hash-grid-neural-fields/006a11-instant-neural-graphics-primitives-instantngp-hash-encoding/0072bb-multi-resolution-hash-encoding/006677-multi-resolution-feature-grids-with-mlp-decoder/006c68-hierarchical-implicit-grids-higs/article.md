# Hierarchical Implicit Grids (HIGS)

Also called: HIGS (Hierarchical Implicit Grids)

## Overview
Hierarchical Implicit Grids (HIGS) is a method for spatial scene understanding that captures both geometric (Signed Distance Field, SDF) and semantic (vision-language) information using a multiresolution feature grid structure [S1, unofficial]. It solves two primary tasks: SDF estimation for scene construction and open-vocabulary object grounding, aligning submaps directly in feature space via hierarchical optimization while avoiding explicit geometric reconstruction [S1, unofficial]. It sets itself apart from its siblings, such as [Multi-Resolution Feature Grids with MLP Decoder](19643-multi-resolution-feature-grids-with-mlp-decoder.md), by explicitly distinguishing between dense and sparse grid indexing strategies tailored to low-dimensional geometric features versus high-dimensional semantic features, and by employing a pre-trained, fixed decoder to prevent catastrophic forgetting during online mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $n$ keyframes with associated poses $\{T_{ks}\}_{k=1}^n$ in a reference frame, and observations $\{X^k\}_{k=1}^n$ (3D point clouds) and $\{Y^k\}_{k=1}^n$ (associated features, e.g., SDF values or vision-language features) for each frame.
*   **Outputs:** Optimized multiresolution feature grids $F$ and a continuous feature field $h(x; F, \theta)$ that predicts SDF or vision-language features for any query position $x \in \mathbb{R}^3$.

**Data Structures:**
The core data structure is the **Multiresolution Feature Grid**, containing $L > 1$ levels of regular grids with increasing spatial resolution from coarse ($l=1$) to fine ($l=L$) [S1, unofficial].

1.  **Feature Storage ($F_l$):** At each level $l$, features are stored as $F_l = \{f_{l,i}\}_{i=1}^{N_l} \in \mathbb{R}^{N_l \times d_l}$, where $d_l$ is the feature dimension at level $l$ [S1, unofficial].
2.  **Grid Indexing ($Q_l$):**
    *   **Dense Grid Indexing:** Used for low-dimensional features (e.g., SDF) where $N_l = L_l \times W_l \times H_l$. The index function is $q_l(i) = i_x \times L_l + i_y \times W_l + i_z \times H_l$, where $i_x, i_y, i_z$ are grid indices along the axes [S1, unofficial].
    *   **Sparse Grid Indexing:** Used for high-dimensional features (e.g., vision-language) where only near-surface grids store features. A lookup tensor $Q_l \in \mathbb{R}^{L_l \times W_l \times H_l}$ maps grid vertices to feature indices. Values in $Q_l$ are:
        *   $-1$: No associated feature (feature value is 0).
        *   $1 \dots N_l$: Index in the feature array $F_l$.
        *   The index function is $q_l(i) = Q_l(i_x, i_y, i_z)$ [S1, unofficial].

**Parameters:**
*   **Levels ($L$):** Default $L=2$ [S1, unofficial].
*   **Grid Resolution:** Default coarse level grid size is 0.5 cm; fine level is 0.1 cm [S1, unofficial].
*   **Feature Dimensions ($d_l$):** Default $d_l=4$ for SDF features; $d_l=64$ for vision-language features [S1, unofficial].
*   **Kernel Function ($k_l$):** Implemented using trilinear interpolation [S1, unofficial].
*   **SDF Surface Threshold:** Points within 30 cm of the surface are classified as "on or near surface" [S1, unofficial].
*   **Exponential Penalty Coefficient ($\beta$):** Default $\beta=5$, used in SDF free-space lower bound cost [S1, unofficial].
*   **SDF Surface Weight ($w_j^{sdf}$):** Default $w_j^{sdf}=5.4$ [S1, unofficial].

**Numbered Steps:**

1.  **Decoder Pre-training:** Pre-train the decoder network $D_\theta$ (implemented as an MLP) offline over multiple scenes. During online mapping, $\theta$ is fixed [S1, unofficial].
2.  **Feature Lifting (Vision-Language):**
    *   Run Segment Anything Model (SAM) on RGB images to obtain object masks [S1, unofficial].
    *   Crop masks into patches and run CLIP to extract features for each patch [S1, unofficial].
    *   Lift 3D points $\{X^k\}$ using depth images and associate them with CLIP features $\{Y^k\}$ [S1, unofficial].
3.  **Feature Lifting (SDF):**
    *   Sample points along rays from depth images or LiDAR [S1, unofficial].
    *   For near-surface points, compute SDF as the distance to the surface along the ray direction [S1, unofficial].
    *   For free-space points, compute lower ($b_j$) and upper ($\bar{b}_j$) bounds on the SDF [S1, unofficial].
4.  **Feature Field Interpolation:** For a query position $x$, compute the feature field at level $l$ using kernel interpolation:
    $$f_l(x) = \sum_{i \in I_l} k_l(x, z_{l,i}) f_{l, q_l(i)}$$
    where $z_{l,i}$ are grid vertices, $I_l$ indexes involved vertices, and $q_l(i)$ maps the vertex to the feature index [S1, unofficial].
5.  **Decoding:** Concatenate features from all levels and pass through the decoder:
    $$h(x; F, \theta) = D_\theta \left( \bigoplus_{l \in [L]} f_l(x) \right)$$
    where $\bigoplus$ denotes concatenation [S1, unofficial].
6.  **Local Feature Mapping Optimization:** Optimize the grid features $F$ by minimizing the total cost over all observations:
    $$\min_F \sum_{k=1}^n \sum_{j=1}^{m_k} c_j(h(T_{ks}x_{kj}; F))$$
    where $c_j$ is the cost function specific to the feature type (SDF or vision-language) [S1, unofficial].

**Cost Functions:**

*   **SDF Surface Cost:** For points on or near the surface:
    $$c_j^{sdf}(h^{sdf}(x_j)) = w_j^{sdf} (h^{sdf}(x_j) - y_j^{sdf})$$
    where $y_j^{sdf}$ is the measured SDF value [S1, unofficial].
*   **SDF Free-space Lower Bound Cost:**
    $$c_j^{sdf, lo}(h^{sdf}(x_j)) = \max(e^{\beta(b_j - h^{sdf}(x_j))} - 1, 0)$$
    where $b_j$ is the lower bound [S1, unofficial].
*   **SDF Free-space Upper Bound Cost:**
    $$c_j^{sdf, up}(h^{sdf}(x_j)) = \max(h^{sdf}(x_j) - \bar{b}_j, 0)$$
    where $\bar{b}_j$ is the upper bound [S1, unofficial].
*   **SDF Bound Cost:** Combined free-space cost:
    $$c_j^{sdf, bnd}(h^{sdf}(x_j)) = \max(c_j^{sdf, lo}(h^{sdf}(x_j)), c_j^{sdf, up}(h^{sdf}(x_j)))$$
    [S1, unofficial].
*   **Vision-Language Cost:** For on-surface points:
    $$c_j^{vl}(h^{vl}(x_j)) = 1 - \cos(h^{vl}(x_j), y_j^{vl})$$
    using cosine similarity loss between predicted and CLIP features [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Eikonal Regularization:** Eikonal regularization is explicitly excluded because it has limited impact on accuracy while making optimization slower [S1, unofficial].
*   **Sparse Grid Handling:** For sparse grids, if $q_l(i) = -1$, the associated feature $f_{l, q_l(i)}$ is treated as 0, ensuring it does not affect the interpolation sum [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def trilinear_interpolation(grid_features, grid_vertices, query_point, lookup_tensor=None, is_sparse=False):
    """
    Computes the interpolated feature value at a query point using trilinear interpolation.
    
    Args:
        grid_features: Array of shape (N_l, d_l) containing feature vectors.
        grid_vertices: Array of shape (N_l, 3) containing vertex coordinates.
        query_point: Array of shape (3,) containing the query position x.
        lookup_tensor: Optional tensor Q_l for sparse indexing. If None, assumes dense.
        is_sparse: Boolean flag indicating if sparse indexing is used.
        
    Returns:
        Interpolated feature vector of shape (d_l,).
    """
    # Determine grid dimensions from vertices or lookup tensor
    if is_sparse and lookup_tensor is not None:
        # Sparse grid: lookup_tensor is (L_l, W_l, H_l)
        # Find the 8 surrounding grid vertices for trilinear interpolation
        # This requires mapping query_point to grid indices and finding neighbors
        # For brevity, assuming a helper function exists to get neighbor indices and weights
        neighbor_indices, weights = get_trilinear_neighbors(query_point, lookup_tensor.shape)
        
        interpolated_feature = np.zeros_like(grid_features[0])
        for idx, weight in zip(neighbor_indices, weights):
            # Map grid index to feature index using lookup tensor
            feat_idx = lookup_tensor[idx[0], idx[1], idx[2]]
            if feat_idx != -1:
                interpolated_feature += weight * grid_features[feat_idx]
            # If feat_idx == -1, feature is 0, so no addition needed
        return interpolated_feature
    else:
        # Dense grid: direct indexing
        # Find the 8 surrounding grid vertices
        neighbor_indices, weights = get_trilinear_neighbors(query_point, grid_vertices.shape[0])
        
        interpolated_feature = np.zeros_like(grid_features[0])
        for idx, weight in zip(neighbor_indices, weights):
            # Dense index calculation: i_x * L_l + i_y * W_l + i_z * H_l
            # Assuming idx is a flat index or tuple that can be converted
            # For simplicity, assuming idx is a flat index into grid_features
            interpolated_feature += weight * grid_features[idx]
        return interpolated_feature

def get_trilinear_neighbors(query_point, grid_shape):
    """
    Helper to find the 8 surrounding grid vertices and their interpolation weights.
    Returns indices and weights.
    Note: In a real implementation, this would involve floor/ceil operations on grid coordinates.
    """
    # Placeholder for actual trilinear neighbor calculation logic
    # This function is not defined in the source, but is necessary for implementation.
    # [G] Implementation detail: Calculate fractional coordinates, determine integer bounds,
    # compute 8 corner indices, and calculate barycentric weights.
    raise NotImplementedError("Trilinear neighbor calculation logic not provided in source.")

def compute_sdf_surface_cost(predicted_sdf, measured_sdf, weight=5.4):
    """
    Computes the SDF surface cost.
    """
    return weight * (predicted_sdf - measured_sdf)

def compute_sdf_free_space_cost(predicted_sdf, lower_bound, upper_bound, beta=5):
    """
    Computes the combined SDF free-space cost (lower and upper bounds).
    """
    # Lower bound cost: exponential penalty
    lower_cost = np.maximum(np.exp(beta * (lower_bound - predicted_sdf)) - 1, 0)
    # Upper bound cost: linear penalty
    upper_cost = np.maximum(predicted_sdf - upper_bound, 0)
    # Combined cost: max of lower and upper
    return np.maximum(lower_cost, upper_cost)

def compute_vision_language_cost(predicted_vl_feature, measured_vl_feature):
    """
    Computes the vision-language cost using cosine similarity.
    """
    # Cosine similarity
    dot_product = np.dot(predicted_vl_feature, measured_vl_feature)
    norm_pred = np.linalg.norm(predicted_vl_feature)
    norm_meas = np.linalg.norm(measured_vl_feature)
    cosine_sim = dot_product / (norm_pred * norm_meas + 1e-8) # Add epsilon for stability
    return 1 - cosine_sim

def decode_features(concatenated_features, decoder_model):
    """
    Passes concatenated features through the pre-trained decoder.
    """
    # decoder_model is a pre-trained MLP
    return decoder_model(concatenated_features)

def optimize_higs(keyframes, poses, observations, decoder_model, grid_configs):
    """
    Main optimization loop for HIGS.
    
    Args:
        keyframes: List of keyframe data.
        poses: List of pose matrices T_ks.
        observations: List of (points, features) tuples.
        decoder_model: Pre-trained MLP decoder.
        grid_configs: Configuration for grid levels (sizes, dimensions, sparsity).
        
    Returns:
        Optimized feature grids F.
    """
    # Initialize feature grids F based on grid_configs
    # [G] Hierarchical Feature Initialization is a separate method; here we assume 
    # initial values are set (e.g., zeros or from prior levels if using HFI).
    F = initialize_feature_grids(grid_configs)
    
    # Pre-compute lifted 3D points and features for each keyframe
    # This step involves SAM, CLIP, and depth lifting as described in Feature Lifting steps
    lifted_data = []
    for k, (keyframe, pose) in enumerate(zip(keyframes, poses)):
        points_3d, features_3d = lift_features(keyframe, pose)
        lifted_data.append((points_3d, features_3d))
        
    # Optimization loop (e.g., using Adam or SGD)
    for iteration in range(max_iterations):
        total_loss = 0
        for k, (points_3d, features_3d) in enumerate(lifted_data):
            pose = poses[k]
            for j, (point, feature) in enumerate(zip(points_3d, features_3d)):
                # Transform point to submap reference frame
                transformed_point = pose @ point
                
                # Interpolate features from all levels
                concatenated_features = []
                for l in range(len(grid_configs)):
                    config = grid_configs[l]
                    is_sparse = config['type'] == 'sparse'
                    lookup_tensor = config.get('lookup_tensor', None)
                    grid_features = F[l]
                    grid_vertices = config['vertices']
                    
                    f_l = trilinear_interpolation(
                        grid_features, 
                        grid_vertices, 
                        transformed_point, 
                        lookup_tensor, 
                        is_sparse
                    )
                    concatenated_features.append(f_l)
                
                # Concatenate features from all levels
                concat_feat = np.concatenate(concatenated_features)
                
                # Decode
                predicted_feature = decode_features(concat_feat, decoder_model)
                
                # Compute cost based on feature type
                if feature_type == 'sdf':
                    # Determine if point is on surface or free space
                    if is_near_surface(point, threshold=0.3): # 30 cm
                        cost = compute_sdf_surface_cost(predicted_feature, feature)
                    else:
                        lower_bound, upper_bound = compute_sdf_bounds(point)
                        cost = compute_sdf_free_space_cost(predicted_feature, lower_bound, upper_bound)
                elif feature_type == 'vl':
                    cost = compute_vision_language_cost(predicted_feature, feature)
                
                total_loss += cost
        
        # Update grid features F using gradient descent
        gradients = compute_gradients(total_loss, F)
        update_features(F, gradients)
        
    return F
```

## Subtopics

### [Hierarchical Feature Initialization](21224-hierarchical-feature-initialization.md)
A method for initializing latent feature grids $F_l$ at level $l$ by leveraging optimization results from previous coarser levels ($F_{1:l-1}$), reducing computational cost compared to optimizing high-dimensional features from scratch. It sets itself apart from naive initialization by using a learned encoder to map residuals from prior levels to the current level's feature space, rather than using zeros or random values.

## Operating characteristics
*   **Time Complexity:** The optimization cost is dominated by the high-dimensional grid features. Hierarchical initialization helps reduce this cost by leveraging coarser levels [S1, unofficial].
*   **Memory Complexity:** Sparse grids are used for high-dimensional features (e.g., vision-language) to save memory, storing features only for near-surface grids [S1, unofficial]. Dense grids are used for low-dimensional features (e.g., SDF) [S1, unofficial].
*   **Performance:** The method avoids online decoder training, which prevents catastrophic forgetting and reduces computational burden [S1, unofficial].
*   **Strengths:** Jointly captures geometric and semantic information; aligns submaps directly in feature space; efficient memory usage for high-dimensional features via sparsity.
*   **Limitations:** Excludes Eikonal regularization, which may limit accuracy in some scenarios; relies on pre-trained decoder which may not generalize to unseen domains without retraining [S1, unofficial].
*   **When to use:** Suitable for spatial scene understanding tasks requiring both SDF estimation and open-vocabulary object grounding, particularly in robotics applications using RGB-D frames [S1, unofficial].
*   **When not to use:** Not suitable for tasks requiring explicit geometric reconstruction or where Eikonal regularization is critical for accuracy [S1, unofficial].

## Background
HIGS was introduced in the paper "HIGS: Hierarchical Implicit Grids for Joint Geometric and Semantic Scene Understanding" [S1, unofficial]. It is used in robotics for spatial scene understanding, specifically for SDF estimation (scene construction via marching cubes) and open-vocabulary object grounding (3D bounding box prediction) [S1, unofficial]. The method builds on the concept of multiresolution feature grids, extending them with specific indexing strategies and cost functions for joint geometric and semantic learning [S1, unofficial].

## Sources
[S1] HIGS: Hierarchical Implicit Grids for Joint Geometric and Semantic Scene Understanding, https://doi.org/10.48550/arxiv.2609.38620, fetched 2026-10-26 (unofficial)
