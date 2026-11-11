# Distance-based Gaussian Splatting

Also called: Mesh-based Gaussian Splatting, Tightly/Loosely-bound Gaussian Splatting

## Overview
Distance-based Gaussian Splatting (MeshGS) connects a large-scale 3D mesh with Gaussian splats by classifying primitives as tightly-bound or loosely-bound based on their distance to the mesh surface, while masking out splats occluded by the mesh [S1]. It differs from sibling methods like Gaussian Opacity Fields (GOF) by explicitly leveraging a pre-existing mesh geometry to prune redundant internal splats and align surface-aligned splats, rather than relying solely on volumetric opacity fields or 2D segmentation masks [S1, S2, S3].

## How it works
**Inputs and Outputs**
*   **Inputs:** A trained set of $N$ 3D Gaussian primitives $\mathcal{G}$ (initialized from mesh triangles), a 3D mesh $M$ (vertices and faces), and a set of training views.
*   **Outputs:** A refined set of Gaussian primitives optimized for high-quality rendering, categorized into tightly-bound ($G_t$) and loosely-bound ($G_l$) splats.

**Data Structures**
*   **Mesh $M$:** Defined by a set of triangles $f_j$.
*   **Gaussian Primitive $g_i$:**
    *   Center position $\boldsymbol{p}_i \in \mathbb{R}^3$.
    *   3D Covariance $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3\times3}$, parameterized by scale $\mathbf{S}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in \mathbb{R}^{3\times3}$ (encoded by quaternion $\mathbf{q} \in \mathbb{R}^4$) such that $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$.
    *   Opacity $o_i \in [0, 1]$.
    *   Spherical Harmonics coefficients $\mathbf{c}_i$.

**Numbered Steps**

1.  **Initialization:** Initialize a 3D Gaussian splat at the center $\boldsymbol{m}_j$ of each triangle $f_j$ of the mesh $M$ [S1].
2.  **Occlusion Masking:** For each training viewpoint $\boldsymbol{P}_i$, render a depth map $D_i$. Assume the mesh surface is entirely opaque. Mask out Gaussian splats that are occluded by the mesh surface in the rendered depth [S1].
3.  **Visibility Assessment:** Assess the visibility of Gaussian splats across all training views. Identify splats located behind the mesh surface that are never observed by any training view [S1].
4.  **Pruning:** Remove all redundant Gaussian splats situated behind the mesh surface. This pruning occurs at specific training iterations (e.g., every 500 iterations) [S1].
5.  **Distance Calculation:** Calculate the Euclidean distance $d_i$ between each Gaussian splat’s center $\boldsymbol{p}_i$ and the corresponding mesh faces $f_j$ [S1].
6.  **Classification:** Classify Gaussian splats based on the distance $d_i$ and a threshold $d_{th}$:
    *   **Tightly-bound ($G_{ti}$):** If $d_i < d_{th}$.
    *   **Loosely-bound ($G_{li}$):** If $d_i > d_{th}$ [S1].
7.  **Alignment:** Flatten the tightly-bound splats $G_{ti}$ and align them with the mesh surface [S1].
8.  **Rendering Projection:** To render, transform camera space into ray space using an affine transformation $\mathbf{J}_i \in \mathbb{R}^{2\times3}$ defined by the camera position $\boldsymbol{p}_{ci}$. Compute the 2D covariance in ray space:
    $$ \boldsymbol{\Sigma}'_i = \mathbf{J}_i \boldsymbol{\Sigma}_{ic} \mathbf{J}_i^T $$
    where $\boldsymbol{\Sigma}_{ic}$ is the 3D covariance in camera space [S1].

**Formulas**
*   **3D Gaussian Definition:**
    $$ G_{i}(\boldsymbol{x}) = e^{-\frac {1}{2}(\boldsymbol{x}-\boldsymbol{p}_{i})^T \boldsymbol{\Sigma }_{i}^{-1}(\boldsymbol{x}-\boldsymbol{p}_{i})} $$
    where $\boldsymbol{p}_i$ is the center and $\boldsymbol{\Sigma}_i$ is the 3D covariance [S1].
