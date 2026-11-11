# Distance-based Gaussian Splatting

Also called: Mesh-based Gaussian Splatting, Tightly/Loosely-bound Gaussian Splatting

## Overview
Distance-based Gaussian Splatting is a method that integrates a pre-existing 3D mesh with 3D Gaussian splats to improve rendering quality and geometric consistency, particularly for large-scale scenes [S1]. It solves the problem of redundant or distorted Gaussian splats in regions where the underlying mesh geometry is inaccurate or overly dense by classifying splats as either "tightly-bound" or "loosely-bound" based on their distance to the mesh surface [S1]. Unlike standard 3D Gaussian Splatting which relies solely on view-dependent densification, this method explicitly aligns a subset of splats to the mesh surface to correct geometric artifacts while allowing others to capture fine details [S1].

## How it works
**Inputs:**
*   A 3D mesh $M$ composed of triangles $f_j$ with centers $m_j$ [S1].
*   A set of training images with known camera poses [S1].
*   Standard 3D Gaussian Splatting parameters (opacity, spherical harmonics coefficients) [S1].

**Outputs:**
*   A set of 3D Gaussian splats $\{G_i\}$, classified into tightly-bound ($G^t_i$) and loosely-bound ($G^l_i$) subsets [S1].
*   A rendered image $\mathbf{c}$ [S1].

**Data Structures:**
*   **Gaussian Splat $i$:**
    *   Center: $\mathbf{p}_i \in \mathbb{R}^3$ [S1].
    *   Covariance: $\mathbf{\Sigma}_i \in \mathbb{R}^{3\times3}$, parameterized as $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$, where $\mathbf{S}_i \in \mathbb{R}^3$ is a scaling vector and $\mathbf{R}_i \in \mathbb{R}^{3\times3}$ is a rotation matrix encoded by a quaternion $\mathbf{q} \in \mathbb{R}^4$ [S1].
    *   Opacity: $o_i \in [0, 1]$ [S1].
    *   Color: View-dependent color $\mathbf{c}_i$ represented by spherical harmonics [S1].
*   **Mesh $M$:** List of triangles $f_j$ with associated surface normals and centers $m_j$ [S1].

