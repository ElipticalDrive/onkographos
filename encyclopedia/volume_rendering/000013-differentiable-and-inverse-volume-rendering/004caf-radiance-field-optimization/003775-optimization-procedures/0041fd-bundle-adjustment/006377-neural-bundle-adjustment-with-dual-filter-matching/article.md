# Neural Bundle Adjustment with Dual-Filter Matching

Also called: BA with ROMA filtering, Neural Bundle Adjustment, neural-BA

## Overview
Neural Bundle Adjustment with Dual-Filter Matching is a refinement technique that improves 3D point cloud geometry and camera parameters (intrinsics and extrinsics) by integrating neural network-based geometry estimation with traditional bundle adjustment (BA) optimization [S1]. It solves the problem of inaccurate epipolar geometric constraints and sparse point clouds in sparse-view inputs, which standard neural methods cannot guarantee [S1]. Unlike its sibling [Joint Optimization of NeRF and Camera Parameters](7492-joint-optimization-of-nerf-and-camera-parameters.md) which optimizes radiance fields, this method explicitly refines the underlying 3D point structure and camera poses using 2D matching points filtered by confidence scores from DUSt3R and ROMA [S1].

## How it works
The method consists of three sequential steps: Neural Dense Initialization, Bundle Adjustment Refinement with Dual Filtering, and Semantic-based Local Refinement [S1].

### Inputs and Outputs
**Inputs:**
- A set of unposed input images $\{I_1, I_2, \dots, I_N\}$.
- DUSt3R model for initial 3D reconstruction.
- ROMA model for confidence filtering.
- Initial camera intrinsics $K$ (optional, if not pre-calibrated) and extrinsics $T$ (initially identity or estimated).

**Outputs:**
- Refined 3D point cloud $X^*$.
- Refined camera intrinsics $K^*$.
- Refined camera extrinsics $T^*$.

### Data Structures
- **Point Map ($P$):** A 3D tensor of shape $W \times H \times 3$, where $W$ and $H$ are image width and height. Each pixel $(i, j)$ contains the 3D coordinates $(P_{i,j,x}, P_{i,j,y}, P_{i,j,z})$ of the point corresponding to that pixel in the reference camera's coordinate frame [S1].
- **Confidence Map ($C$):** A 2D tensor of shape $W \times H$, where each element $C_{i,j}$ represents the confidence score of the 3D point at $(i, j)$ [S1].
- **Matching Points ($M$):** A set of 2D coordinate pairs $\{(i, j)\}$ representing corresponding points between image views [S1].
- **Visibility Term ($v_{ij}$):** A binary or continuous weight indicating if point $j$ is visible in view $i$ [S1].

### Parameters
- **DUSt3R Confidence Threshold:** 3 [S1].
- **ROMA Confidence Threshold:** 0.05 [S1].
- **Max Matching Points per View:** 50,000 (to limit computational complexity) [S1].

### Steps

#### 1. Neural Dense Initialization
This step generates an initial dense point cloud and estimates relative poses and focal length using DUSt3R [S1].

1.  **Reconstruct Point Maps:** For each image pair $\{I_k, I_l\}$, apply DUSt3R to obtain the point map $P$ and confidence map $C$:
    $$ P_{W \times H \times 3}, C_{W \times H} = \text{DUSt3R}(I_k, I_l) $$
    where $P \in \mathbb{R}^{W \times H \times 3}$ is the 3D point map and $C \in \mathbb{R}^{W \times H}$ is the confidence map [S1].

2.  **Estimate Relative Poses:** Solve for the transformation matrix $T_{k,l}$ and scaling factor $\sigma_{k,l}$ that aligns the point map from $P_l$ to $P_k$ by minimizing the registration problem:
    $$ \arg \min_{T_{k,l}, \sigma_{k,l}} \sum_{k \in K} \sum_{l \in K \setminus \{k\}} \sum_{i,j} C_{k,i,j} \cdot C_{l,i,j} \| P_{k,i,j} - \sigma_{k,l} T_{k,l} P_{l,i,j} \| $$
    This process is applied across all image pairs to transform all point maps into a unified coordinate system, generating a unified dense point cloud [S1].

