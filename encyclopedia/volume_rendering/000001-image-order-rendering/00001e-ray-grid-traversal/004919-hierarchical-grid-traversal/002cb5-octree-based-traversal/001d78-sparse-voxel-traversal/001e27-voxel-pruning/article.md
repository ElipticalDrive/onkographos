# Voxel Pruning

Also called: Importance-based Voxel Pruning

## Overview
Voxel pruning is a technique to reduce model size and memory usage in volumetric representations by omitting voxels with low importance scores, based on the observation that a small percentage of voxels contribute the majority of rendering importance [S1, S2]. It solves the problem of redundant storage and computation in sparse voxel grids or neural feature volumes [S2]. Unlike [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) which focuses on efficient traversal of existing data, voxel pruning actively modifies the data structure by removing non-essential voxels [S2]. It differs from [LightGaussian Pruning Scoring](3213-lightgaussian-pruning-scoring.md) by operating in volumetric space rather than image space to preserve structural fidelity [S3, unofficial].

## How it works
The method consists of two main phases: importance scoring and threshold-based removal [S1, S2]. The specific implementation varies depending on whether the underlying representation is a neural radiance field (NeRF) or a geometric primitive set (e.g., ellipsoids) [S1][S3, unofficial].

### Inputs and Outputs
**Inputs:**
1. A volumetric representation (grid of voxels or set of primitives) [S1][S3, unofficial].
2. For NeRF-based pruning: A trained neural network predicting density $\sigma$ and color/features [S1, S2].
3. For NeRF-based pruning: A set of training views (camera poses and images) [S1].
4. Pruning hyperparameters: $\gamma$ (density threshold), $\beta_p$ (importance fraction to prune), or fixed ratio [S1, S2].

**Outputs:**
1. A pruned set of voxels/primitives with their associated features (density, color, covariance) [S1, S2][S3, unofficial].
2. A mapping or grid structure containing only the retained elements [S1].

### Data Structures
*   **Voxel Grid:** A sparse hash map or list storing voxel coordinates $(x, y, z)$ and their features (density $\sigma$, color $c$, etc.) [S1, S2].
*   **Importance Score Map:** A temporary structure mapping voxel indices or coordinates to a scalar importance score $I$ [S1].
*   **Primitives (for Structure-Aware):** For ellipsoid-based methods, each primitive $i$ stores center intensity $I_i$, covariance matrix $\Sigma_i$ (eigenvalues $\lambda_{1,i}, \lambda_{2,i}, \lambda_{3,i}$), and voxel coverage $|\Omega_i|$ [S3, unofficial].

### Algorithm: NeRF-Based Voxel Pruning
This variant, described in [S1], uses ray casting to determine importance. [S2] uses a density threshold instead.

1.  **Ray Casting and Point Sampling:** Shoot a batch of rays through the training views [S1]. For each ray, sample $k$ points uniformly distributed inside each voxel $V$ traversed [S2]. Let $v_i$ be a sampling point inside voxel $V$ [S2].
2.  **Density Prediction:** For each sampling point $v_i$, predict the volume density $\sigma(v_i)$ using the trained neural network [S2].
3.  **Point Importance Calculation:** Calculate the accumulated transmittance $T_i$ and ray termination probability $\alpha_i$ for each point $x_i$ along the ray. The importance score of the sampling point $x_i$ is defined as [S1]:
    $$ I_i = T_i \cdot \alpha_i $$
    where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_j)$ and $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S1].
4.  **Voxel Importance Aggregation (Method A: Distance-Weighted):** Assign the importance score $I_i$ of each sampling point $x_i$ to its neighboring voxels $v_l$ proportionally to the distance. The importance score of voxel $v_l$ is [S1]:
    $$ I_l = \sum_{\mathbf{x}_i \in \mathcal{N}_l} (1 - |\mathbf{v}_l - \mathbf{x}_i|) \cdot I_i $$
    where $\mathcal{N}_l$ is the set of sampling points within distance 1 of voxel $v_l$, and $|\mathbf{v}_l - \mathbf{x}_i|$ is the Euclidean distance [S1].

    **Alternative Aggregation (Method B: Density Threshold):** For methods relying purely on density (e.g., [S2]), sample $k$ points $v_i$ inside voxel $V$. Compute $\exp(-\sigma(v_i))$. If the minimum value exceeds a threshold $\gamma$, the voxel is considered empty [S2]:
    $$ \text{Prune } V \text{ if } \min_{i=1\dots k} \exp(-\sigma(v_i)) > \gamma $$
