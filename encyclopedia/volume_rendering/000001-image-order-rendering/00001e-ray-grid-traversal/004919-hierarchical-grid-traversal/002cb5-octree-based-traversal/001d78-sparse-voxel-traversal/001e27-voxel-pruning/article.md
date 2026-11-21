# Voxel Pruning

Also called: Importance-based Voxel Pruning

## Overview
Voxel Pruning is a technique to reduce model size and rendering cost by omitting voxels with low importance scores, based on the observation that a small percentage of voxels contribute the majority of rendering importance [S1]. It solves the problem of excessive memory usage in volumetric representations by removing non-essential voxels that contain no significant scene content [S1, S2]. It sets itself apart from intensity-only heuristics by incorporating geometric cues (such as fractional anisotropy) or cumulative ray contributions to determine voxel significance [S1][S3, unofficial].

## How it works
The method generally follows a two-phase approach: **Scoring** (calculating importance) and **Pruning** (removing low-importance voxels) [G]. Two primary scoring strategies are documented: Image-space accumulation [S1] and Volume-density sampling [S2].

### Inputs and Outputs
*   **Inputs:**
    *   A volumetric grid or set of primitives (voxels, ellipsoids, or particles) [S1][S3, unofficial][S4].
    *   For Image-space scoring: A set of training views (images) [S1].
    *   For Volume-density scoring: A radiance field or density predictor capable of evaluating density $\sigma$ at arbitrary points [S2].
    *   Hyperparameters: Pruning ratio $\beta_p$ [S1] or density threshold $\gamma$ [S2].
*   **Outputs:**
    *   A sparse set of voxels/primitives retained for rendering [G].
    *   Omitted voxels are removed from the data structure, including their density and color features [S1].

### Data Structures
*   **Voxel Grid:** Typically a 3D array or hash map storing density $\sigma$, color $C$, and importance score $I$ [G].
*   **Primitive List:** For ellipsoid-based variants, a list of primitives with center position, covariance matrix $\Sigma$, intensity $I$, and computed score [S3, unofficial].
*   **Particle List:** For AMR data, a list of particles with position, color, size, and brightness [S4].

### Algorithm: Image-Space Importance Scoring (S1)
This method assigns importance based on ray contributions in image space [S1].

1.  **Ray Shooting:** Shoot a batch of rays through the training views [S1].
2.  **Point Importance Calculation:** For each sampling point $\mathbf{x}_i$ on a ray, calculate its importance score $I_i$:
    $$I_i = T_i \cdot \alpha_i$$
    Where:
    *   $T_i$ is the accumulated transmittance up to point $i$ [S1].
    *   $\alpha_i$ is the ray termination probability at point $i$ [S1].
3.  **Voxel Score Assignment:** Assign $I_i$ to neighboring voxels $\mathbf{v}_l$ proportionally to their distance. The importance score $I_l$ for voxel $\mathbf{v}_l$ is:
    $$I_l = \sum_{\mathbf{x}_i \in \mathcal{N}_l} (1 - |\mathbf{v}_l - \mathbf{x}_i|) \cdot I_i$$
    Where:
    *   $\mathcal{N}_l$ is the set of sampling points within the neighborhood of $\mathbf{v}_l$ [S1].
    *   $|\mathbf{v}_l - \mathbf{x}_i| \le 1$ is the distance constraint [S1].
    *   The term $(1 - |\mathbf{v}_l - \mathbf{x}_i|)$ acts as a linear distance weight [S1].
4.  **Threshold Selection:** Compute the cumulative score rate $\theta(\theta)$ for a threshold $\theta$:
    $$\theta(\theta) = \frac{\sum I_l \cdot \mathbf{1}\{I_l < \theta\}}{\sum I_l}$$
    Select the pruning threshold $\theta_p$ using the quantile function:
    $$\theta_p = F^{-1}(\beta_p)$$
    Where $\beta_p$ is a hyperparameter representing the total amount of importance to be pruned [S1].
