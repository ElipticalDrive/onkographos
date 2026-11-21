# Adaptive Primary Surface Sampling

Also called: APS, HashPoint Sampling

## Overview
Adaptive Primary Surface Sampling is a technique that accelerates neural rendering by adaptively sampling 3D points only on the primary surface (the first surface encountered by a camera ray), avoiding redundant sampling of multiple surfaces or empty space [S1]. It distinguishes itself from [Depth Map Sampling](11356-surface-based-sampling.md) by not requiring dense depth inputs, and from [Multi-Surface Sampling](11356-surface-based-sampling.md) by focusing exclusively on the front-most surface to balance precision and efficiency [S1]. It builds on [Surface-Based Sampling](11356-surface-based-sampling.md) by using a point cloud distribution to guide this selection rather than explicit depth maps [S1].

## How it works
**Inputs:**
- Camera parameters: optical center $\mathbf{o}$, ray direction $\mathbf{d}$, focal length $f$, pixel center $\mathbf{p}_o$ [S1].
- Point cloud data $\mathcal{P}$ stored in a Hashed-Point Searching structure (hash table mapping 2D image plane regions to 3D points) [S1].
- Hyperparameters: $K$ (number of neighbors for pseudo-UDF calculation), $\gamma$ (sampling range control), $\beta$ (density-dependent scaling), $r_{min}$ (minimum search radius at $t_{near}$), $t_{near}$, $t_{far}$ [S1].

**Outputs:**
- A set of definitive sample points $\{x_j^{sp}\}$ located on the primary surface along the ray [S1].
- Rendered color $c$ for the pixel [S1].

**Data Structures:**
- **Hashed-Point Searching Structure:** A hash table that maps 2D pixel coordinates (or regions on the image plane) to lists of 3D point cloud indices. This allows $O(1)$ lookup of points near a ray [S1].
- **Sample Point Candidate List:** A temporary list of projected points $x_j^{sp}$ derived from neighboring points [S1].

**Steps:**

1.  **Ray Cone Definition and Search Kernel Construction:**
    Define a ray cone from the optical center $\mathbf{o}$ through the pixel center $\mathbf{p}_o$. To mitigate holes caused by point cloud sparsity, expand the search kernel radius $r$ dynamically based on distance $t$ along the ray [S1]. The radius $r$ at distance $t$ is computed as:
    $$
    r(t) = \|\mathbf{p}_o - \mathbf{o}\|_2 \cdot \sqrt{\left(\sqrt{\|\mathbf{p}_o - \mathbf{o}\|_2^2 - f^2} - \ddot{r}\right)^2 + f^2}
    $$
    where $\ddot{r}$ is the base radius of the searching kernel [S1]. The magnitude of the searching kernel $s$ is deduced as $s = 2 \cdot \ddot{r} + 1$ [S1]. The search is performed by traversing pixels in the search kernel and looking up associated points in the hash table [S1].

2.  **Point Projection:**
    Identify point clouds near the camera ray using the Hashed-Point Searching structure. Project these neighboring points onto the camera ray. Each projected point becomes a sample point candidate $x_j^{sp}$, defined as:
    $$
    x_j^{sp} = \mathbf{o} + t_j \mathbf{d}
    $$
    where $t_j$ is the intersection parameter of the projection [S1].

3.  **Pseudo-UDF Calculation:**
    For each candidate $x_j^{sp}$, calculate the average distance $d_j$ to its $K$ nearest neighboring points in the point cloud $\mathcal{P}$. This distance is termed "pseudo-UDF" because it adheres to Unsigned Distance Function (UDF) principles but is not a strict UDF [S1]:
    $$
    d_j = \frac{1}{K} \sum_{i=1}^{K} \|x_j^{sp} - x_i^{pc}\|_2
    $$
    where $x_i^{pc}$ are the coordinates of the $K$ nearest neighbors [S1].

4.  **Confidence and Weight Calculation:**
    Transform the distance $d_j$ into a confidence value $\alpha_j$ using a Gaussian-like function dependent on point cloud density $\beta$ and sampling range $\gamma$ [S1]:
    $$
    \alpha_j = \gamma \exp \left( -\frac{d_j^2}{\beta_j^2} \right)
    $$
    Here, $\beta$ is a hyperparameter depending on the local density of the point cloud; a smaller $d_j$ (proximity to surface) necessitates a reduced $\beta$ to ensure distinct differentiation in confidence [S1].
    
    To ensure occlusion awareness (prioritizing points closer to the viewpoint), define the weight $w_j$ for each candidate:
    $$
    w_j = \alpha_j \prod_{k=1}^{j-1} (1 - \alpha_k)
    $$
    This formula ensures that if a candidate has high confidence ($\alpha$), it contributes significantly, while subsequent candidates are weighted by the transmittance of previous ones [S1].