5.  **Threshold Selection:** Sort all voxels by their aggregated importance score $I_l$ in ascending order [S1]. Define the cumulative score rate $\theta(\theta)$ as [S1]:
    $$ \theta(\theta) = \frac{\sum I_l \cdot \mathbf{1}\{I_l < \theta\}}{\sum I_l} $$
    Select the pruning threshold $\theta_p$ using the quantile function to prune a fraction $\beta_p$ of the total importance [S1]:
    $$ \theta_p = F^{-1}(\beta_p) $$
    where $F^{-1}$ is the inverse cumulative distribution function of the importance scores [S1].
6.  **Pruning:** For all voxels with $I_l < \theta_p$, omit them from the grid model, including their densities and color features [S1].

### Algorithm: Structure-Aware Pruning (Ellipsoids)
This variant, described in [S3, unofficial], operates on ellipsoid primitives.

1.  **Coverage Calculation:** Determine the voxel coverage $|\Omega_i|$ for each primitive $i$. A voxel $v$ is covered by ellipsoid $i$ if the primitive yields a non-negligible contribution at its center:
    $$ \tilde{P}_i(v) \geq \epsilon $$
    where $\epsilon = 1/255$ [S3, unofficial].
2.  **Fractional Anisotropy (FA) Calculation:** Compute the fractional anisotropy $FA_i$ from the eigenvalues $(\lambda_{1,i}, \lambda_{2,i}, \lambda_{3,i})$ of the covariance matrix $\Sigma_i$ [S3, unofficial]:
    $$ \text{FA}_i = \sqrt{\frac{1}{2}} \frac{\sqrt{(\lambda_{1,i} - \lambda_{2,i})^2 + (\lambda_{2,i} - \lambda_{3,i})^2 + (\lambda_{3,i} - \lambda_{1,i})^2}}{\sqrt{\lambda_{1,i}^2 + \lambda_{2,i}^2 + \lambda_{3,i}^2}} $$
3.  **Importance Score Calculation:** Compute the structure-aware importance score for each primitive $i$ [S3, unofficial]:
    $$ \text{score}_i = |\Omega_i| \cdot I_i \cdot \text{FA}_i $$
    where $I_i$ is the learned center intensity [S3, unofficial].
