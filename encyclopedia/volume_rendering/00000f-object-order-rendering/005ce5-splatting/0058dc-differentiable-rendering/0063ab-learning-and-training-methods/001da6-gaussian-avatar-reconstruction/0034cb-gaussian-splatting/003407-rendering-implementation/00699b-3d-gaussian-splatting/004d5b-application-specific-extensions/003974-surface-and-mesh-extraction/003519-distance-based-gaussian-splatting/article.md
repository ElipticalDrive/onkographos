# Distance-based Gaussian Splatting

Also called: Mesh-based Gaussian Splatting, Tightly/Loosely-bound Gaussian Splatting

## Overview
Distance-based Gaussian Splatting is a method for integrating 3D Gaussian Splatting (3DGS) with explicit mesh representations to improve rendering quality and memory efficiency in large-scale scenes [S1]. It solves the problem of redundant Gaussian splats occupying GPU memory and degrading rendering fidelity when initialized on high-resolution meshes by classifying splats based on their distance to the mesh surface [S1]. Unlike sibling methods that extract meshes from trained Gaussians, this approach builds a connection between a pre-existing mesh and Gaussian splats, masking occluded splats and distinguishing between tightly-bound splats (aligned with the surface) and loosely-bound splats (capturing artifacts or details) [S1].

## How it works
**Inputs:**
- A trained 3D Gaussian splatting model $\mathcal{G} = \{G_i\}_{i=1}^M$ [S1].
- A 3D mesh $M$ consisting of triangles $f_j$ and vertices $m_j$ [S1].
- Training views defined by camera poses $P_k$ [S1].

**Outputs:**
- A refined set of Gaussian splats $\mathcal{G}'$ classified as tightly-bound ($G_t$) or loosely-bound ($G_l$) [S1].
- Rendered images with improved geometric consistency and reduced memory usage [S1].

**Data Structures:**
- **3D Gaussian $G_i$**: Defined by center position $\mathbf{p}_i \in \mathbb{R}^3$, 3D covariance matrix $\mathbf{\Sigma}_i \in \mathbb{R}^{3\times3}$, opacity $o_i \in [0,1]$, and view-dependent color $\mathbf{c}_i$ (represented by spherical harmonics) [S1].
- **Covariance Parameterization**: $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$, where $\mathbf{S}_i \in \mathbb{R}^3$ is a scaling vector and $\mathbf{R}_i \in \mathbb{R}^{3\times3}$ is a rotation matrix encoded by a quaternion $\mathbf{q} \in \mathbb{R}^4$ [S1].
- **2D Gaussian $g_i$**: Defined by 2D center $\mathbf{p}'_i$ and 2D covariance $\mathbf{\Sigma}'_i$ in ray space [S1].
- **Distance Threshold**: $d_{th} = 0.01$ [S1].
- **Pruning Interval**: Every 500 training iterations [S1].

**Numbered Steps:**

1.  **Initialization**: Initialize a 3D Gaussian splat $G_i$ at the center $\mathbf{m}_j$ of each triangle $f_j$ of the mesh $M$ [S1].

2.  **Depth Rendering and Occlusion Masking**:
    - For each training viewpoint $P_k$, render a depth map $D_k$ [S1].
    - Add a small offset to the depth: $D_k \leftarrow D_k + 0.01$ [S1].
    - Assume the mesh surface is entirely opaque. Mask out Gaussian splats that are occluded by the mesh surface using the rendered depth map [S1]. Specifically, if a Gaussian splat's depth is greater than the mesh depth at that pixel, it is considered occluded and excluded from rendering [S1].

3.  **Visibility Assessment and Pruning**:
    - Assess the visibility of all Gaussian splats across all training views [S1].
    - Identify splats located behind the mesh surface that are never observed by any training view [S1].
    - Remove all redundant Gaussian splats situated behind the mesh surface at specific training iterations (every 500 iterations) [S1].

4.  **Distance Calculation and Classification**:
    - Calculate the distance $d_i$ between each Gaussian splat’s center $\mathbf{p}_i$ and the corresponding mesh face $f_j$ [S1].
    - Classify Gaussian splats into two categories based on the distance threshold $d_{th} = 0.01$ [S1]:
        - **Tightly-bound splats ($G_t$)**: If $d_i < d_{th}$ [S1].
        - **Loosely-bound splats ($G_l$)**: If $d_i > d_{th}$ [S1].