5.  **Pruning:** Sort voxels by ascending $I_l$. Omit all voxels where $I_l < \theta_p$ from the grid model [S1].

### Algorithm: Volume-Density Pruning (S2)
This method prunes based on local density estimates, suitable for neural feature volumes [S2].

1.  **Sampling:** For each voxel $V$, sample $k$ uniformly distributed points $\{v_i\}_{i=1}^k$ inside $V$ [S2].
2.  **Density Prediction:** Predict the density $\sigma(v_i)$ for each point $v_i$ using the radiance field [S2].
3.  **Transmittance Check:** Compute $\exp(-\sigma(v_i))$ for each point [S2].
4.  **Minimization:** Find the minimum value:
    $$m = \min_{i=1\dots k} \exp(-\sigma(v_i))$$
5.  **Pruning Decision:** If $m > \gamma$, prune voxel $V$ [S2].
    Where $\gamma$ is a pruning threshold [S2].
6.  **Execution:** Perform this step during later training phases or inference once the global feature volume $V^g_t$ is obtained [S2].

### Algorithm: Structure-Aware Pruning (S3)
Used for 3D Ellipsoid representations [S3, unofficial].

1.  **Coverage Check:** A voxel $v$ is considered covered by ellipsoid $i$ if the primitive yields a non-negligible contribution at its center:
    $$\tilde{P}_i(v) \ge \epsilon, \quad \epsilon = 1/255$$
2.  **Score Calculation:** Compute the importance score $score_i$ for each ellipsoid:
    $$score_i = |\Omega_i| \cdot I_i \cdot FA_i$$
    Where:
    *   $|\Omega_i|$ is the voxel coverage (number of covered voxels) [S3, unofficial].
    *   $I_i$ is the learned center intensity [S3, unofficial].
    *   $FA_i$ is the Fractional Anisotropy [S3, unofficial].
3.  **FA Calculation:** Compute $FA_i$ from eigenvalues $(\lambda_{1,i}, \lambda_{2,i}, \lambda_{3,i})$ of covariance matrix $\Sigma_i$:
    $$\text{FA}_i = \sqrt{\frac{1}{2}} \frac{\sqrt{(\lambda_{1,i} - \lambda_{2,i})^2 + (\lambda_{2,i} - \lambda_{3,i})^2 + (\lambda_{3,i} - \lambda_{1,i})^2}}{\sqrt{\lambda_{1,i}^2 + \lambda_{2,i}^2 + \lambda_{3,i}^2}}$$
4.  **Ranking and Removal:** Rank primitives by $score_i$. Remove the lowest-scoring primitives according to a preset pruning ratio [S3, unofficial].

### Pseudocode: General Voxel Pruning (Image-Space)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_pruning_image_space(voxels, training_views, beta_p):
    """
    Prunes voxels based on accumulated ray importance scores.
    """
    # Step 1: Initialize voxel importance scores
    voxel_importance = {v.id: 0.0 for v in voxels}
    
    # Step 2: Shoot rays and accumulate importance
    for view in training_views:
        # Standard ray generation for a view
        rays = [(view.origin + t * view.direction) for t in range(view.num_samples)]
        for ray in rays:
            # Standard volume rendering to get sampling points
            points, transmittances, alphas = volume_render_ray(ray)
            
            for i, point in enumerate(points):
                T_i = transmittances[i]
                alpha_i = alphas[i]
                I_i = T_i * alpha_i
                
                # Find neighboring voxels within distance 1.0
                neighbors = get_neighboring_voxels(point, voxels)
                
                for v_l in neighbors:
                    dist = distance(v_l.center, point)
                    if dist <= 1.0:
                        weight = 1.0 - dist
                        voxel_importance[v_l.id] += weight * I_i

    # Step 3: Determine threshold using quantile function
    scores = list(voxel_importance.values())
    total_importance = sum(scores)
    if total_importance == 0:
        return voxels, []

    sorted_scores = sorted(scores)
    target_pruned_mass = beta_p * total_importance
    current_mass = 0.0
    threshold = 0.0
    
    for s in sorted_scores:
        if current_mass + s <= target_pruned_mass:
            current_mass += s
            threshold = s
        else:
            break
            
    # Step 4: Prune
    pruned_voxels = []
    retained_voxels = []
    for v in voxels:
        if voxel_importance[v.id] < threshold:
            pruned_voxels.append(v)
        else:
            retained_voxels.append(v)
            
    return retained_voxels, pruned_voxels

