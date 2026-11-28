# Stage II: Align 3DGS with Radiance Field

Also called: Field-wise Feature Error Minimization, Radial Feature Optimization

## Overview
This method is the second stage of the iVR-GS training pipeline, designed to minimize the discrepancy between the rendering of deformed 3D Gaussian Splatting (3DGS) and the rendering of the deformed radiance field defined by radial features [S1]. While Stage I establishes the initial deformation, Stage II finetunes the Gaussian configuration (position, rotation, scaling, spherical harmonics, and opacity) to ensure the deformed Gaussians faithfully capture the field-wise deformation [S1]. It distinguishes itself from standard 3DGS optimization by explicitly aligning the Gaussian splats with a ground-truth radiance field derived from an embedded deformation graph, rather than just matching pixel colors [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $n$ Gaussians $\{G_i\}$ with parameters $\{p_i, r_i, s_i, H_i, \alpha_i\}$ at rest shape.
*   A deformation graph defined by control points (selected via Furthest Point Sampling from Gaussian centers) and KNN edge connections [S1].
*   A rest-shape radiance field representation.
*   Hyperparameters: $\lambda_{lpf}$ (low-pass filter bandwidth), $n_{dim}$ (sample points per axis), $\lambda_1, \lambda_2$ (error weights) [S1].

**Outputs:**
*   Optimized Gaussian parameters $\{p_i, r_i, s_i, H_i, \alpha_i\}$ that minimize the total error $E_{total}$ [S1].

### Data Structures
*   **Gaussian $i$**:
    *   $p_i \in \mathbb{R}^3$: Position.
    *   $r_i \in \mathbb{R}^4$: Rotation (quaternion).
    *   $s_i \in \mathbb{R}^3$: Scaling.
    *   $H_i$: Spherical Harmonics coefficients.
    *   $\alpha_i$: Opacity (raw scalar).
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, computed from $r_i, s_i$ [S1].
*   **Sample Points**:
    *   Set $\{x_j\}$ uniformly distributed within the Axis-Aligned Bounding Boxes (AABB) of Gaussians at rest shape [S1].
    *   Adjacency list mapping each sample point to its nearest control points on the deformation graph (computed via KNN) [S1].
*   **Spatial Hash**:
    *   Used to map Gaussians to grid cells for fast updates of spatial relations between Gaussians and sample points during optimization [S1].

### Formulas
The radiance field is characterized by two radial features: Feature $F$ (encoding rotated harmonics color) and Opacity $O$ (indicating transparency) [S1]:

$$
F(p) = \sum_{i=1}^n \alpha_i G_i(p - p_i) \cdot H_i \quad (7)
$$

$$
O(p) = \sum_{i=1}^n \alpha_i G_i(p - p_i) \quad (8)
$$

where $G_i(x)$ is the Gaussian kernel:

$$
G_i(x) = \exp\left(-\frac{1}{2} x^\top \Sigma_i^{-1} x\right) \quad (8)
$$

The ground truth radial features $F^*$ and $O^*$ are computed at sample points $x_j$ after applying embedded deformation. The deformation of a point $v_i$ to $\tilde{v}_i$ is given by [S1]:

$$
\tilde{v}_i = \sum_j w_j(v_i) \left( A_j (v_i - g_j) + g_j \right) + t_j \quad (5)
$$

where $w_j(v_i)$ is the weight from the $j$-th control node, $g_j$ is the control point position, $A_j$ is the affine transformation, and $t_j$ is translation [S1]. The deformation gradient tensor is used to compute $F^*$ and $O^*$ at the new positions [S1].

The total error to minimize is [S1]:

$$
E_{total} = E_{feature} + \lambda_1 E_{opacity} + \lambda_2 E_{boundary} \quad (14)
$$

Where:
*   $E_{feature} \approx \sum_{j=1}^m |F^*(x_j) - F(x_j)|$ (9, 11)
*   $E_{opacity} \approx \sum_{j=1}^m |O^*(x_j) - O(x_j)|$ (10, 12)
*   $E_{boundary} = \sum_{x_j \in \mathcal{O}} |O^*(x_j) - O(x_j)|$ (13)
*   $\mathcal{O} = \{ \forall x_j : O^*(x_j) = 0 \}$ denotes sample points in unoccupied space [S1].

The adaptive anisotropic low-pass filter adjusts the covariance matrix $\Sigma_i$ to handle aliasing during deformation [S1]:

$$
\Sigma_i = R_i S_i S_i^\top R_i^\top + \frac{\lambda_{lpf}}{n_{dim}^2} A_k A_k^\top \quad (15)
$$

where $A_k$ is a $3 \times 3$ matrix estimating the first-order approximation of the deformation field near the grid cell center $k$, and $n_{dim}$ is the number of sample points per axis [S1].

### Parameters
*   $\lambda_{lpf}$: Controls the passing bandwidth of the low-pass filter. Typical value: 0.2 [S1].
*   $n_{dim}$: Number of sample points per axis in each grid cell [S1].
*   $\lambda_1, \lambda_2$: Weights for opacity and boundary errors. Not given in the sources.
*   $w_{rot}, w_{reg}, w_{con}$: Weights for the deformation graph optimization (Stage I). Values: 1, 10, 100 respectively [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stage_2_align_3dgs_with_radiance_field(gaussians, deformation_graph, hyperparams):
    """
    Finetunes Gaussian configuration to align with deformed radiance field.
    
    Args:
        gaussians: List of Gaussian objects with attributes p, r, s, H, alpha.
        deformation_graph: Object containing control points and KNN adjacency.
        hyperparams: Dict with keys lambda_lpf, n_dim, lambda_1, lambda_2.
    
    Returns:
        Optimized gaussians.
    """
    # 1. Initialization and Sampling
    # Extract AABB for each Gaussian at rest shape
    aabbs = [extract_aabb(g) for g in gaussians]
    
    # Uniformly place sample points within bounding boxes
    sample_points = []
    for aabb in aabbs:
        points = uniform_sample_in_aabb(aabb, count_per_axis=hyperparams['n_dim'])
        sample_points.extend(points)
        
    # Construct adjacency to control points on deformation graph for sample points using KNN
    # Note: KNN is performed against the control points of the deformation graph
    sample_point_adjacency = knn(sample_points, deformation_graph.control_points, k=4) # k is typical for deformation weights
    
    # Use spatial hashing to fast update spatial relations
    spatial_hash = SpatialHash(gaussians, cell_size=hyperparams['n_dim'])
    
    # Pre-compute harmonic features and opacity features at sample points for rest-shape
    # This establishes the baseline O* and F* before deformation is applied to the field
    # However, the method describes computing F* and O* AFTER deformation
    # So we store rest-shape samples for reference if needed, but primary targets are deformed
    
    # 2. Optimization Loop
    optimizer = Adam(gaussians.parameters(), lr=0.001) # Standard 3DGS learning rate
    
    for iteration in range(max_iterations):
        optimizer.zero_grad()
        
        # A. Compute Ground Truth Radial Features (F*, O*)
        # Apply embedded deformation to sample points to get new positions in the warped field
        deformed_sample_points = []
        for x_j in sample_points:
            # Find influence weights w_j from control points
            weights = compute_deformation_weights(x_j, sample_point_adjacency)
            # Compute deformed position using Eq. 5
            deformed_pos = compute_embedded_deformation(x_j, weights, deformation_graph)
            deformed_sample_points.append(deformed_pos)
            
        # Compute ground truth radial features F* and O* at deformed sample points
        # This requires the deformation gradient tensor at sample points
        # F* and O* are derived from the deformed radiance field definition
        F_star, O_star = compute_ground_truth_radial_features(deformed_sample_points, deformation_graph)
        
        # B. Compute Deformed Gaussian Radial Features (F, O)
        # Update Gaussian positions based on current optimization variables
        # Note: In this stage, we optimize the Gaussians themselves to match the field
        # The Gaussians are deformed according to their current parameters
        deformed_gaussians = apply_deformation_to_gaussians(gaussians, deformation_graph)
        
        # Compute radial features F and O from deformed Gaussians using standard equations (7, 8)
        F_gauss, O_gauss = compute_gaussian_radial_features(deformed_gaussians, deformed_sample_points)
        
        # C. Compute Errors
        # E_feature: Difference in harmonic features
        E_feature = sum(abs(F_star[j] - F_gauss[j]) for j in range(len(sample_points)))
        
        # E_opacity: Difference in opacity
        E_opacity = sum(abs(O_star[j] - O_gauss[j]) for j in range(len(sample_points)))
        
        # E_boundary: Restrict optimization to unoccupied space sample points
        # O = {∀x_j : O*(x_j) = 0} denotes sample points in unoccupied space
        unoccupied_indices = [j for j, o_star in enumerate(O_star) if o_star == 0]
        E_boundary = sum(abs(O_star[j] - O_gauss[j]) for j in unoccupied_indices)
        
        # Total Error
        E_total = E_feature + hyperparams['lambda_1'] * E_opacity + hyperparams['lambda_2'] * E_boundary
        
        # D. Apply Adaptive Anisotropic Low-Pass Filter
        # Estimate post-deformation sampling intervals by calculating 3x3 matrix A_k
        # A_k offers first-order approximation of deformation field near grid cell center
        for g in gaussians:
            grid_cell = spatial_hash.get_cell(g.p)
            A_k = compute_deformation_gradient_matrix(grid_cell, deformation_graph)
            n_dim = hyperparams['n_dim']
            lambda_lpf = hyperparams['lambda_lpf']
            
            # Compute covariance with low-pass filter
            R = quaternion_to_rotation_matrix(g.r)
            S = scale_matrix(g.s)
            Sigma_base = R @ S @ S.T @ R.T
            
            # Add low-pass filter term
            Sigma_filtered = Sigma_base + (lambda_lpf / (n_dim ** 2)) * (A_k @ A_k.T)
            
            # Update Gaussian covariance
            g.Sigma = Sigma_filtered
            
        # E. Backpropagation and Update
        E_total.backward()
        
        # Apply sigmoid activation to opacity and exponential activation to scaling for stability
        for g in gaussians:
            g.alpha = sigmoid(g.alpha_raw)
            g.s = exp(g.s_raw)
            
        optimizer.step()
        
    return gaussians
```

### Edge Cases and Pitfalls
*   **Boundary Deformation**: The deformation graph is agnostic to model boundaries, potentially deforming unoccupied space. This is handled by the $E_{boundary}$ term, which restricts optimization influence to sample points where $O^*(x_j) = 0$ [S1].
*   **Aliasing**: Undersampling high-frequency Gaussians during deformation causes aliasing. This is mitigated by the adaptive anisotropic low-pass filter, which adjusts the Gaussian size based on the deformation gradient [S1].
*   **Filter Sensitivity**: An over-conservative $\lambda_{lpf}$ (e.g., 0.02) leads to under-sampling, while an over-aggressive value (e.g., 2.0) causes blurriness. A value of 0.2 is empirically found to be robust [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time**: The method requires computing KNN for sample points and updating spatial hashes at each optimization iteration. The computation of radial features $F$ and $O$ scales with the number of Gaussians $n$ and sample points $m$. The use of spatial hashing accelerates the update of spatial relations [S1].
*   **Memory**: Requires storing sample points, their adjacency to control points, and the spatial hash structure. The number of sample points is proportional to the number of Gaussians [S1].

### Measured Performance
*   Implemented on a single NVIDIA RTX 4090 24GB GPU using C++ and CUDA [S1].
*   The method is validated on synthetic and real-world data (e.g., Stripes, Glove, Pinocchio, Bunny, Shoe, Cup) [S1].
*   The results are not sensitive to $\lambda_{lpf}$ within a reasonable range [S1].

### Strengths and Limitations
*   **Strengths**: Faithfully captures field-wise deformation; handles aliasing adaptively; prevents deformation of unoccupied space [S1].
*   **Limitations**: Requires pre-computation of features and KNN adjacency; adds computational overhead to the training process [S1].

## Background

### Origin
The method originates from the paper "As-Rigid-As-Possible Deformation of Gaussian Radiance Fields" [S1]. It was developed to address the issue where rasterizing deformed 3DGS does not always align with the rendering obtained from the deformed radiance field [S1].

### History and Development
The method builds upon previous work in 3DGS [9], EWA-Splatting [42], and Mip-Splatting [20], adapting their filtering strategies to the dynamic deformation context [S1]. It utilizes embedded deformation techniques similar to those in [16] and [36] [S1].

## Sources

[S1] As-Rigid-As-Possible Deformation of Gaussian Radiance Fields, https://doi.org/10.1109/TVCG.2025.3555404, fetched 2026-10-30