**Parameters:**
*   Distance threshold $d_{th} = 0.01$ [S1].
*   Pruning interval: Every 500 training iterations [S1].
*   Depth offset: $+0.01$ added to rendered depth $D_i$ [S1].
*   Loss weights: $\lambda_{img} = 0.001$, $\lambda_{nc} = 0.1$, $\lambda_{min} = 0.1$, $\lambda_{max} = 10$, $\lambda_{proj} = 50$ [S1].
*   Threshold $\rho = 0.1$ [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize a 3D Gaussian splat $G_i$ at the center $\mathbf{m}_j$ of each triangle $f_j$ of the mesh $M$ [S1]. The initial covariance and orientation are derived from the mesh geometry [S1].

2.  **Occlusion Masking:** For each training viewpoint $\mathbf{P}_i$, render a depth map $D_i$ from the mesh $M$. Add $0.01$ to $D_i$ to ensure proper depth testing [S1]. Use this depth map to mask out Gaussian splats that are occluded by the mesh surface, assuming the mesh is entirely opaque [S1]. Splats located behind the mesh surface relative to the camera are excluded from rendering [S1].

3.  **Visibility Assessment and Pruning:** Assess the visibility of all Gaussian splats across all training views. Identify splats located behind the mesh surface that are never observed by any training view [S1]. Remove these redundant splats at every 500 training iterations [S1].

4.  **Distance Calculation:** Calculate the Euclidean distance $d_i$ between the center $\mathbf{p}_i$ of each Gaussian splat and the corresponding mesh face $f_j$ [S1].

5.  **Classification:** Classify each Gaussian splat based on the distance $d_i$ and threshold $d_{th}$:
    *   If $d_i < d_{th}$, classify as **tightly-bound** ($G^t_i$) [S1].
    *   If $d_i > d_{th}$, classify as **loosely-bound** ($G^l_i$) [S1] [S1].

6.  **Alignment:** Flatten the tightly-bound splats $G^t_i$ and align them with the mesh surface [S1]. This involves constraining their covariance and orientation to match the local surface geometry [S1].

7.  **Rendering:** Render the image using the standard differentiable rasterization pipeline [S1]:
    *   Transform camera space into ray space using an affine transformation $\mathbf{J}_i \in \mathbb{R}^{2\times3}$ defined by the camera position $\mathbf{p}_{ci}$ [S1].
    *   Compute the 2D covariance in ray space: $\mathbf{\Sigma}'_i = \mathbf{J}_i \mathbf{\Sigma}_{ic} \mathbf{J}_i^T$, where $\mathbf{\Sigma}_{ic}$ is the 3D covariance in camera space [S1].
    *   Compute the 2D Gaussian value: $g_i(\mathbf{x}) = e^{-\frac{1}{2}(\mathbf{x}-\mathbf{p}'_i)^T {\mathbf{\Sigma}'_i}^{-1}(\mathbf{x}-\mathbf{p}'_i)}$ [S1].
    *   Render the image using alpha blending: $\mathbf{c} = \sum_{i=1}^{n} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1}(1-\alpha_j)$, where $\alpha_i = o_i g_i(\mathbf{x})$ [S1].

8.  **Optimization:** Minimize the total loss function, which includes image loss, normal consistency loss for tightly-bound splats, and other regularization terms [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_based_gaussian_splatting(mesh M, training_views V, iterations=30000):
    # Initialization
    splats = []
    for triangle f_j in M.triangles:
        center m_j = f_j.center
        # Initialize Gaussian at mesh triangle center
        G = Gaussian(center=m_j, covariance=compute_covariance_from_mesh(f_j), 
                     opacity=1.0, color=compute_color_from_mesh(f_j))
        splats.append(G)
    
    d_th = 0.01
    prune_interval = 500
    
    for iter in range(iterations):
        # Occlusion Masking
        for view in V:
            # Render depth from mesh, adding 0.01 offset
            depth_map = render_mesh_depth(M, view.camera) + 0.01
            
            # Mask out splats occluded by mesh
            for G in splats:
                if is_occluded_by_mesh(G.center, view.camera, depth_map):
                    G.visible = False
                else:
                    G.visible = True
        
        # Visibility Assessment and Pruning
        if iter % prune_interval == 0:
            # Identify splats behind mesh and never observed
            redundant_splats = []
            for G in splats:
                if not G.visible and is_behind_mesh(G.center, M):
                    redundant_splats.append(G)
            # Remove redundant splats
            splats = [G for G in splats if G not in redundant_splats]
            
        # Distance Calculation and Classification
        tightly_bound = []
        loosely_bound = []
        for G in splats:
            # Calculate distance to corresponding mesh face
            d_i = distance_to_mesh_face(G.center, M)
            if d_i < d_th:
                G.type = 'tightly_bound'
                tightly_bound.append(G)
            else:
                G.type = 'loosely_bound'
                loosely_bound.append(G)
                
        # Alignment for tightly-bound splats
        for G in tightly_bound:
            align_to_mesh_surface(G, M)
            
        # Rendering and Loss Computation
        for view in V:
            # Standard 3DGS Rendering Pipeline
            # 1. Transform to camera space
            for G in splats:
                if G.visible:
                    G.camera_center = view.camera.transform(G.center)
                    G.camera_cov = view.camera.R @ G.covariance @ view.camera.R.T
            
            # 2. Project to 2D ray space
            for G in splats:
                if G.visible:
                    J_i = view.camera.affine_transform(G.camera_center)
                    G.sigma_2d = J_i @ G.camera_cov @ J_i.T
                    G.g_2d = compute_2d_gaussian(G.sigma_2d, G.camera_center)
            
            # 3. Alpha Blending
            rendered_image = alpha_blend(splats, view.camera)
            
            # Compute Loss
            L_img = compute_image_loss(rendered_image, view.gt_image)
            L_nc = compute_normal_consistency_loss(tightly_bound, M)
            total_loss = (1 - 0.001) * L_img + 0.001 * SSIM(rendered_image, view.gt_image) + 0.1 * L_nc
            # Add other regularization losses (lambda_min, lambda_max, lambda_proj)
            
            # Backpropagation
            optimize(splats, total_loss)
```

**Edge Cases and Pitfalls:**
*   **Mesh Artifacts:** If the mesh has severe geometric artifacts, tightly binding all splats can degrade rendering performance. The loose-bound classification allows splats to deviate from the mesh to capture these inaccuracies or fine details [S1].
*   **Depth Precision:** The addition of $0.01$ to the depth map is critical to prevent self-occlusion errors when masking splats against the mesh [S1].
*   **Pruning Timing:** Pruning must occur at regular intervals (every 500 iterations) to prevent memory bloat from redundant splats behind the mesh [S1].

## Operating characteristics
*   **Time Complexity:** Rendering complexity is dominated by the number of Gaussian splats $n$, similar to standard 3D Gaussian Splatting $O(n)$ per pixel [S1]. The additional cost comes from mesh intersection tests for occlusion masking and distance calculations, which are performed per splat per view [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of splats. By pruning redundant splats behind the mesh, this method can reduce memory usage compared to a naive initialization of one splat per triangle if many triangles are occluded [S1]. However, if the mesh is very dense (e.g., 44 million triangles), initializing a splat per triangle can still lead to high memory usage before pruning [S1].
*   **Performance:** The method improves rendering quality in regions with mesh artifacts by allowing loosely-bound splats to compensate [S1]. It is particularly effective for large-scale scenes where mesh quality varies [S1].
*   **Strengths:** Explicit geometric alignment improves normal consistency and reduces artifacts in tightly-bound regions. Pruning reduces redundancy.
*   **Limitations:** Relies on the quality of the input mesh. If the mesh is poor, the classification and alignment may not be beneficial. The method requires additional computation for mesh-splat distance calculations and occlusion testing.

## Background
Distance-based Gaussian Splatting was introduced in the paper "MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering" by Jaehoon et al. [S1]. The method addresses the limitations of existing approaches like LTM (Lightweight Triangular Mesh) and SuGaR, which either use minimal triangles (leading to loss of detail) or impose strict vertex constraints [S1]. The authors note that standard 3D Gaussian Splatting can require millions of splats to achieve high fidelity, which is inefficient when a mesh representation is available [S1]. The method was evaluated on the Mip-NeRF360 dataset, using 30,000 training iterations [S1].

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
