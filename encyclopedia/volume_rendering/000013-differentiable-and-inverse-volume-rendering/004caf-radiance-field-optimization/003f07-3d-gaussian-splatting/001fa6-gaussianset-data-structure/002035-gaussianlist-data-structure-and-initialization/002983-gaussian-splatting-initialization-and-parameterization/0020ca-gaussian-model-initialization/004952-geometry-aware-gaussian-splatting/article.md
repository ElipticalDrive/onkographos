# Geometry-aware Gaussian Splatting

Also called: GeoGaussian

## Overview
Geometry-aware Gaussian Splatting (GeoGaussian) is a method that initializes thin Gaussians aligned with surfaces in smoothly connected areas and uses constrained optimization to preserve scene geometry and texture, particularly in low-textured regions [S1]. It differs from standard 3D Gaussian Splatting by explicitly encoding geometric normals into the Gaussian parameterization for smooth surfaces, ensuring that densification (splitting and cloning) occurs within the tangent plane of the surface rather than randomly in 3D space [S1]. This approach enhances rendering fidelity in structured regions where traditional methods often suffer from geometric artifacts or texture degradation [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Input point cloud $P$ (typically from SLAM or SfM).
*   Camera poses $T_{cw}$ for training views.
*   Training images $I$.
*   Hyperparameters: $\lambda_1=0.8$ (photometric weight), $\lambda_2=0.3$ (geometric weight), $\beta_1=0.05$ (smoothness position weight), $\beta_2=0.01$ (smoothness normal weight), $\gamma=0.0002$ (gradient threshold), learning rate $0.0002$ [S1].

**Outputs:**
*   A set of 3D Gaussians $\mathcal{G}$, including both standard spherical Gaussians ($G_{ind}$) and thin ellipsoidal Gaussians ($G_{\theta}$), optimized to render novel views.

### Data Structures

**Standard Gaussian ($G_{ind}$):**
*   Position: $\mu \in \mathbb{R}^3$
*   Scale: $S \in \mathbb{R}^3$ (diagonal scaling matrix $S_w = \text{Diag}[s_1, s_2, s_3]$)
*   Rotation: $R \in SO(3)$
*   Opacity: $\alpha \in \mathbb{R}$
*   Spherical Harmonics: $C \in \mathbb{R}^{(m+1)\times 3}$

**Thin Gaussian ($G_{\theta}$):**
*   Position: $\mu \in \mathbb{R}^3$
*   Scale: $S = [s_1, s_2, 0.001]^T$. The third value $s_3$ is fixed to $0.001$ to enforce thinness [S1].
*   Rotation: $R = [r_1, r_2, n]$. The third column $r_3$ (denoted as $n$) is the surface normal vector, decoupled from the covariance optimization [S1]. The first two columns $r_1, r_2$ are normalized basis vectors perpendicular to $n$.
*   Opacity: $\alpha \in \mathbb{R}$
*   Spherical Harmonics: $C \in \mathbb{R}^{(m+1)\times 3}$

**Neighbor List:**
*   For each thin Gaussian $G_{\theta}$, a list of $K=8$ nearest neighbors is maintained to compute geometric constraints. This list is updated periodically [S1].

### Initialization

1.  **Normal Extraction:** Extract surface normals for all points in the input point cloud using a standard normal estimation method [S1].
2.  **Point Classification:** Classify points into two groups:
    *   $S_{ind}$ (Individual points): Points with unreliable normals. A normal is unreliable if:
        *   The point has a long distance from its neighbors.
        *   Neighbors within a distance threshold have normals with large angle differences relative to the point's normal [S1].
    *   $S_{co}$ (Smooth surfaces): Points with reliable normals, forming smoothly connected areas [S1].
3.  **Gaussian Creation:**
    *   For $p \in S_{ind}$: Initialize a standard Gaussian. Set scale $S$ based on the distance to the nearest neighbor, similar to standard 3DGS [S1].
    *   For $p \in S_{co}$: Initialize a thin Gaussian $G_{\theta}$.
        *   Set $r_3 = n$ (the extracted normal).
        *   Compute $r_1, r_2$ as orthonormal basis vectors perpendicular to $r_3$.
        *   Set $S = [s_1, s_2, 0.001]^T$. The values $s_1, s_2$ are determined by the local point density (distance to nearest neighbor) [S1].

### Densification

Densification occurs via splitting and cloning. The strategy depends on the Gaussian type.

**1. Standard Gaussian Densification ($G_{ind}$):**
*   **Split:** A large Gaussian is split into two smaller Gaussians. The new positions are chosen randomly inside the original ellipsoid [S1].

**2. Thin Gaussian Densification ($G_{\theta}$):**
*   **Cloning (Tangential Cloning):**
    *   Calculate the accumulated gradient of the position $\mu$ over the last 10 iterations, denoted $\delta \mu_i$ [S1].
    *   If $\|\delta \mu_i\| > \gamma$, clone the Gaussian.
    *   The new position $\mu_{i+1}$ is calculated by projecting the gradient step onto the tangent plane defined by the normal $r_3$:
        $$ \mu_{i+1} = \mu_i + \delta \mu_i - r_3^T \delta \mu_i $$
        *(Note: In vector notation, if $\delta \mu_i$ is a column vector, the projection of $\delta \mu_i$ onto $r_3$ is $(r_3^T \delta \mu_i) r_3$. The formula in the source [S1] $\mu_{i+1} = \mu_i + \delta \mu_i - r_3^T \delta \mu_i$ implies a scalar subtraction or a specific notation where $r_3^T \delta \mu_i$ represents the vector component along $r_3$. Given $r_3$ is a unit vector, the vector component is $(r_3 \cdot \delta \mu_i)r_3$. The source text says "decoupling the component perpendicular to the normal... to guide the position". The equation provided is $\mu_{i+1} = \mu_i + \delta \mu_i - r_3^T \delta \mu_i$. This is likely a typo in the source for $\mu_{i+1} = \mu_i + \delta \mu_i - (r_3^T \delta \mu_i)r_3$ or similar, but we must follow the source's explicit formula [S1]. However, standard projection logic suggests removing the normal component. Let's look at the splitting formula for context.)*
    *   Clone parameters $r_3, S, C, \alpha$ to the new Gaussian [S1].
*   **Splitting (Tangential Splitting):**
    *   When splitting a thin Gaussian, the new position $\mu_{i+1}$ must lie on the plane established by the original position $\mu_i$ and normal $r_3$.
    *   The condition is:
        $$ \mu_{i+1}^T r_3 = \mu_i^T r_3 $$
        And the new Gaussian $G_{i+1}$ must remain in $S_{co}$ [S1].

### Optimization

The optimization minimizes a combined loss function.

**1. Photometric Loss:**
Rendered color $c_i$ is compared to ground truth $\bar{c}_i$:
$$ E_{pho}^i = \| c_i - \bar{c}_i \| = \| f(\mathcal{G}, T_{cw}) - \bar{c}_i \| $$
where $f$ rasterizes and blends all Gaussians (standard and thin) [S1].

**2. Geometric Smoothness Loss:**
For view-dependent thin Gaussians $G_{\theta}$, compute the loss based on $K=8$ nearest neighbors $m$. Outliers with large normal angle differences are filtered out [S1].
$$ E_{geo} = \beta_1 \left\| r_3^T \mu - \frac{1}{m} \sum_{j \in m} r_3^{j T} \mu^j \right\| + \beta_2 \| r_3^j - r_3 \|_2 $$
*   $r_3^T \mu$: Distance from world origin to the plane of $G_{\theta}$.
*   $\frac{1}{m} \sum r_3^{j T} \mu^j$: Average point-plane distance of the neighborhood.
*   The second term encourages normal alignment with neighbors [S1].

**3. Total Loss:**
$$ L = \lambda_1 \sum_{\forall k \in H \times W} E_{pho}^k + \lambda_2 \sum_{G_{\theta}^i} E_{geo}^i $$
where $\lambda_1=0.8, \lambda_2=0.3$ [S1].

### Optimization Schedule

1.  **Iterations 0–2,000:** Only photometric constraints ($E_{pho}$) are used. No geometric loss [S1].
2.  **Iterations 2,000–10,000:** Both photometric and geometric losses are used. Densification (split/clone) is active [S1].
3.  **Iterations 10,000–30,000:** Densification ceases. Optimization continues with both losses [S1].
4.  **Pruning:** Gaussians with opacity $\alpha < 0.05$ are removed throughout training [S1].
5.  **Neighbor Update:**
    *   Iterations 0–20,000: Update nearest neighbor list every 100 iterations [S1].
    *   Iterations >20,000: Update every 1,000 iterations [S1].
6.  **Total Iterations:** 30,000 [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_geogaussian(point_cloud, normals, camera_poses, images):
    """
    Initialize GeoGaussian model from point cloud and normals.
    """
    gaussians = []
    
    # Classify points into S_ind and S_co
    S_ind = []
    S_co = []
    
    for i, point in enumerate(point_cloud):
        normal = normals[i]
        # Check reliability: distance to neighbors and angle consistency
        if is_normal_reliable(point, normal, point_cloud, normals):
            S_co.append(i)
        else:
            S_ind.append(i)
            
    # Initialize S_ind (Standard Gaussians)
    for idx in S_ind:
        p = point_cloud[idx]
        dist = distance_to_nearest_neighbor(p, point_cloud)
        scale = compute_scale_from_distance(dist)
        # Standard 3DGS initialization for rotation and SH
        R = random_rotation()
        C = random_sh()
        alpha = 1.0
        gaussians.append(Gaussian(pos=p, scale=scale, rot=R, sh=C, alpha=alpha, type='ind'))
        
    # Initialize S_co (Thin Gaussians)
    for idx in S_co:
        p = point_cloud[idx]
        n = normals[idx]
        dist = distance_to_nearest_neighbor(p, point_cloud)
        s1, s2 = compute_scale_from_distance(dist)
        scale = np.array([s1, s2, 0.001])
        
        # Construct rotation matrix with normal as 3rd column
        r3 = n / np.linalg.norm(n)
        r1, r2 = compute_orthonormal_basis(r3)
        R = np.column_stack((r1, r2, r3))
        
        C = random_sh()
        alpha = 1.0
        gaussians.append(Gaussian(pos=p, scale=scale, rot=R, sh=C, alpha=alpha, type='theta', normal=r3))
        
    return gaussians

def is_normal_reliable(point, normal, point_cloud, normals, threshold_dist=0.05, threshold_angle=np.pi/4):
    """
    Determine if a normal is reliable based on distance and angle consistency.
    """
    # Find neighbors within distance threshold
    neighbors = get_neighbors(point, point_cloud, threshold_dist)
    
    if len(neighbors) == 0:
        return False # Long distance from others
        
    for n_idx, n_point in neighbors:
        n_normal = normals[n_idx]
        angle = np.arccos(np.clip(np.dot(normal, n_normal), -1.0, 1.0))
        if angle > threshold_angle:
            return False
            
    return True

def compute_orthonormal_basis(r3):
    """
    Compute two orthonormal vectors r1, r2 perpendicular to r3.
    """
    # Simple method: cross product with arbitrary vector not parallel to r3
    if abs(r3[0]) < 0.9:
        arbitrary = np.array([1, 0, 0])
    else:
        arbitrary = np.array([0, 1, 0])
        
    r1 = np.cross(arbitrary, r3)
    r1 = r1 / np.linalg.norm(r1)
    r2 = np.cross(r3, r1)
    return r1, r2

def densify_geogaussian(gaussians, gradients, iteration, threshold_gamma=0.0002):
    """
    Perform densification (split/clone) based on Gaussian type.
    """
    new_gaussians = []
    active_gaussians = []
    
    for g in gaussians:
        if g.type == 'ind':
            # Standard split: random position inside ellipsoid
            if is_large(g):
                g1, g2 = split_standard(g)
                new_gaussians.extend([g1, g2])
            else:
                active_gaussians.append(g)
                
        elif g.type == 'theta':
            # Tangential cloning/splitting
            accumulated_grad = g.accumulated_gradient
            
            if np.linalg.norm(accumulated_grad) > threshold_gamma:
                # Clone
                # Formula: mu_new = mu + delta_mu - r3^T delta_mu
                # Note: Source formula is ambiguous on vector/scalar. 
                # Assuming projection: mu_new = mu + (delta_mu - (r3 . delta_mu) * r3)
                delta_mu = accumulated_grad
                projection_on_normal = np.dot(g.normal, delta_mu)
                normal_component = projection_on_normal * g.normal
                tangent_step = delta_mu - normal_component
                
                mu_new = g.pos + tangent_step
                
                # Clone parameters
                g_clone = Gaussian(
                    pos=mu_new,
                    scale=g.scale.copy(),
                    rot=g.rot.copy(),
                    sh=g.sh.copy(),
                    alpha=g.alpha,
                    type='theta',
                    normal=g.normal
                )
                new_gaussians.append(g_clone)
                
            # Splitting for thin Gaussians
            if is_large(g):
                # Split on tangent plane
                # Constraint: mu_new^T * r3 = mu_old^T * r3
                # Randomly generate a point on the plane within the ellipsoid extent
                mu_new = split_on_tangent_plane(g)
                
                g_split = Gaussian(
                    pos=mu_new,
                    scale=g.scale.copy() / 1.6, # Standard scale reduction
                    rot=g.rot.copy(),
                    sh=g.sh.copy(),
                    alpha=g.alpha / 2,
                    type='theta',
                    normal=g.normal
                )
                new_gaussians.append(g_split)
            else:
                active_gaussians.append(g)
                
    return active_gaussians + new_gaussians

def split_on_tangent_plane(g):
    """
    Generate a new position on the tangent plane of g.
    """
    # Plane equation: (mu - g.pos) . g.normal = 0
    # Generate random offset in the tangent space (spanned by r1, r2)
    offset = np.random.randn(2)
    tangent_offset = offset[0] * g.rot[:, 0] + offset[1] * g.rot[:, 1]
    
    # Scale offset by ellipsoid size
    scale_factor = np.random.uniform(0, 1) * np.max(g.scale[:2])
    tangent_offset *= scale_factor
    
    return g.pos + tangent_offset

def compute_loss(gaussians, images, camera_poses, neighbor_lists, iteration):
    """
    Compute total loss.
    """
    photometric_loss = 0.0
    geometric_loss = 0.0
    
    # Photometric loss for all visible Gaussians
    for i, (img, pose) in enumerate(zip(images, camera_poses)):
        rendered = render(gaussians, pose)
        photometric_loss += np.mean((rendered - img) ** 2)
        
    # Geometric loss for thin Gaussians
    if iteration > 2000:
        for g in gaussians:
            if g.type == 'theta':
                neighbors = neighbor_lists[g.id]
                if len(neighbors) > 0:
                    # Filter outliers
                    valid_neighbors = []
                    for n in neighbors:
                        angle = np.arccos(np.clip(np.dot(g.normal, n.normal), -1.0, 1.0))
                        if angle < np.pi/4: # Example threshold
                            valid_neighbors.append(n)
                            
                    if len(valid_neighbors) > 0:
                        m = len(valid_neighbors)
                        # Term 1: Planar consistency
                        dist_self = np.dot(g.normal, g.pos)
                        dist_avg = np.mean([np.dot(n.normal, n.pos) for n in valid_neighbors])
                        term1 = np.abs(dist_self - dist_avg)
                        
                        # Term 2: Normal alignment
                        term2 = np.mean([np.linalg.norm(g.normal - n.normal) for n in valid_neighbors])
                        
                        geometric_loss += 0.05 * term1 + 0.01 * term2
                        
    lambda1 = 0.8
    lambda2 = 0.3
    total_loss = lambda1 * photometric_loss + lambda2 * geometric_loss
    return total_loss

def update_neighbor_lists(gaussians, update_interval, current_iter):
    """
    Update K-NN lists for thin Gaussians.
    """
    if current_iter % update_interval == 0:
        for g in gaussians:
            if g.type == 'theta':
                neighbors = get_k_nearest_neighbors(g.pos, gaussians, k=8)
                # Filter by type? Source says "nearest neighbors", implies all, but loss uses normals.
                # Assuming we store all and filter in loss.
                g.neighbors = neighbors
```

## Operating characteristics

**Time and Memory Complexity:**
*   The method requires more GPU resources for training compared to standard 3DGS due to the additional geometric loss computation and neighbor list maintenance [S1].
*   Neighbor lists are updated periodically (every 100 iterations until 20,000, then every 1,000), adding overhead proportional to the number of thin Gaussians [S1].

**Measured Performance:**
*   Evaluated on Replica, TUM RGB-D, and ICL-NUIM datasets [S1].
*   **Sparse View Evaluation (R1, 10% data):**
    *   GeoGaussian: PSNR 31.65, SSIM 0.937, LPIPS 0.041.
    *   3DGS: PSNR 30.49, SSIM 0.932, LPIPS 0.051.
    *   LightGS: PSNR 30.54, SSIM 0.932, LPIPS 0.051 [S1].
*   **Sparse View Evaluation (R1, 100% data):**
    *   GeoGaussian: PSNR 38.24, SSIM 0.979, LPIPS 0.021.
    *   3DGS: PSNR 37.60, SSIM 0.965, LPIPS 0.028.
    *   LightGS: PSNR 38.44, SSIM 0.967, LPIPS 0.025 [S1].

**Strengths and Limitations:**
*   **Strengths:** Improves rendering quality in novel views, especially in low-textured regions and structured areas (walls, ceilings) where standard 3DGS degrades [S1]. Preserves geometry and texture through constrained optimization [S1].
*   **Limitations:** Requires more training time and GPU resources than standard 3DGS. It functions as a global refinement module and cannot perform real-time optimization, unlike online methods such as Vox-Fusion [S1].

**When to Use:**
*   Use when high-fidelity rendering of structured scenes is required, particularly with sparse input views or in regions with low texture.
*   Use when the input data includes reliable surface normals (e.g., from high-quality SLAM/SfM).

**When Not to Use:**
*   Do not use if real-time optimization is required.
*   Do not use if the input point cloud has poor normal estimation, as the classification of $S_{co}$ may be unreliable.

## Background

**Origin:**
*   GeoGaussian was proposed by Y. Li et al. in the paper "GeoGaussian: Geometry-Aware Gaussian Splatting for Scene Rendering" [S1].
*   The paper was published in 2024 (based on the DOI prefix and context of 3DGS evolution, though exact year is not explicitly in the snippet, the DOI 10.1007/978-3-031-72761-0_25 suggests a 2024 LNCS volume) [S1].

**History and Development:**
*   The method builds upon 3D Gaussian Splatting [19] and addresses its limitations in handling smooth surfaces and low-texture regions [S1].
*   It contrasts with voxel-based methods like Vox-Fusion, which integrate implicit neural representations and operate online, whereas GeoGaussian is a global refinement method using differentiable 3D Gaussians [S1].

**Usage:**
*   Evaluated on Replica, TUM RGB-D, and ICL-NUIM datasets [S1].

## Sources

[S1] GeoGaussian: Geometry-Aware Gaussian Splatting for Scene Rendering, https://doi.org/10.1007/978-3-031-72761-0_25, fetched 2026-10-26
