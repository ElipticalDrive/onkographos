# Distance-based Gaussian Splatting

Also called: Mesh-based Gaussian Splatting, Tightly/Loosely-bound Gaussian Splatting

## Overview
Distance-based Gaussian Splatting is a method for high-quality rendering that integrates a pre-existing large-scale mesh with 3D Gaussian splats by classifying the splats as either tightly-bound or loosely-bound based on their distance to the mesh surface [S1]. It solves the problem of redundant GPU memory usage and rendering artifacts caused by initializing Gaussian splats on every triangle of a dense mesh, which often contains geometric inaccuracies or excessive detail [S1]. Unlike standard 3D Gaussian Splatting which treats all splats equally, this method masks out splats occluded by the opaque mesh and applies distinct alignment constraints to tightly-bound splats to ensure they adhere to the surface while allowing loosely-bound splats to capture appearance details outside the immediate surface vicinity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh $M$ with faces $f_j$ and centers $m_j$; a set of training views with camera poses $P_i$; ground truth images.
*   **Output:** A set of 3D Gaussian splats $\{G_i\}$ with optimized parameters (position, covariance, opacity, color) and a rendered image.

**Data Structures:**
*   **Mesh $M$:** A collection of triangles. Each triangle $f_j$ has a center $m_j$ and associated geometry (normal, vertices).
*   **Gaussian Splat $G_i$:** Defined by:
    *   Center position $p_i \in \mathbb{R}^3$ [S1].
    *   3D covariance matrix $\Sigma_i \in \mathbb{R}^{3\times3}$ in world space [S1].
    *   Per-point opacity $o_i \in [0, 1]$ [S1].
    *   View-dependent color $c_i$ represented by spherical harmonics (SH) [S1].
    *   Classification flag: Tightly-bound ($G_{ti}$) or Loosely-bound ($G_{li}$) [S1].

**Gaussian Definitions:**
The 3D Gaussian is defined as [S1]:
$$ G_{i}(x) = e^{-\frac {1}{2}(x-p_{i})^T \Sigma _{i}^{-1}(x-p_{i})} $$
The covariance matrix is parameterized to ensure semi-definiteness [S1]:
$$ \Sigma_i = R_i S_i S_i^T R_i^T $$
where $S_i \in \mathbb{R}^3$ is a scaling vector and $R_i \in \mathbb{R}^{3\times3}$ is a rotation matrix encoded by a quaternion $q \in \mathbb{R}^4$ [S1].

