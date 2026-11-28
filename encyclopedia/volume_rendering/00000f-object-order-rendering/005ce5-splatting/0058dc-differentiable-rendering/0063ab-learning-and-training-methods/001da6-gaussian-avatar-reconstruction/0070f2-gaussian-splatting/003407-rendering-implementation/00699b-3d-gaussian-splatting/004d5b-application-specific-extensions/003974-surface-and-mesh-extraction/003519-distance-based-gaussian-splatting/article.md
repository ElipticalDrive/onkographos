# Distance-based Gaussian Splatting

Also called: Mesh-based Gaussian Splatting, Tightly/Loosely-bound Gaussian Splatting

## Overview
Distance-based Gaussian Splatting integrates 3D Gaussian Splatting with explicit mesh representations to improve rendering quality and memory efficiency in large-scale scenes [S1]. It solves the problem of redundant Gaussian splats occupying GPU memory and degrading rendering fidelity when initialized on high-resolution meshes by classifying splats based on their distance to the mesh surface [S1]. Unlike sibling methods that extract meshes from trained Gaussians, this approach builds a connection between a pre-existing mesh and Gaussian splats, masking occluded splats and distinguishing between tightly-bound splats (aligned with the surface) and loosely-bound splats (capturing artifacts or details) [S1].

## How it works
**Inputs:**
- A trained 3D Gaussian splatting model $\mathcal{G} = \{G_i\}_{i=1}^M$ [S1].
- A 3D mesh $M$ consisting of triangles $f_j$ and centers $m_j$ [S1].
- Training viewpoints $P_i$ [S1].

**Outputs:**
- A refined set of Gaussian splats $\mathcal{G}'$ classified as tightly-bound ($G_t$) or loosely-bound ($G_l$) [S1].

**Data Structures:**
- **3D Gaussian $G_i$**: Defined by center position $\mathbf{p}_i \in \mathbb{R}^3$, 3D covariance matrix $\mathbf{\Sigma}_i \in \mathbb{R}^{3\times3}$, opacity $o_i \in [0,1]$, and view-dependent color $\mathbf{c}_i$ (represented by spherical harmonics) [S1].
- **Covariance Parameterization**: $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$, where $\mathbf{S}_i \in \mathbb{R}^3$ is a scaling vector and $\mathbf{R}_i \in \mathbb{R}^{3\times3}$ is a rotation matrix encoded by a quaternion $\mathbf{q} \in \mathbb{R}^4$ [S1].
- **2D Gaussian $g_i$**: Defined by 2D center $\mathbf{p}'_i$ and 2D covariance $\mathbf{\Sigma}'_i$ in ray space [S1].
- **Distance Threshold**: $d_{th} = 0.01$ [S1].
- **Pruning Interval**: Every 500 training iterations [S1].

**Numbered Steps:**

1.  **Initialization**: Initialize a 3D Gaussian splat at the center $\mathbf{m}_j$ of each triangle $f_j$ of the mesh $M$ [S1].

2.  **Depth Rendering and Occlusion Masking**:
    - Render a depth $D_i$ from a given viewpoint $P_i$ [S1].
    - Add 0.01 to the depth $D_i$ [S1].
    - Assume the mesh surface is entirely opaque. Mask out Gaussian splats occluded by the surface using the corresponding viewpoint depth map [S1]. Specifically, if a Gaussian splat's depth is greater than the mesh depth at that pixel, it is considered occluded and excluded from rendering [S1].

3.  **Visibility Assessment and Pruning**:
    - Assess the visibility of all Gaussian splats across all training views [S1].
    - Identify splats located behind the mesh surface that are never observed by any training view [S1].
    - Remove all redundant Gaussian splats situated behind the mesh surface at specific training iterations (every 500 iterations) [S1].

4.  **Distance Calculation and Classification**:
    - Calculate the distance $d_i$ between each Gaussian splat’s center $\mathbf{p}_i$ and the corresponding mesh face $f_j$ [S1].
    - Classify Gaussian splats into two categories based on the distance threshold $d_{th} = 0.01$ [S1]:
        - **Tightly-bound splats ($G_{ti}$)**: If $d_i < d_{th}$ [S1].
        - **Loosely-bound splats ($G_{li}$)**: If $d_i > d_{th}$ [S1].