def volume_render_ray(ray):
    points = []
    transmittances = []
    alphas = []
    T = 1.0
    for step in ray.steps:
        sigma = query_density(step.position)
        delta = step.distance
        alpha = 1.0 - math.exp(-sigma * delta)
        points.append(step.position)
        transmittances.append(T)
        alphas.append(alpha)
        T *= (1.0 - alpha)
    return points, transmittances, alphas

def get_neighboring_voxels(point, voxels):
    # Returns voxels within distance 1.0 of point
    return [v for v in voxels if distance(v.center, point) <= 1.0]
```

### Edge Cases and Pitfalls
*   **Boundary Conditions:** The neighborhood constraint $|\mathbf{v}_l - \mathbf{x}_i| \le 1$ must be strictly enforced to prevent "bleeding" of importance across voxel boundaries [S1].
*   **Degenerate Input:** If all voxels have zero density (empty scene), the cumulative score rate is undefined. A guard clause should handle empty scenes by retaining all or none based on policy [G].
*   **Threshold Sensitivity:** The choice of $\beta_p$ [S1] or $\gamma$ [S2] significantly impacts quality. Too aggressive pruning causes visible artifacts; too conservative yields minimal size reduction [S1, S2].

## Subtopics

### [LightGaussian Pruning Scoring](3213-lightgaussian-pruning-scoring.md)
LightGaussian Pruning Scoring is a 2D projection-based method for ranking 3D Gaussian primitives to determine which ones to prune, based on accumulated ray contributions across multiple synthesized views. It differs from volume-based scoring methods by operating in image space rather than volumetric space, making it less accurate for volumetric data where intensity integration and anisotropy weighting are critical.

### [FA-based Pruning](3236-fa-based-pruning.md)
FA-based pruning reduces model size and rendering cost by omitting voxels with low importance scores, leveraging the observation that a small percentage of voxels contribute the majority of rendering importance [G]. It sets itself apart from intensity-only or scale-only heuristics by incorporating Feature Anisotropy (FA) cues, which yields superior reconstruction quality when combined with intensity and scale.

### [Importance-Driven Volume Rendering](7632-importance-driven-volume-rendering.md)
Importance-Driven Volume Rendering (IDVR) is a technique that assigns an importance value to volumetric features to dynamically adjust their visual sparseness (opacity/density) based on viewpoint-dependent occlusion, ensuring important features remain visible while less important ones are suppressed. It solves the problem of volume object occlusion by cutting or suppressing transparency of less important parts if they are in front of more important objects, allowing users to view hidden but important objects regardless of view direction. Unlike traditional techniques that classify features independently from the viewpoint using constant optical properties, IDVR uses multiple levels of sparseness with smooth transitions that vary continuously within a single feature depending on viewpoint and occlusion. It differs from standard DVR because standard DVR combines color/opacity values regardless of importance, meaning high-importance hidden objects cannot be visualized entirely.

### [Volume Thinning](7633-volume-thinning.md)
Volume Thinning is a sparseness control technique that reduces the visibility of a volume by progressively dissolving voxels based on their geometric importance, specifically gradient magnitude and surface curvature. It solves the problem of rendering large volumetric datasets interactively by allowing the user to trade visual fidelity for performance, reducing the volume to a wireframe-like structure of strong iso-surfaces and high-curvature lines. Unlike opacity modulation which fades the entire object uniformly, Volume Thinning selectively removes voxels, preserving the structural integrity of complex features.

### [Nonuniform Compression on Grid Models](11898-nonuniform-compression-on-grid-models.md)
Nonuniform Compression on Grid Models is a storage optimization technique for explicit volume rendering grids that reduces model size by applying different down-sampling scales to volume density and color features, based on the insight that density is more critical for rendering quality than color. It sets itself apart from uniform compression by asymmetrically compressing attributes and combining low-resolution grids with a sparse set of high-fidelity "important" voxels to preserve high-frequency details. This method builds on [Voxel Pruning](1693-voxel-pruning.md) by using importance scores to identify which voxels require full resolution storage.

### Structure-Aware Pruning
Structure-Aware Pruning is a pruning strategy that computes a structure-aware importance score for each ellipsoid to remove redundant primitives while preserving structural fidelity [S3, unofficial]. It differs from view-dependent scoring by operating in volumetric space and integrating intensity, voxel coverage, and fractional anisotropy. A voxel is covered if the primitive's contribution at its center exceeds $\epsilon = 1/255$. The score is $score_i = |\Omega_i| \cdot I_i \cdot FA_i$. Primitives are ranked by score and lowest-scoring ones are removed [S3, unofficial].

### Narrow Area Filtering
Narrow Area Filtering is a particle reduction technique applied to AMR volume data to optimize rendering for XR devices by removing redundant particles based on density and depth [S4]. It iterates through Octree leaf nodes: if particle count $n \le T$ and depth $l=D$, remove 80%; if $l=D-1$, remove 60%; if $c \ge 0.5 T$, remove 50%. Tested on iPad (AR) and HoloLens 2 (MR), reducing 48,764 particles to 4,649 (AR, 20 iters) and 3,459 (MR, 30 iters) [S4].

## Operating characteristics
*   **Time Complexity:** Scoring requires $O(N_r \cdot N_s)$ for $N_r$ rays and $N_s$ samples per ray. Sorting and pruning are $O(N_v \log N_v)$ for $N_v$ voxels. Overall complexity is dominated by ray shooting and density evaluation [S1, S2].
*   **Memory Complexity:** Reduces memory usage by the pruning ratio. The exact reduction depends on the importance distribution and threshold selection [S1].
*   **Measured Performance:** In AR devices (iPad), filtering reduced particles from 48,764 to 4,649, achieving ~47 FPS [S4]. In MR devices (HoloLens 2), 30 iterations reduced particles to 3,459, with diminishing FPS returns after 30 iterations [S4].
*   **Strengths:** Significant model size reduction; adaptive to scene content; preserves visual quality of important features [S1, S2].
*   **Limitations:** Requires pre-computation or iterative training; threshold selection can be sensitive; may introduce artifacts if pruning is too aggressive [S1, S2].
*   **When to Use:** When memory or bandwidth is constrained; for static or slowly changing scenes where pre-computation is feasible; for neural radiance fields or grid-based representations [S1, S2].
*   **When Not to Use:** For highly dynamic scenes requiring real-time pruning; when uniform quality is required across the entire volume [S2].

## Background
Voxel Pruning originated as a method to compress volumetric radiance fields, notably in the context of Deep Voxel Grids (DVGO) and NeRF compression [S1]. It was introduced to address the large model sizes of these representations by leveraging the observation that voxel importance follows a long-tail distribution [S1]. The technique has been extended to include structure-aware scoring for ellipsoids [S3, unofficial] and particle filtering for XR applications [S4]. It is widely used in novel view synthesis, volume visualization, and compressed neural rendering [S1, S2, S4].

## Sources
[S1] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
[S2] NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00537, fetched 2026-10-26
[S3] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
[S4] Visualization of AMR Volume Data for Development of Extended Reality Realistic Content, https://doi.org/10.15701/kcgs.2023.29.3.105, fetched 2026-10-26