**Rendering:**
Gaussians are projected into camera space ($p_{ci} = R_{cw} p_i + t_{cw}$, $\Sigma_{ic} = R_{cw} \Sigma_i R_{cw}^T$) and then into ray space using an affine transformation $J_i \in \mathbb{R}^{2\times3}$ [S1]:
$$ \Sigma'_i = J_i \Sigma_{ic} J_i^T $$
The 2D Gaussian value is [S1]:
$$ g_{i}(x) = e^{-\frac {1}{2}(x-p'_{i})^T {\Sigma '_{i}}^{-1}(x-p'_{i})} $$
The final color $c$ is computed via alpha blending [S1]:
$$ c = \sum _{i=1}^{n} c_{i}\alpha _{i}\prod _{j=1}^{i-1}(1-\alpha _{j}), \quad \alpha _{i} = o_{i}g_{i}(x) $$

**Algorithm Steps:**
1.  **Initialization:** Initialize a 3D Gaussian splat at the center $m_j$ of each triangle $f_j$ of the mesh $M$ [S1].
2.  **Occlusion Masking:** For each training viewpoint $P_i$, render a depth map $D_i$. Assume the mesh surface is entirely opaque. Mask out Gaussian splats that are occluded by the mesh surface (i.e., splats with depth greater than the mesh depth at that pixel) [S1].
3.  **Visibility Assessment:** Assess the visibility of all Gaussian splats across all training views. Identify splats located behind the mesh surface that are never observed by any training view [S1].
4.  **Pruning:** Remove all redundant Gaussian splats situated behind the mesh surface. This pruning occurs at specific training iterations, specifically every 500 iterations [S1].
5.  **Distance Calculation:** Calculate the distance $d_i$ between each Gaussian splat’s center $p_i$ and the corresponding mesh face $f_j$ [S1].
6.  **Classification:** Classify Gaussian splats based on the distance $d_i$ and a threshold $d_{th}$ [S1]:
    *   Tightly-bound splats ($G_{ti}$): if $d_i < d_{th}$.
    *   Loosely-bound splats ($G_{li}$): if $d_i > d_{th}$ [S1].
7.  **Alignment:** Flatten and align the tightly-bound splats ($G_{ti}$) with the mesh surface [S1]. This involves using a normal consistency loss to refine the orientation of these splats to match the mesh normals [S1].
8.  **Training:** Optimize the parameters of all splats using an image loss (L1 + D-SSIM) and specific losses for tightly-bound splats (normal consistency) [S1].

**Parameters:**
*   Distance threshold $d_{th}$: Set to 0.01 [S1].
*   Pruning interval: Every 500 training iterations [S1].
*   Depth offset: Add 0.01 to the rendered depth $D_i$ when applying occlusion masking [S1].
*   Loss weights: $\lambda_{img} = 0.001$, $\lambda_{nc} = 0.1$, $\lambda_{min} = 0.1$, $\lambda_{max} = 10$, $\lambda_{proj} = 50$ [S1].
*   Threshold $\rho$: Set to 0.1 [S1].
*   Training iterations: 30,000 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_based_gaussian_splatting(mesh M, training_views V, ground_truth_images I):
    # 1. Initialization
    splats = []
    for triangle f in M.triangles:
        center m = f.center
        # Initialize Gaussian parameters (position, covariance, opacity, color)
        # Covariance initialized based on triangle size/orientation
        G = create_gaussian(center=m, covariance=init_covariance(f))
        splats.append(G)

    # 2. Classification Setup
    d_th = 0.01
    prune_interval = 500

    # 3. Training Loop
    for iteration in range(30000):
        # Randomly select a training view
        view = random.choice(V)
        gt_image = get_gt_image(view, I)

        # 4. Occlusion Masking
        # Render depth map from current splat positions
        depth_map = render_depth(splats, view.camera)
        
        # Apply depth offset
        depth_map += 0.01
        
        # Mask out splats occluded by the mesh
        # Assume mesh is opaque. If splat depth > mesh depth, it is occluded.
        # Note: The mesh depth is derived from M. 
        # In practice, this involves checking if the splat is behind the mesh surface.
        visible_splats = []
        for G in splats:
            if not is_occluded_by_mesh(G, M, view.camera, depth_map):
                visible_splats.append(G)
        
        # 5. Visibility Assessment & Pruning
        if iteration % prune_interval == 0:
            # Identify splats behind mesh not observed by any view
            # This requires tracking visibility across all views
            # For simplicity, we prune based on current occlusion status if never seen
            # [G] Implementation detail: Maintain a 'seen' flag for each splat.
            # If a splat is occluded in all views and 'seen' is false, remove it.
            splats = prune_behind_mesh_splats(splats, M, V)

        # 6. Distance Calculation & Classification
        tightly_bound = []
        loosely_bound = []
        for G in splats:
            # Calculate distance from G.center to closest mesh face
            dist = distance_to_mesh(G.center, M)
            if dist < d_th:
                G.type = 'tightly_bound'
                tightly_bound.append(G)
            else:
                G.type = 'loosely_bound'
                loosely_bound.append(G)

        # 7. Rendering
        # Project Gaussians to 2D
        # Transform camera space to ray space
        # Compute 2D covariance Sigma' = J * Sigma_c * J^T
        # Compute 2D Gaussian values g_i(x)
        # Alpha blending: c = sum(c_i * alpha_i * prod(1 - alpha_j))
        rendered_image = render_alpha_blend(visible_splats, view.camera)

        # 8. Loss Calculation
        # Image Loss
        L_img = (1 - 0.001) * L1_loss(rendered_image, gt_image) + 0.001 * SSIM_loss(rendered_image, gt_image)
        
        # Normal Consistency Loss for tightly-bound splats
        L_nc = 0
        for G in tightly_bound:
            # Estimate normal of Gaussian from covariance
            G_normal = get_gaussian_normal(G)
            # Get mesh normal at closest point
            M_normal = get_mesh_normal(G.center, M)
            L_nc += normal_consistency_loss(G_normal, M_normal)
        L_nc *= 0.1 # lambda_nc

        # Total Loss
        total_loss = L_img + L_nc

        # 9. Optimization
        optimize_gaussians(splats, total_loss)

    return splats

def is_occluded_by_mesh(G, M, camera, depth_map):
    # Project G.center to 2D
    p_2d = project_to_screen(G.center, camera)
    # Get depth of mesh at p_2d from depth_map
    mesh_depth = depth_map[p_2d]
    # Get depth of G
    G_depth = distance_from_camera(G.center, camera)
    # If G is behind mesh, it is occluded
    return G_depth > mesh_depth
```

## Operating characteristics
*   **Time Complexity:** Rendering complexity is linear with respect to the number of visible Gaussian splats, similar to standard 3D Gaussian Splatting [S1]. However, the additional steps of mesh intersection checks for occlusion and distance calculations for classification add overhead during initialization and pruning [S1].
*   **Memory Complexity:** Significantly reduced compared to initializing splats on every triangle of a dense mesh. By pruning occluded splats and classifying them, the method avoids allocating redundant splats in distorted or occluded regions, leading to lower GPU memory usage [S1].
*   **Performance:** The method achieves high-fidelity rendering quality while using a minimal number of triangles in the underlying mesh (e.g., 10^3 to 10^6 units) [S1]. It handles severe mesh artifacts better than strictly tightly-bound methods by allowing loosely-bound splats to capture appearance details [S1].
*   **Strengths:** Reduces memory usage; handles mesh artifacts; aligns geometry with surface for tightly-bound splats.
*   **Limitations:** Relies on the quality of the input mesh; requires careful tuning of the distance threshold $d_{th}$ and pruning interval [S1].

## Background
Distance-based Gaussian Splatting is introduced in the paper "MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering" by Jaehoon et al. [S1]. It builds upon the original 3D Gaussian Splatting (3D-GS) method [S1, S22] and addresses issues related to large-scale mesh reconstruction where meshes may have excessive triangles or geometric inaccuracies [S1]. The method is designed to work with lightweight meshes (LTM) while maintaining high rendering quality [S1].

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