3.  **Estimate Focal Length:** If the focal length $f$ is not pre-calibrated, calculate it by solving the minimization problem:
    $$ \arg \min_f \sum_{i=0}^{W} \sum_{j=0}^{H} C_{i,j} \left( \left(\frac{i}{W} - \frac{1}{2}\right)^2 + \left(\frac{j}{H} - \frac{1}{2}\right)^2 - f \cdot \frac{P_{i,j,x}^2 + P_{i,j,y}^2}{P_{i,j,z}} \right) $$
    where $C_{i,j}$ is the confidence value at pixel $(i, j)$, and $P_{i,j,x}, P_{i,j,y}, P_{i,j,z}$ are the components of the 3D point at $(i, j)$ [S1].

#### 2. Bundle Adjustment Refinement with Dual Filtering
This step introduces epipolar geometric constraints to correct inaccuracies in the initial point cloud [S1].

1.  **Extract Matching Points:** Iterate over all image pairs to identify corresponding 2D matching points $M_{k,l}$ using nearest neighbor matching on the point maps:
    $$ M_{k,l} = \{ (i, j) \mid i = NN_k^l(j) \text{ and } j = NN_l^k(i) \} $$
    where $NN_k^l(j)$ denotes the nearest neighbor matching point extracted from the point map of image $k$ for point $j$ in image $l$ [S1].

2.  **Dual-Filter Matching Points:** Filter the matching points to ensure high quality. Preserve only those points that satisfy both:
    - DUSt3R confidence $> 3$
    - ROMA confidence $> 0.05$
    This dual-filtering approach refines the location of 2D matching points and prevents failures in enforcing epipolar constraints due to inaccurate initial alignment [S1].

3.  **First Round of Bundle Adjustment:** Perform bundle adjustment to minimize the loss $L_{BA}$ with respect to the 3D points $X$, camera intrinsics $K$, and extrinsics $T$:
    $$ L_{BA} = \sum_{i=1}^{N_I} \sum_{j=1}^{N_x} v_{ij} \| P_i(x_j, K_i, T_i) - y_{ij} \|^2 $$
    where $N_I$ is the number of input views, $N_x$ is the number of matching points, $v_{ij}$ is the visibility term, $P_i(x_j, K_i, T_i)$ is the projection of 3D point $x_j$ with camera parameters $K_i, T_i$, and $y_{ij}$ is the observed 2D location [S1].
    The optimization is formulated as:
    $$ X^*, K^*, T^* = \text{BA}(M, X, K, T) = \arg \min_{X,K,T} L_{BA} $$
    Limit the number of matching points selected from each view to no more than 50,000 to manage computational complexity [S1].

