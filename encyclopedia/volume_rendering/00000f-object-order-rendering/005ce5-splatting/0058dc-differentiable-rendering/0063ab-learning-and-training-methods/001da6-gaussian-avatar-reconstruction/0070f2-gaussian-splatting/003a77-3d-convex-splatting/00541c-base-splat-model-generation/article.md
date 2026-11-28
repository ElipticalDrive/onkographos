# Base Splat Model Generation

Also called: Elliptical Splat Generation, PCA-based Splat Fitting

## Overview
Base Splat Model Generation replaces isolated 3D points with elliptical surface patches (splats) that approximate the underlying local surface geometry, addressing the limitation of points as mere isolated samples [S1]. It differs from its parent, 3D Convex Splatting, by using PCA-derived elliptical primitives rather than smooth convex shapes, and from its sibling, Compositional Splatting Strategy, by generating the initial model via local neighbor statistics rather than compositional merging [S1, 13273-compositional-splatting-strategy.md].

## How it works
**Inputs:** A sparse 3D point cloud $P = \{p_i\}_{i=0}^{M-1}$ where each $p_i \in \mathbb{R}^3$, and for each point, the camera position $c_i$ that observed it.
**Outputs:** A set of splats, each defined by a center $s_i$, a normal vector $n_i$, and two orthogonal tangential vectors $u_i, v_i$ representing the major and minor axes.

**Data Structures:**
*   **Voxel Grid:** A 3D grid used to accelerate neighbor search. Points are hashed into voxels based on their coordinates.
*   **Neighbor List $N_r(p_i)$:** The set of points within a fixed radius $r$ of $p_i$.

**Parameters:**
*   $r$: Fixed search radius for neighbors. [S1]
*   Threshold: Empirical threshold for the flatness coefficient $f_i$ to decide splat center placement. [S1]

**Steps:**

1.  **Voxelization:** Discretize the 3D space into voxels. Assign each point $p_i$ to its corresponding voxel.
2.  **Neighbor Search:** For each point $p_i$, identify neighbors $N_r(p_i)$ by checking the point's voxel and its 26 adjacent voxels. Filter points within distance $r$. [S1]
3.  **PCA Computation:** Perform Principal Component Analysis on the neighbor set $N_r(p_i)$:
    *   Compute the average point $\bar{p}_i = \frac{1}{|N_r(p_i)|} \sum_{q \in N_r(p_i)} q$. [S1]
    *   Compute the covariance matrix of the neighbors.
    *   Solve the eigenvalue problem to obtain three eigenvalues $\lambda_1 \ge \lambda_2 \ge \lambda_3$ and their corresponding eigenvectors $w_1, w_2, w_3$. [S1]
4.  **Orientation and Axis Calculation:**
    *   Set the normal vector $n_i = w_3$ (eigenvector of smallest eigenvalue). [S1]
    *   Compute the tangential vectors (axes):
        $$u_i = \sqrt{\lambda_1} \cdot w_1$$
        $$v_i = \sqrt{\lambda_2} \cdot w_2$$ [S1]
    *   **Normal Flipping:** Ensure $n_i$ points outwards from the model interior. If $n_i \cdot (c_i - p_i) < 0$, flip the orientation of $n_i$ and adjust $u_i, v_i$ to maintain the right-handed cross-product relationship ($u_i \times v_i = n_i$). [S1]
5.  **Flatness Coefficient:** Calculate the planarity measure:
    $$f_i = \frac{\sqrt{\lambda_1} + \sqrt{\lambda_2}}{\sqrt{\lambda_1} + \sqrt{\lambda_2} + \sqrt{\lambda_3}}$$ [S1]
    *   $f_i = 1$ indicates a completely planar area ($\lambda_3 = 0$).
    *   $f_i = 2/3$ indicates a perfectly isotropic point set ($\lambda_1 = \lambda_2 = \lambda_3$). [S1]