5.  **Alignment and Rendering**:
    - Flatten $G_{ti}$ and align these splats with the mesh surface [S1].
    - Transform camera space into ray space using the affine transformation $\mathbf{J}_i \in \mathbb{R}^{2\times3}$ defined by the camera position $\mathbf{p}_{ci}$ [S1].
    - Compute the 2D covariance in ray space: $\mathbf{\Sigma}'_i = \mathbf{J}_i \mathbf{\Sigma}_{ic} \mathbf{J}_i^T$ [S1].
    - Render the image using alpha blending:
      $$ c = \sum_{i=1}^{n} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1-\alpha_j) $$
      where $\alpha_i = o_i g_i(\mathbf{x})$, $g_i(\mathbf{x}) = e^{-\frac{1}{2}(\mathbf{x}-\mathbf{p}'_i)^T {\mathbf{\Sigma}'_i}^{-1}(\mathbf{x}-\mathbf{p}'_i)}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_based_gaussian_splatting(mesh, gaussians, training_views, d_th=0.01, prune_interval=500, max_iterations=30000):
    # Step 1: Initialization
    # Initialize a 3D Gaussian splat at the center m_j of each triangle f_j of the mesh M.
    for triangle in mesh.triangles:
        gaussians.append(Gaussian(center=triangle.center, covariance=compute_triangle_covariance(triangle), opacity=1.0))

    # Step 2: Depth Rendering and Occlusion Masking
    for view in training_views:
        # Render a depth D_i from a given viewpoint P_i
        depth_map = rasterize_mesh_depth(view, mesh)
        # Add 0.01 to the depth D_i
        depth_map = depth_map + 0.01
        # Mask out Gaussian splats occluded by the surface
        for g in gaussians:
            p_2d = project_to_2d(g.center, view)
            if depth_map[p_2d.y, p_2d.x] < g.depth_in_camera_space:
                g.occluded = True
            else:
                g.occluded = False

    # Step 3: Visibility Assessment and Pruning
    current_iteration = 0
    while current_iteration < max_iterations:
        visible_splats = []
        for g in gaussians:
            is_observed = False
            for view in training_views:
                if not g.occluded and is_within_image_bounds(g.center, view):
                    is_observed = True
                    break
            if is_observed:
                visible_splats.append(g)
        # Remove all redundant Gaussian splats situated behind the mesh surface at specific training iterations
        if current_iteration % prune_interval == 0:
            gaussians = visible_splats
        current_iteration += 1

    # Step 4: Distance Calculation and Classification
    tightly_bound = []
    loosely_bound = []
    for g in gaussians:
        # Calculate the distance d_i between each Gaussian splat’s center p_i and corresponding mesh faces f_j
        distance = compute_point_to_mesh_distance(g.center, mesh)
        if distance < d_th:
            g.type = "tightly_bound"
            tightly_bound.append(g)
        else:
            g.type = "loosely_bound"
            loosely_bound.append(g)

    # Step 5: Alignment and Rendering
    # Flatten Gti and align these splats with the mesh surface
    for g in tightly_bound:
        align_gaussian_to_mesh_surface(g, mesh)

    # Transform camera space into ray space using affine transformation Ji
    for g in gaussians:
        p_cam = view.rotation @ g.center + view.translation
        J_i = compute_ray_space_jacobian(p_cam)
        g.covariance_ray = J_i @ g.covariance @ J_i.T

    # Rendering is performed using the standard 3DGS alpha blending pipeline
    return gaussians

def compute_triangle_covariance(triangle):
    # [G] Standard initialization scales covariance to triangle dimensions
    scale = triangle.bounding_box.diagonal() / 2.0
    return np.diag(scale**2)

def rasterize_mesh_depth(view, mesh):
    # [G] Standard depth buffer rasterization of mesh triangles
    depth_map = np.full((view.height, view.width), np.inf)
    for tri in mesh.triangles:
        depth_map = rasterize_triangle_to_buffer(depth_map, tri, view)
    return depth_map

def project_to_2d(point, view):
    # [G] Standard perspective projection to normalized device coordinates
    p_cam = view.rotation @ point + view.translation
    p_ndc = view.projection @ p_cam
    return (p_ndc.x / p_ndc.w, p_ndc.y / p_ndc.w)

def is_within_image_bounds(point, view):
    # [G] Check if projected point lies within image dimensions
    p_2d = project_to_2d(point, view)
    return 0 <= p_2d.x < view.width and 0 <= p_2d.y < view.height

def compute_point_to_mesh_distance(point, mesh):
    min_dist = float('inf')
    for tri in mesh.triangles:
        dist = point_to_triangle_distance(point, tri)
        if dist < min_dist:
            min_dist = dist
    return min_dist

def point_to_triangle_distance(point, tri):
    # [G] Standard point-to-triangle distance using barycentric projection
    v0, v1, v2 = tri.vertices
    e1, e2 = v1 - v0, v2 - v0
    p1 = point - v0
    dot00 = np.dot(e1, e1)
    dot01 = np.dot(e1, e2)
    dot02 = np.dot(e1, p1)
    dot11 = np.dot(e2, e2)
    dot12 = np.dot(e2, p1)
    inv_denom = 1.0 / (dot00 * dot11 - dot01 * dot01)
    u = (dot11 * dot02 - dot01 * dot12) * inv_denom
    v = (dot00 * dot12 - dot01 * dot02) * inv_denom
    if u < 0.0 or v < 0.0 or u + v > 1.0:
        return np.linalg.norm(point - tri.closest_point(point))
    return np.linalg.norm(point - (v0 + u * e1 + v * e2))

def align_gaussian_to_mesh_surface(g, mesh):
    # [G] Flatten covariance along mesh normal and rotate to align with surface tangent plane
    normal = mesh.get_normal_at_point(g.center)
    g.covariance = flatten_covariance(g.covariance, normal)

def compute_ray_space_jacobian(p_cam):
    # [G] Standard affine approximation of projection for ray space covariance
    return np.eye(2, 3) * p_cam.z  # Simplified for pseudocode
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