4.  **Second Round of Bundle Adjustment:** Lower the confidence threshold for the remaining points $X'$ (those filtered out in step 2) and optimize their positions with fixed camera parameters $K^*, T^*$:
    $$ X^* = X^* \cup \text{BA}(M^*, X', K^*, T^*) $$
    where $M^*$ are the matching points for $X'$ with relaxed thresholds [S1].

#### 3. Semantic-based Local Refinement
This step addresses the reduced density of the point cloud after BA [S1].

1.  **Estimate Rigid Transformation:** Calculate a rigid transformation $T$ to align the initial point cloud $X$ with the optimized point cloud $X^*$:
    $$ T = \arg \min_T \| X - T X^* \| $$
    [S1].

2.  **Local Refinement:** Identify local 3D regions with large misalignment by measuring the point-wise distance $\| X - T X^* \|$. Refine these regions to correct local errors [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_bundle_adjustment_with_dual_filter(images, dust3r_model, roma_model):
    """
    Performs Neural Bundle Adjustment with Dual-Filter Matching.
    
    Args:
        images: List of input images {I_1, ..., I_N}.
        dust3r_model: DUSt3R model for 3D reconstruction.
        roma_model: ROMA model for confidence filtering.
        
    Returns:
        X_star: Refined 3D point cloud.
        K_star: Refined camera intrinsics.
        T_star: Refined camera extrinsics.
    """
    
    # Step 1: Neural Dense Initialization
    # Initialize unified point cloud and camera parameters
    unified_point_cloud = {}
    relative_poses = {}
    
    for k in range(len(images)):
        for l in range(k + 1, len(images)):
            I_k = images[k]
            I_l = images[l]
            
            # Get point map and confidence map from DUSt3R
            P_k, C_k = dust3r_model(I_k) # P_k is WxHx3, C_k is WxH
            P_l, C_l = dust3r_model(I_l)
            
            # Estimate relative pose T_kl and scale sigma_kl
            # Minimize: sum_{i,j} C_k[i,j] * C_l[i,j] * ||P_k[i,j] - sigma_kl * T_kl * P_l[i,j]||
            T_kl, sigma_kl = solve_registration(P_k, C_k, P_l, C_l)
            relative_poses[(k, l)] = (T_kl, sigma_kl)
            
            # Transform P_l to k's coordinate frame and add to unified cloud
            P_l_transformed = sigma_kl * T_kl @ P_l # Broadcasting for 3D points
            add_to_unified_cloud(unified_point_cloud, P_k, k)
            add_to_unified_cloud(unified_point_cloud, P_l_transformed, k)
            
    # Estimate focal length f if not pre-calibrated
    # Minimize: sum_{i,j} C[i,j] * ((i/W - 0.5)^2 + (j/H - 0.5)^2 - f * (P_x^2 + P_y^2)/P_z)
    f = solve_focal_length(unified_point_cloud)
    
    # Initialize camera parameters
    K = build_intrinsics(f)
    T = build_extrinsics(relative_poses)
    X = unified_point_cloud
    
    # Step 2: Bundle Adjustment Refinement with Dual Filtering
    matching_points_M = []
    
    for k in range(len(images)):
        for l in range(k + 1, len(images)):
            # Extract nearest neighbor matching points
            M_kl = extract_matching_points(P_k, P_l) # Uses NN matching
            
            # Dual-filtering
            filtered_M_kl = []
            for point in M_kl:
                # Get confidence from DUSt3R and ROMA
                conf_dust3r = get_confidence_dust3r(point)
                conf_roma = roma_model.confidence(point)
                
                if conf_dust3r > 3 and conf_roma > 0.05:
                    filtered_M_kl.append(point)
            
            matching_points_M.extend(filtered_M_kl)
            
    # Limit matching points per view to 50,000
    matching_points_M = limit_matching_points(matching_points_M, max_per_view=50000)
    
    # First round of BA
    X_star, K_star, T_star = bundle_adjustment(matching_points_M, X, K, T)
    
    # Second round of BA for remaining points with fixed camera params
    remaining_points_X_prime = get_remaining_points(X, X_star)
    remaining_matching_points_M_prime = get_matching_points_for(remaining_points_X_prime, matching_points_M)
    
    # Lower confidence threshold (implementation detail: re-run filtering with lower thresholds or use all points)
    # The source says "lower the confidence threshold", implying re-evaluation or inclusion of previously filtered points
    # Assuming we use the previously filtered set M but with relaxed criteria or just the remaining points
    # The formula X* = X* U BA(M*, X', K*, T*) suggests optimizing X' with fixed K*, T*
    
    X_prime_optimized = bundle_adjustment(remaining_matching_points_M_prime, remaining_points_X_prime, K_star, T_star, fixed_camera=True)
    X_star = union(X_star, X_prime_optimized)
    
    # Step 3: Semantic-based Local Refinement
    # Estimate rigid transformation T_align
    T_align = solve_rigid_alignment(X, X_star)
    
    # Refine local regions with large misalignment
    misalignment = distance(X, T_align @ X_star)
    local_refined_X_star = refine_local_regions(X_star, misalignment)
    
    return local_refined_X_star, K_star, T_star

def solve_registration(P_k, C_k, P_l, C_l):
    """
    Solves for T_kl and sigma_kl to align P_l to P_k.
    Minimize: sum_{i,j} C_k[i,j] * C_l[i,j] * ||P_k[i,j] - sigma_kl * T_kl * P_l[i,j]||
    """
    # Implementation of weighted Procrustes analysis or similar optimization
    # This is a standard geometric registration problem
    pass

def solve_focal_length(point_cloud):
    """
    Solves for focal length f.
    Minimize: sum_{i,j} C[i,j] * ((i/W - 0.5)^2 + (j/H - 0.5)^2 - f * (P_x^2 + P_y^2)/P_z)
    """
    # This is a linear least squares problem for f
    # Let A = sum C[i,j] * (P_x^2 + P_y^2)/P_z
    # Let B = sum C[i,j] * ((i/W - 0.5)^2 + (j/H - 0.5)^2)
    # f = B / A
    pass

def extract_matching_points(P_k, P_l):
    """
    Extracts nearest neighbor matching points between two point maps.
    M_kl = {(i, j) | i = NN_k^l(j) and j = NN_l^k(i)}
    """
    matching_points = []
    # Iterate over pixels in P_k
    for i, j in pixels(P_k):
        point_k = P_k[i, j]
        # Find nearest neighbor in P_l
        point_l = find_nearest_neighbor(point_k, P_l)
        # Check if reciprocal
        point_k_reciprocal = find_nearest_neighbor(point_l, P_k)
        if point_k_reciprocal == point_k:
            matching_points.append((i, j))
    return matching_points

def bundle_adjustment(matching_points, X, K, T, fixed_camera=False):
    """
    Performs bundle adjustment to minimize L_BA.
    L_BA = sum_{i,j} v_ij || P_i(x_j, K_i, T_i) - y_ij ||^2
    """
    # Use a standard BA optimizer (e.g., Levenberg-Marquardt)
    # If fixed_camera is True, only optimize X
    pass

def get_confidence_dust3r(point):
    """
    Retrieves confidence from DUSt3R's confidence map for a given point.
    """
    # Map 3D point back to 2D pixel coordinates to get confidence
    pass

def refine_local_regions(X_star, misalignment):
    """
    Refines local regions with large misalignment.
    """
    # Identify regions with high misalignment
    # Apply local optimization or smoothing
    pass
```

## Operating characteristics
- **Time Complexity:** Global bundle adjustment has high computational complexity, which is mitigated by limiting the number of matching points per view to no more than 50,000 [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** The method improves geometry accuracy and point cloud density under sparse-view inputs compared to methods without explicit geometry constraints [S1].
- **Strengths:**
    - Combines neural network-based geometry estimation with traditional BA for improved accuracy [S1].
    - Dual-filtering with ROMA prevents incorrect matches and failure in enforcing epipolar constraints [S1].
    - Handles unposed image pairs directly [S1].
- **Limitations:**
    - High computational complexity of global BA [S1].
    - Initial inaccurate alignment can cause incorrect matches if not filtered [S1].
    - Point cloud density may be lower after BA than the initial DUSt3R output, requiring local refinement [S1].
- **When to use:** When sparse-view inputs require high-accuracy 3D point clouds and camera poses, and epipolar geometric constraints are critical [S1].
- **When not to use:** When computational resources are severely limited and approximate geometry is acceptable [S1].

## Background
The method was proposed in the paper "GBR: Generative Bundle Refinement for High-Fidelity Gaussian Splatting With Enhanced Mesh Reconstruction" [S1]. It addresses the limitations of neural network-based methods that lack explicit geometry constraints, which can lead to inaccurate camera poses and point clouds [S1]. The method integrates DUSt3R for dense initialization and ROMA for confidence filtering to enhance the bundle adjustment process [S1].

## Sources
[S1] GBR: Generative Bundle Refinement for High-Fidelity Gaussian Splatting With Enhanced Mesh Reconstruction, https://doi.org/10.1109/tcsvt.2025.3596613, fetched 2026-10-29