5.  **Alignment and Rendering**:
    - Flatten and align the tightly-bound splats $G_t$ with the mesh surface [S1].
    - For rendering, transform camera space into ray space using the affine transformation $\mathbf{J}_i \in \mathbb{R}^{2\times3}$ defined by the camera position $\mathbf{p}_{ci}$ [S1].
    - Compute the 2D covariance in ray space: $\mathbf{\Sigma}'_i = \mathbf{J}_i \mathbf{\Sigma}_{ic} \mathbf{J}_i^T$ [S1].
    - Render the image using alpha blending:
      $$ c = \sum_{i=1}^{n} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1-\alpha_j) $$
      where $\alpha_i = o_i g_i(\mathbf{x})$, $g_i(\mathbf{x}) = e^{-\frac{1}{2}(\mathbf{x}-\mathbf{p}'_i)^T {\mathbf{\Sigma}'_i}^{-1}(\mathbf{x}-\mathbf{p}'_i)}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_based_gaussian_splatting(mesh, gaussians, training_views, d_th=0.01, prune_interval=500):
    """
    Implements Distance-based Gaussian Splatting.
    
    Args:
        mesh: 3D mesh object with triangles f_j and vertices m_j.
        gaussians: List of 3D Gaussian splats G_i.
        training_views: List of camera poses P_k.
        d_th: Distance threshold for classification (default 0.01).
        prune_interval: Iteration interval for pruning (default 500).
        
    Returns:
        refined_gaussians: List of refined Gaussian splats.
    """
    # Step 1: Initialization
    # Initialize a 3D Gaussian splat at the center m_j of each triangle f_j of the mesh M.
    # Note: This assumes gaussians are initialized based on mesh triangles.
    # In practice, this step happens before training starts.
    for triangle in mesh.triangles:
        center = triangle.center
        # Create Gaussian G_i at center with initial covariance and opacity
        gaussians.append(create_gaussian(center, mesh))

    # Step 2: Depth Rendering and Occlusion Masking
    for view in training_views:
        # Render depth map D_k from viewpoint P_k
        depth_map = render_depth(view, mesh)
        
        # Add offset to depth
        depth_map = depth_map + 0.01
        
        # Mask out Gaussian splats occluded by the surface
        for gaussian in gaussians:
            if is_occluded(gaussian, depth_map, view):
                gaussian.occluded = True
            else:
                gaussian.occluded = False

    # Step 3: Visibility Assessment and Pruning
    current_iteration = 0
    while current_iteration < max_iterations:
        # Assess visibility across all training views
        visible_splats = []
        for gaussian in gaussians:
            is_visible = False
            for view in training_views:
                if not gaussian.occluded and is_observed(gaussian, view):
                    is_visible = True
                    break
            if is_visible:
                visible_splats.append(gaussian)
            else:
                # Identify splats behind mesh never observed
                pass
        
        # Prune redundant splats every 500 iterations
        if current_iteration % prune_interval == 0:
            gaussians = [g for g in gaussians if g in visible_splats]
        
        current_iteration += 1

    # Step 4: Distance Calculation and Classification
    tightly_bound = []
    loosely_bound = []
    
    for gaussian in gaussians:
        # Calculate distance d_i between Gaussian center p_i and mesh face f_j
        distance = calculate_distance_to_mesh(gaussian.center, mesh)
        
        if distance < d_th:
            gaussian.type = "tightly_bound"
            tightly_bound.append(gaussian)
        else:
            gaussian.type = "loosely_bound"
            loosely_bound.append(gaussian)
            
    # Step 5: Alignment and Rendering
    # Flatten and align tightly-bound splats with mesh surface
    for gaussian in tightly_bound:
        align_with_mesh_surface(gaussian, mesh)
        
    # Rendering is performed using the standard 3DGS pipeline with the refined splats
    return gaussians

def is_occluded(gaussian, depth_map, view):
    """
    Check if a Gaussian is occluded by the mesh surface.
    """
    # Project Gaussian center to 2D
    p_2d = project_to_2d(gaussian.center, view)
    
    # Get depth of Gaussian
    depth_gaussian = gaussian.center.z # In camera space
    
    # Get depth of mesh at p_2d
    depth_mesh = depth_map[p_2d.y][p_2d.x]
    
    # If Gaussian depth is greater than mesh depth, it is occluded
    return depth_gaussian > depth_mesh

def calculate_distance_to_mesh(point, mesh):
    """
    Calculate the minimum distance from a point to the mesh surface.
    """
    min_dist = infinity
    for triangle in mesh.triangles:
        dist = point_to_triangle_distance(point, triangle)
        if dist < min_dist:
            min_dist = dist
    return min_dist
```

**Parameters:**
- $d_{th} = 0.01$: Distance threshold to distinguish tightly-bound and loosely-bound splats [S1].
- Pruning interval: 500 training iterations [S1].
- Depth offset: 0.01 added to depth map [S1].
- Loss weights: $\lambda_{img} = 0.001$, $\lambda_{nc} = 0.1$, $\lambda_{min} = 0.1$, $\lambda_{max} = 10$, $\lambda_{proj} = 50$ [S1].
- Threshold $\rho = 0.1$ [S1].
- Training iterations: 30,000 [S1].

**Edge Cases and Pitfalls:**
- **Mesh Artifacts**: Tightly binding all splats exclusively to the mesh surface can suffer rendering performance issues when addressing severe mesh artifacts [S1]. The loosely-bound splats help capture these inaccuracies [S1].
- **Redundant Splats**: Without pruning, initializing splats on every triangle of a high-resolution mesh (e.g., 44 million triangles) leads to a rapid increase in GPU memory usage and undermines rendering quality [S1].
- **Occlusion Assumption**: The method assumes the mesh surface is entirely opaque [S1].

## Operating characteristics
- **Memory Efficiency**: Significantly reduces GPU memory usage compared to initializing splats on high-resolution meshes by pruning occluded and redundant splats [S1].
- **Rendering Quality**: Improves rendering quality by aligning tightly-bound splats with the mesh surface while using loosely-bound splats to capture details and artifacts [S1].
- **Scalability**: Suitable for large-scale scenes where meshes have many triangles (e.g., $10^6$ units) [S1].
- **Limitations**: Performance may suffer if the mesh has severe artifacts that are not well-captured by loosely-bound splats [S1].

## Background
- **Origin**: Proposed in "MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering" by Jaehoon et al. [S1].
- **History**: Builds on the original 3D Gaussian Splatting (3DGS) method [22] and addresses limitations in methods like LTM and SuGaR which either use minimal triangles (lacking detail) or require significant vertex counts [S1].
- **Usage**: Used for high-quality rendering of large-scale scenes with explicit mesh representations [S1].

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