*   **2D Gaussian Definition:**
    $$ g_{i}(\boldsymbol{x}) = e^{-\frac {1}{2}(\boldsymbol{x}-\boldsymbol{p}'_{i})^T {\boldsymbol{\Sigma }'_{i}}^{-1}(\boldsymbol{x}-\boldsymbol{p}'_{i})} $$
    where $\boldsymbol{p}'_i$ is the 2D center and $\boldsymbol{\Sigma}'_i$ is the 2D covariance [S1].
*   **Rendering Equation:**
    $$ c = \sum _{i=1}^{n} c_{i}\alpha _{i}\prod _{j=1}^{i-1}(1-\alpha _{j}), \quad \alpha _{i} = o_{i}g_{i}(\boldsymbol{x}) $$
    where $c_i$ is view-dependent color, $o_i$ is opacity, and $g_i(\boldsymbol{x})$ is the 2D Gaussian value [S1].

**Parameters**
*   **Depth Offset:** Add $0.01$ to the rendered depth $D_i$ when applying occlusion masking [S1].
*   **Pruning Interval:** Prune splats behind the mesh every $500$ training iterations [S1].
*   **Distance Threshold ($d_{th}$):** Set to $0.01$ to distinguish tightly-bound from loosely-bound splats [S1].
*   **Loss Weights:** $\lambda_{img} = 0.001$, $\lambda_{nc} = 0.1$, $\lambda_{min} = 0.1$, $\lambda_{max} = 10$, $\lambda_{proj} = 50$ [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_based_gaussian_splatting(mesh, training_views, num_iterations=30000):
    """
    Implements Distance-based Gaussian Splatting (MeshGS).
    
    Args:
        mesh: 3D mesh object with triangles f_j and vertices.
        training_views: List of camera poses and images.
        num_iterations: Total training iterations.
        
    Returns:
        Optimized set of Gaussian primitives.
    """
    # 1. Initialization
    gaussians = []
    for triangle in mesh.triangles:
        center = triangle.center
        # Initialize Gaussian at triangle center
        g = Gaussian(center=center)
        gaussians.append(g)
        
    # Parameters
    d_th = 0.01
    prune_interval = 500
    depth_offset = 0.01
    
    for iteration in range(1, num_iterations + 1):
        # 2. Occlusion Masking & 3. Visibility Assessment
        occluded_indices = set()
        for view in training_views:
            # Render depth map from current view
            depth_map = render_depth(gaussians, view.camera)
            
            # Apply depth offset
            depth_map += depth_offset
            
            # Identify splats occluded by mesh surface
            # Assume mesh is opaque; if splat depth > mesh depth, it is occluded
            for idx, g in enumerate(gaussians):
                if is_occluded_by_mesh(g.center, view.camera, mesh, depth_map):
                    occluded_indices.add(idx)
                    
        # 4. Pruning (at specific intervals)
        if iteration % prune_interval == 0:
            # Remove splats that are behind the mesh and never observed
            # (Simplified: remove those identified as occluded/redundant)
            gaussians = [g for i, g in enumerate(gaussians) if i not in occluded_indices]
            
        # 5. Distance Calculation & 6. Classification
        tightly_bound = []
        loosely_bound = []
        
        for g in gaussians:
            # Calculate distance to nearest mesh face
            dist = distance_to_mesh(g.center, mesh)
            
            if dist < d_th:
                tightly_bound.append(g)
            else:
                loosely_bound.append(g)
                
        # 7. Alignment for tightly-bound splats
        for g in tightly_bound:
            align_with_mesh_surface(g, mesh)
            
        # 8. Rendering and Optimization
        # Standard 3DGS optimization step using L1 and D-SSIM losses
        # plus normal consistency loss for tightly-bound splats
        optimize_gaussians(gaussians, training_views, tightly_bound)
        
    return gaussians

def is_occluded_by_mesh(point, camera, mesh, depth_map):
    """
    Checks if a 3D point is occluded by the mesh surface in the given view.
    """
    # Project point to 2D screen coordinates
    screen_pos = project_to_screen(point, camera)
    
    # Get depth of the point in camera space
    point_depth = get_depth_in_camera_space(point, camera)
    
    # Get depth of the mesh surface at screen_pos from depth_map
    mesh_depth = depth_map[screen_pos.y][screen_pos.x]
    
    # If point is behind the mesh surface (greater depth value)
    return point_depth > mesh_depth

def distance_to_mesh(point, mesh):
    """
    Calculates the minimum Euclidean distance from a point to the mesh faces.
    """
    min_dist = infinity
    for face in mesh.faces:
        dist = point_to_triangle_distance(point, face)
        if dist < min_dist:
            min_dist = dist
    return min_dist

def align_with_mesh_surface(gaussian, mesh):
    """
    Flattens the Gaussian and aligns its orientation with the mesh surface normal.
    """
    # Find nearest face
    nearest_face = get_nearest_face(gaussian.center, mesh)
    
    # Get surface normal
    normal = nearest_face.normal
    
    # Adjust Gaussian covariance to be flat along the normal
    # This constrains the scale along the normal to be very small
    gaussian.scale = adjust_scale_for_alignment(gaussian.scale, normal)
    
    # Adjust rotation to align with normal
    gaussian.rotation = align_rotation_with_normal(gaussian.rotation, normal)
```

**Edge Cases and Pitfalls**
*   **Mesh Artifacts:** Real-world reconstructed meshes often exhibit geometric artifacts. Tightly binding all splats exclusively to the surface can suffer rendering performance issues in these regions. The loose-bound classification helps mitigate this by allowing splats to deviate slightly from the surface [S1].
*   **Precision:** The depth offset ($0.01$) is critical to prevent self-occlusion or incorrect masking due to floating-point precision errors between the rendered depth and the mesh geometry [S1].
*   **Redundancy:** Without pruning, initializing a Gaussian per triangle can lead to a rapid increase in GPU memory usage, especially if the mesh has a high triangle count (e.g., 44 million triangles) [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and the number of training views. Pruning every 500 iterations reduces the number of Gaussians over time, potentially speeding up later iterations.
*   **Memory Complexity:** Proportional to the number of Gaussians. Initialization scales with the number of mesh triangles. Pruning reduces memory usage compared to initializing splats for every triangle without removal [S1].
*   **Performance:** The method is designed for high-quality rendering on large-scale scenes. It balances geometric fidelity (via tight binding) and robustness to mesh artifacts (via loose binding) [S1].
*   **Strengths:** Reduces redundant splats, aligns surface geometry with appearance, and handles mesh artifacts better than strict surface binding.
*   **Limitations:** Relies on the quality of the input mesh. If the mesh is poor, the classification and alignment may not yield optimal results.
*   **When to use:** When a high-quality mesh is available and the goal is to extract a mesh-aligned Gaussian representation for high-fidelity rendering.
*   **When not to use:** When no mesh is available or when the scene geometry is too complex/unstructured for a clean mesh representation.

## Background
Distance-based Gaussian Splatting was introduced in the paper "MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering" by Jaehoon et al. [S1]. It builds upon the original 3D Gaussian Splatting (3DGS) framework [S1] and aims to address the issue of redundant Gaussian splats and geometric artifacts in large-scale mesh reconstruction. The method was developed to improve rendering quality by adaptively aligning Gaussians with the mesh surface while allowing flexibility for inaccuracies.

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