5.  **Sample Selection:**
    Retain sample point candidates with $w_j > 0$. These candidates are located near the primary surface. The number of retained samples is adaptive, ranging from 0 to $n$, depending on the point cloud distribution and ray path [S1].

6.  **Rendering Integration:**
    Sample between $t_{near}$ and $t_{far}$ using the selected definitive sample points. Simulate radiance using volume density accumulation [S1]:
    $$
    c = \sum_{j=1}^{N} \tau_j \left( 1 - \exp(-\sigma_j \Delta_t) \right) c_j^{sp}
    $$
    where:
    - $N$ is the number of selected sample points [S1].
    - $\tau_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \Delta_k)$ is the volume transmittance [S1].
    - $\sigma_j$ is the density predicted by an MLP or estimated from point features for sample point $j$ [S1].
    - $c_j^{sp}$ is the color for sample point $j$ [S1].
    - $\Delta_t$ is the distance between adjacent sample points [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_primary_surface_sampling(ray_origin, ray_direction, camera_params, point_cloud_hash_table, K, gamma, beta, t_near, t_far):
    """
    ray_origin: 3D vector o
    ray_direction: 3D vector d
    camera_params: dict containing focal_length f, pixel_center p_o, optical_center o
    point_cloud_hash_table: Hash table mapping 2D regions to 3D points
    K: int, number of neighbors for pseudo-UDF
    gamma: float, sampling range control
    beta: float, density-dependent scaling
    t_near: float, near plane distance
    t_far: float, far plane distance
    """
    f = camera_params['focal_length']
    p_o = camera_params['pixel_center']
    o = camera_params['optical_center']
    
    # Step 1: Determine search radius at t_near (r_min)
    # Note: The formula in source uses d for distance from origin to pixel center
    d_po_o = np.linalg.norm(p_o - o)
    # Base radius r_tilde is derived from input hyperparameters or kernel size s
    # Assuming r_tilde is provided or calculated from kernel size s = 2 * r_tilde + 1
    # For implementation, we assume r_tilde is known or passed.
    # If not given, we use the dynamic radius formula for any t.
    
    sample_candidates = []
    
    # Step 2: Identify points near the ray using Hashed-Point Searching
    # This involves querying the hash table for points within the expanded ray cone.
    # The cone radius r(t) expands with t.
    # We iterate through relevant pixels in the search kernel and retrieve points.
    nearby_points = query_hash_table(point_cloud_hash_table, ray_origin, ray_direction, camera_params)
    
    # Project nearby points onto the ray to form candidates
    for point in nearby_points:
        # Project point onto ray direction
        # t_j = dot(point - o, d) / dot(d, d)
        # Assuming d is normalized, dot(d,d) = 1
        t_j = np.dot(point - o, ray_direction)
        
        # Only consider points within the rendering range
        if t_j < t_near or t_j > t_far:
            continue
            
        x_sp = o + t_j * ray_direction
        sample_candidates.append({'point': x_sp, 't': t_j, 'original_point': point})
        
    # Step 3 & 4: Calculate Pseudo-UDF and Weights
    weighted_samples = []
    
    # Sort candidates by distance from viewpoint (t) to ensure occlusion order
    sample_candidates.sort(key=lambda x: x['t'])
    
    for i, cand in enumerate(sample_candidates):
        x_sp = cand['point']
        
        # Find K nearest neighbors in the original point cloud
        # In a real implementation, this uses the hash table or a KD-tree
        neighbors = find_k_nearest_neighbors(point_cloud_hash_table, x_sp, K)
        
        # Calculate pseudo-UDF (average distance to K neighbors)
        dists = [np.linalg.norm(x_sp - n) for n in neighbors]
        d_j = np.mean(dists)
        
        # Calculate confidence alpha
        # beta might be local; here we use the passed beta parameter
        # If beta is density-dependent, it should be computed per point.
        # Source says "beta is a hyperparameter which depends on the density of the point cloud"
        # and "lesser dj necessitates a reduced beta".
        # For simplicity, we use the passed beta, but in practice beta_j might vary.
        alpha_j = gamma * np.exp(-(d_j**2) / (beta**2))
        
        # Calculate weight w_j
        # w_j = alpha_j * product(1 - alpha_k) for k < j
        # We need to accumulate the product of (1 - alpha) from previous steps
        if i == 0:
            transmittance_so_far = 1.0
        else:
            # Recalculate or store previous transmittance
            # To avoid O(N^2), we can maintain a running product
            # However, the formula is explicit. We assume we have access to previous alphas.
            # Let's assume we store previous alphas in a list or compute on the fly.
            # For O(N) efficiency, we maintain a running product.
            pass 
            
        # Efficient calculation of w_j
        # We need the product of (1 - alpha_k) for all k < i
        # Let's assume we have a variable `prev_transmittance` updated in the loop
        w_j = alpha_j * prev_transmittance
        
        if w_j > 0:
            weighted_samples.append({
                'point': x_sp,
                't': t_j,
                'alpha': alpha_j,
                'weight': w_j
            })
            
        # Update transmittance for next iteration
        # The term (1 - alpha_j) is the transmittance through this "layer"
        # Note: The weight formula uses product(1-alpha_k). 
        # So for the next step, we multiply current transmittance by (1 - alpha_j)
        prev_transmittance *= (1 - alpha_j)

    # Step 6: Render
    # Sort selected samples by t (they should already be sorted)
    weighted_samples.sort(key=lambda x: x['t'])
    
    color = 0.0
    current_transmittance = 1.0
    
    for i, sample in enumerate(weighted_samples):
        # Predict density sigma and color c_sp for this sample point
        # This depends on the specific neural rendering method (e.g., MLP)
        sigma_j = predict_density(sample['point'])
        c_sp_j = predict_color(sample['point'])
        
        # Distance to next sample or t_far
        if i + 1 < len(weighted_samples):
            delta_t = weighted_samples[i+1]['t'] - sample['t']
        else:
            delta_t = t_far - sample['t']
            
        # Volume rendering contribution
        # c = sum( tau_j * (1 - exp(-sigma_j * delta_t)) * c_sp_j )
        # tau_j is the transmittance arriving at sample j
        # In the loop, current_transmittance represents tau_j
        
        contribution = current_transmittance * (1 - np.exp(-sigma_j * delta_t)) * c_sp_j
        color += contribution
        
        # Update transmittance for the next sample
        current_transmittance *= np.exp(-sigma_j * delta_t)
        
        # Early termination if transmittance is negligible
        if current_transmittance < 1e-4:
            break
            
    return color
```

**Edge Cases and Pitfalls:**
- **Point Cloud Sparsity:** If no points fall within the ray cone, rendering holes may occur. This is mitigated by modifying the ray cone for broader coverage using a larger disc radius for the searching kernel [S1].
- **Beta Parameter:** The hyperparameter $\beta$ depends on point cloud density. A smaller distance $d_j$ (proximity to surface) necessitates a reduced $\beta$ to ensure distinct differentiation in confidence $\alpha_j$ across sample points [S1].
- **Zero Samples:** The number of sample points can range from 0 to $n$. If 0 points are selected, the pixel may remain black or require a fallback strategy (not specified in source, but implied by "holes" mitigation) [S1].

## Operating characteristics
- **Time Complexity:** The hash table lookup for point searching is $O(1)$ per query [S1]. The overall complexity depends on the number of points in the search kernel and the $K$-nearest neighbor search, which is typically efficient with spatial hashing.
- **Memory Complexity:** Requires storage for the point cloud and the hash table structure [S1].
- **Performance:** Demonstrates significant acceleration in rendering process with similar or better accuracy compared to traditional methods on synthetic, real, indoor, and outdoor datasets [S1].
- **Strengths:** Avoids redundant sampling on multiple surfaces (unlike Multi-Surface Sampling) and does not require dense depth maps (unlike Depth Map Sampling) [S1].
- **Limitations:** Performance depends on the quality and density of the input point cloud. Sparsity can lead to holes if not mitigated by the expanded search kernel [S1].

## Background
Adaptive Primary Surface Sampling was introduced in the paper "HashPoint: Accelerated Point Searching and Sampling for Neural Rendering" [S1]. It is part of the HashPoint method, which aims to accelerate neural rendering by optimizing point searching and sampling strategies. The method was validated on various benchmark datasets, including NeRF-Synthesis [S1].

## Sources
[S1] HashPoint: Accelerated Point Searching and Sampling for Neural Rendering, https://doi.org/10.1109/cvpr52733.2024.00427, fetched 2026-10-27