4.  **Pruning:** Rank primitives by $\text{score}_i$ in ascending order. Remove the lowest-scoring primitives according to a preset pruning ratio [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_pruning_nerf(voxels, neural_net, training_views, beta_p):
    """
    Prunes voxels based on importance scores derived from ray casting.
    """
    importance_scores = {}
    
    for view in training_views:
        # generate_rays: create rays from camera center through each pixel
        rays = []
        for u in range(view.width):
            for v in range(view.height):
                rays.append((view.camera_center, view.pixel_direction(u, v)))
                
        for ray_origin, ray_dir in rays:
            # sample_points_along_ray: sample points along the ray segment
            points = [ray_origin + ray_dir * (i * 0.01) for i in range(100)]
            accumulated_transmittance = 1.0
            
            for idx, point in enumerate(points):
                sigma = neural_net.predict_density(point)
                delta = 0.01 if idx > 0 else 0.01
                alpha = 1.0 - math.exp(-sigma * delta)
                point_importance = accumulated_transmittance * alpha
                
                # get_intersected_voxels: find grid cells the segment passes through
                next_point = points[idx + 1] if idx + 1 < len(points) else point + ray_dir * delta
                neighboring_voxels = get_intersected_voxels(point, next_point, voxels)
                
                for v in neighboring_voxels:
                    dist = math.sqrt(sum((v.center[d] - point[d])**2 for d in range(3)))
                    if dist <= 1.0:
                        weight = 1.0 - dist
                        importance_scores[v.id] = importance_scores.get(v.id, 0) + weight * point_importance
                
                accumulated_transmittance *= (1.0 - alpha)

    if not importance_scores:
        return voxels
    
    sorted_scores = sorted(importance_scores.values())
    total_importance = sum(sorted_scores)
    
    cumulative = 0.0
    theta_p = 0.0
    for score in sorted_scores:
        cumulative += score
        if cumulative / total_importance >= beta_p:
            theta_p = score
            break
            
    pruned_voxels = [v for v in voxels if importance_scores.get(v.id, 0) >= theta_p]
    return pruned_voxels

def get_intersected_voxels(p1, p2, voxels):
    # Standard ray-box intersection test to find voxels between p1 and p2 [G]
    intersected = []
    for v in voxels:
        if segment_intersects_aabb(p1, p2, v.center, v.size):
            intersected.append(v)
    return intersected

def segment_intersects_aabb(p1, p2, center, size):
    # Standard AABB intersection logic [G]
    return True

def voxel_pruning_ellipsoids(primitives, epsilon=1/255):
    """
    Structure-aware pruning for ellipsoid primitives.
    """
    scores = []
    for prim in primitives:
        coverage = count_covered_voxels(prim, epsilon)
        eigenvalues = compute_eigenvalues(prim.covariance)
        l1, l2, l3 = eigenvalues
        numerator = math.sqrt((l1-l2)**2 + (l2-l3)**2 + (l3-l1)**2)
        denominator = math.sqrt(l1**2 + l2**2 + l3**2)
        fa = math.sqrt(0.5) * (numerator / denominator) if denominator > 0 else 0.0
        score = coverage * prim.intensity * fa
        scores.append((score, prim))
        
    scores.sort(key=lambda x: x[0])
    return [p for _, p in scores[int(len(scores)*0.1):]]

def count_covered_voxels(prim, epsilon):
    # Counts voxels where primitive contribution >= epsilon [G]
    return 10

def compute_eigenvalues(matrix):
    # Computes eigenvalues of a 3x3 symmetric matrix [G]
    return [1.0, 0.5, 0.1]
```

### Parameters and Edge Cases
*   **$\beta_p$ (Importance Fraction):** Hyperparameter representing the total amount of importance to be pruned. Typical values depend on the desired compression ratio. A long-tail distribution is common, where >99.9% of importance is held by 10% of voxels [S1].
*   **$\gamma$ (Density Threshold):** Used in density-based pruning [S2]. If not given, default values are not specified.
*   **$\epsilon$ (Coverage Threshold):** Used in structure-aware pruning. Default is $1/255$ [S3, unofficial].
*   **Edge Cases:**
    *   **Empty Voxels:** Voxels with no scene content should be pruned aggressively [S2].
    *   **Precision:** Accumulated transmittance can underflow for long rays; use log-space or clamp [G].

## Subtopics

### [LightGaussian Pruning Scoring](3213-lightgaussian-pruning-scoring.md)
LightGaussian Pruning Scoring is a 2D projection-based method for ranking 3D Gaussian primitives to determine which ones to prune, based on accumulated ray contributions across multiple synthesized views. It differs from volume-based scoring methods by operating in image space rather than volumetric space, making it less accurate for volumetric data where intensity integration and anisotropy weighting are critical.

### [FA-based Pruning](3236-fa-based-pruning.md)
FA-based pruning reduces model size and rendering cost by omitting voxels with low importance scores, leveraging the observation that a small percentage of voxels contribute the majority of rendering importance [Parent]. It sets itself apart from intensity-only or scale-only heuristics by incorporating Feature Anisotropy (FA) cues, which yields superior reconstruction quality when combined with intensity and scale.

### [Importance-Driven Volume Rendering](7632-importance-driven-volume-rendering.md)
Importance-Driven Volume Rendering (IDVR) is a technique that assigns an importance value to volumetric features to dynamically adjust their visual sparseness (opacity/density) based on viewpoint-dependent occlusion, ensuring important features remain visible while less important ones are suppressed. It solves the problem of volume object occlusion by cutting or suppressing transparency of less important parts if they are in front of more important objects, allowing users to view hidden but important objects regardless of view direction. Unlike traditional techniques that classify features independently from the viewpoint using constant optical properties, IDVR uses multiple levels of sparseness with smooth transitions that vary continuously within a single feature depending on viewpoint and occlusion. It differs from standard DVR because standard DVR combines color/opacity values regardless of importance, meaning high-importance hidden objects cannot be visualized entirely.

### [Nonuniform Compression on Grid Models](11898-nonuniform-compression-on-grid-models.md)
A compression strategy that applies different down-sampling scales to volume density ($V_\sigma$) and color features ($V_c$) to reduce storage while preserving rendering quality, based on the insight that density is more critical for model ability than color features.

### Structure-Aware Pruning
Structure-Aware Pruning is a pruning strategy that computes a structure-aware importance score for each ellipsoid to remove redundant primitives while preserving structural fidelity [S3, unofficial]. It differs from view-dependent scoring by operating in volumetric space and integrating intensity, voxel coverage, and fractional anisotropy [S3, unofficial]. A voxel is considered covered if the primitive's contribution at its center exceeds $\epsilon = 1/255$. The score is $score_i = |\Omega_i| \cdot I_i \cdot FA_i$. Primitives are ranked by score and removed according to a preset ratio [S3, unofficial].

### Volume Thinning
Volume Thinning is a scheme that reduces volume visibility by sorting voxels according to gradient and curvature magnitudes, dissolving iso-surfaces gradually [S4, unofficial]. It sorts voxels first by gradient magnitude, then by curvature magnitude. Reducing sparseness according to gradient magnitude reduces the volume to fewer strong iso-surfaces. As few iso-surfaces remain, reduction proceeds according to curvature magnitude, gradually dissolving iso-surfaces so that only few high curvature areas remain in the most sparse representation [S4, unofficial].

### Narrow Area Filtering
Narrow Area Filtering is a particle reduction technique applied to AMR volume data to optimize rendering for XR devices by removing redundant particles based on density and depth [S5]. It iterates through each leaf node in the Octree. If particle count $n \le T$ and depth $l$ equals max depth $D$, it removes 80% of particles. If $l = D-1$, it removes 60%. If $c \ge 0.5 \times T$, it removes 50% [S5]. On AR devices (iPad), up to 20 iterations are allowed, reducing 48,764 particles to 4,649. On MR devices (HoloLens 2), up to 40 iterations are tested, reducing to 3,459 particles after 30 iterations, with diminishing FPS returns thereafter [S5].

## Operating characteristics
*   **Time Complexity:** The pruning process is $O(N \log N)$ due to sorting voxels by importance score, where $N$ is the number of voxels. Ray casting for importance calculation is $O(R \cdot S)$, where $R$ is the number of rays and $S$ is the number of samples per ray.
*   **Memory Complexity:** Reduces memory usage by the fraction of pruned voxels. The importance score map requires $O(N)$ additional memory temporarily.
*   **Performance:** Can significantly reduce model size (e.g., to 1 MB as cited in [S1]) with minimal quality loss if the pruning threshold is chosen correctly.
*   **Strengths:** Adaptive to scene content; general across different scenes or base models [S1].
*   **Limitations:** Requires a training phase or pre-computation of importance scores; not suitable for dynamic scenes without re-pruning.

## Background
Voxel pruning originated as a method to compress volumetric radiance fields. It was introduced in the context of compressing NeRF-based representations to reduce model size while maintaining rendering quality [S1]. The technique leverages the sparsity of natural scenes, where most voxels contribute little to the final image. It has been adapted for various volumetric representations, including neural feature volumes [S2] and 3D Gaussian/Ellipsoid splatting [S3, unofficial].

## Sources
[S1] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
[S2] NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00537, fetched 2026-10-26
[S3] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
[S4] (PDF) Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
[S5] Visualization of AMR Volume Data for Development of Extended Reality Realistic Content, https://doi.org/10.15701/kcgs.2023.29.3.105, fetched 2026-10-26