6.  **Splat Center Determination:**
    *   If $f_i$ is high (above the empirical threshold), set the splat center $s_i = p_i$. [S1]
    *   If $f_i$ is low, set the splat center $s_i = \bar{p}_i$ (the average point from PCA) to reduce holes in flat regions. [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def base_splat_generation(points, camera_positions, radius, flatness_threshold):
    """
    Generate base splats from a point cloud using PCA.
    
    Args:
        points: Array of shape (M, 3) containing 3D coordinates.
        camera_positions: Array of shape (M, 3) containing camera position for each point.
        radius: Search radius for neighbors.
        flatness_threshold: Empirical threshold for flatness coefficient.
        
    Returns:
        splats: List of dictionaries with keys 'center', 'normal', 'u', 'v'.
    """
    M = points.shape[0]
    splats = []
    
    # Step 1: Voxelization
    # Hash points to voxels. Assuming a simple grid for illustration.
    voxel_size = radius
    voxel_map = {} # Key: (vx, vy, vz), Value: list of point indices
    
    for i, p in enumerate(points):
        vx, vy, vz = int(p[0] / voxel_size), int(p[1] / voxel_size), int(p[2] / voxel_size)
        if (vx, vy, vz) not in voxel_map:
            voxel_map[(vx, vy, vz)] = []
        voxel_map[(vx, vy, vz)].append(i)
        
    # Step 2 & 3: Neighbor Search and PCA
    for i in range(M):
        p_i = points[i]
        c_i = camera_positions[i]
        
        # Find neighbors in 27 voxels (current + 26 adjacent)
        vx, vy, vz = int(p_i[0] / voxel_size), int(p_i[1] / voxel_size), int(p_i[2] / voxel_size)
        neighbors = []
        
        for dx in [-1, 0, 1]:
            for dy in [-1, 0, 1]:
                for dz in [-1, 0, 1]:
                    key = (vx + dx, vy + dy, vz + dz)
                    if key in voxel_map:
                        for j in voxel_map[key]:
                            dist_sq = np.sum((points[j] - p_i)**2)
                            if dist_sq <= radius**2:
                                neighbors.append(points[j])
        
        if len(neighbors) < 3:
            # Degenerate case: not enough neighbors for PCA
            # [G] Default to identity orientation if insufficient neighbors
            splats.append({
                'center': p_i,
                'normal': np.array([0, 0, 1]),
                'u': np.array([1, 0, 0]),
                'v': np.array([0, 1, 0])
            })
            continue
            
        neighbors_arr = np.array(neighbors)
        
        # PCA
        mean_p = np.mean(neighbors_arr, axis=0)
        centered = neighbors_arr - mean_p
        cov = np.cov(centered, rowvar=False)
        
        eigenvalues, eigenvectors = np.linalg.eigh(cov)
        
        # Sort eigenvalues and eigenvectors in descending order
        idx = np.argsort(eigenvalues)[::-1]
        eigenvalues = eigenvalues[idx]
        eigenvectors = eigenvectors[:, idx]
        
        w1, w2, w3 = eigenvectors[:, 0], eigenvectors[:, 1], eigenvectors[:, 2]
        lambda1, lambda2, lambda3 = eigenvalues[0], eigenvalues[1], eigenvalues[2]
        
        # Step 4: Orientation and Axes
        n_i = w3
        u_i = np.sqrt(lambda1) * w1
        v_i = np.sqrt(lambda2) * w2
        
        # Flip normal if pointing towards camera
        if np.dot(n_i, c_i - p_i) < 0:
            n_i = -n_i
            # Adjust tangents to maintain right-handed system: u x v = n
            # If we flip n, we can flip u or v. Flipping u is sufficient.
            u_i = -u_i
            
        # Step 5: Flatness Coefficient
        sqrt_l1 = np.sqrt(lambda1)
        sqrt_l2 = np.sqrt(lambda2)
        sqrt_l3 = np.sqrt(lambda3)
        
        f_i = (sqrt_l1 + sqrt_l2) / (sqrt_l1 + sqrt_l2 + sqrt_l3)
        
        # Step 6: Splat Center
        if f_i > flatness_threshold:
            s_i = p_i
        else:
            s_i = mean_p
            
        splats.append({
            'center': s_i,
            'normal': n_i,
            'u': u_i,
            'v': v_i
        })
        
    return splats
```

## Subtopics

### [Covariance Matrix (CM) Splatting](15385-covariance-matrix-cm-splatting.md)
A variant that explicitly solves the eigenvalue/eigenvector problem from local neighbors to align splats with minimal curvature. It differs from the base method by focusing on surface-aligned orientations and is chosen when shape-aware, rotationally invariant orientations are required, though it may produce less coherent fields than global methods.

## Operating characteristics
*   **Time Complexity:** Dominated by neighbor search. With voxelization, average case is $O(M)$ for $M$ points, assuming constant average neighbors within radius $r$. Without voxelization, it is $O(M^2)$. PCA is $O(k)$ where $k$ is the number of neighbors (typically small constant).
*   **Memory Complexity:** $O(M)$ to store the point cloud, voxel grid, and resulting splats.
*   **Strengths:** Adapts splat shape to local geometry; reduces holes in flat regions by moving centers to the local mean; filters noise via fixed-radius search.
*   **Limitations:** Requires dense enough sampling for PCA to be meaningful; sensitive to the choice of radius $r$ and flatness threshold; local orientation may lack global coherence compared to methods using global tangent fields.
*   **When to use:** When starting from a sparse or moderately dense point cloud and needing an initial approximation of surface geometry with elliptical primitives.
*   **When not to use:** When global surface coherence is critical without post-processing, or when the point cloud is extremely sparse (PCA fails).

## Background
The method was introduced in the context of textured splat-based point clouds for rendering on handheld devices [S1]. It builds on earlier work by Pauly et al. [2002] for hierarchical clustering and Wu and Kobbelt [2004; 2005] for optimized subsampling, but specifically adapts PCA to define elliptical splat shapes from local neighbor statistics [S1]. The approach aims to improve model compactness and rendering quality, particularly for multi-texturing, by replacing isolated points with surface-adaptive patches [S1].

## Sources
[S1] Textured splat-based point clouds for rendering in handheld devices, https://doi.org/10.1145/2775292.2782779, fetched 2026-11-03
