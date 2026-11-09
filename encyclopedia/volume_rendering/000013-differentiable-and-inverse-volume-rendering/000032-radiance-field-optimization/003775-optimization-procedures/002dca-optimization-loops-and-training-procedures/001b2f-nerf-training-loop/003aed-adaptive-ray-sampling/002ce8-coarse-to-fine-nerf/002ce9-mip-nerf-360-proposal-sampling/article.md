# Mip-NeRF 360 Proposal Sampling

Also called: Modified Resampling, Proposal Sampling, midpoint resampling

## Overview
Mip-NeRF 360 Proposal Sampling is a hierarchical ray sampling strategy that refines sample intervals based on the opacity distribution of a coarse neural radiance field (NeRF) to reduce aliasing and improve geometric accuracy [S1]. It distinguishes itself from standard hierarchical sampling by using a "midpoint resampling" technique that defines fine intervals via the midpoints of sampled points, rather than using the sampled points directly as interval boundaries [S1]. This method builds on [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by introducing a specific geometric refinement step that ensures fine samples are centered within regions of high opacity, thereby mitigating the "erosion" of the coarse histogram [S1].

## How it works
The method operates in two distinct phases: a coarse bounding phase and a fine refinement phase. It relies on a coarse NeRF to estimate the probability density of scene geometry along a ray, which is then used to generate a denser set of samples for a fine NeRF.

### Inputs and Outputs
*   **Inputs**:
    *   Ray parameters: origin $\mathbf{o}$, direction $\mathbf{d}$, near plane $t_n$, far plane $t_f$ [S1].
    *   Coarse NeRF model: $\mathbf{F}_{coarse}(\mathbf{x}, \mathbf{d})$ which outputs density $\sigma$ and color $\mathbf{c}$ [S1].
    *   Number of coarse samples $N_c$ [S1].
    *   Number of fine samples $N_f$ [S1].
*   **Outputs**:
    *   A set of $N_f$ sample points along the ray for the fine NeRF [S1].
    *   The final rendered pixel color and depth [S2].

### Data Structures
The core data structure is a discrete histogram representing the probability mass of the scene along the ray.
*   **Coarse Histogram**: A vector of size $N_c$. Each element $i$ corresponds to the integral of density over the $i$-th coarse interval $[t_i, t_{i+1}]$.
*   **Sample Points**: A sorted list of $N_c$ depth values $t^{(1)}, t^{(2)}, \dots, t^{(N_c)}$ sampled from the coarse histogram.
*   **Fine Intervals**: A set of $N_f - 1$ intervals defined by endpoints derived from the coarse samples.

### Algorithm Steps
1.  **Coarse Sampling**:
    *   Divide the ray segment $[t_n, t_f]$ into $N_c$ equal-length intervals.
    *   Evaluate the coarse NeRF at the center of each interval to obtain density $\sigma_i$.
    *   Compute the probability mass for each interval $i$:
        $$ p_i = \frac{\sigma_i \Delta t_i}{\sum_{j=1}^{N_c} \sigma_j \Delta t_j} $$
        where $\Delta t_i$ is the length of the interval [S1].
    *   Sample $N_c$ points $t^{(1)}, \dots, t^{(N_c)}$ from this discrete distribution using inverse transform sampling or stratified sampling. Sort these points such that $t^{(1)} < t^{(2)} < \dots < t^{(N_c)}$ [S1].

2.  **Midpoint Resampling (Modified Resampling)**:
    *   Standard hierarchical sampling uses the sampled points $t^{(i)}$ directly as the boundaries of the fine intervals. Mip-NeRF 360 modifies this to reduce aliasing [S1].
    *   Define the endpoints of the $N_f - 1$ fine intervals using the midpoints of adjacent coarse samples. Let the fine interval endpoints be $u_1, u_2, \dots, u_{N_f}$.
    *   For $k = 1$ to $N_c - 1$:
        $$ u_k = \frac{t^{(k)} + t^{(k+1)}}{2} $$
    *   Handle boundary conditions by reflecting the first and last coarse samples around the first and last endpoints of the ray segment $[t_n, t_f]$ [S1].
        *   Let $t_{start} = t_n$ and $t_{end} = t_f$.
        *   Reflect the first sample $t^{(1)}$ around $t_{start}$:
            $$ u_0 = t_{start} - (t^{(1)} - t_{start}) = 2t_{start} - t^{(1)} $$
        *   Reflect the last sample $t^{(N_c)}$ around $t_{end}$:
            $$ u_{N_f-1} = t_{end} + (t_{end} - t^{(N_c)}) = 2t_{end} - t^{(N_c)} $$
        *   Note: The exact mapping of reflected points to interval endpoints depends on the specific implementation of $N_f$ relative to $N_c$. Typically, the $N_c$ samples define $N_c-1$ internal midpoints, and the boundaries are handled by the reflection logic to ensure coverage of the full $[t_n, t_f]$ range [S1]. The resulting set of endpoints $\{u_0, \dots, u_{N_f-1}\}$ defines $N_f-1$ intervals.
    *   Sample points uniformly or stratified within these new fine intervals $[u_k, u_{k+1}]$ [S1].

3.  **Fine Rendering**:
    *   Evaluate the fine NeRF at the new sample points.
    *   Render the image using standard volume rendering integration [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mip_nerf_360_proposal_sampling(ray_origin, ray_direction, t_near, t_far, 
                                   coarse_nerf, fine_nerf, N_coarse, N_fine):
    """
    Implements Mip-NeRF 360 Proposal Sampling.
    
    Args:
        ray_origin: (3,) array, ray origin.
        ray_direction: (3,) array, normalized ray direction.
        t_near: float, near plane distance.
        t_far: float, far plane distance.
        coarse_nerf: function, takes (points, directions) -> (sigma, color).
        fine_nerf: function, takes (points, directions) -> (sigma, color).
        N_coarse: int, number of coarse samples.
        N_fine: int, number of fine samples.
        
    Returns:
        rendered_color: (3,) array, final pixel color.
    """
    
    # Step 1: Coarse Sampling
    # Create N_coarse equal intervals
    t_coarse = np.linspace(t_near, t_far, N_coarse + 1)
    centers_coarse = (t_coarse[:-1] + t_coarse[1:]) / 2
    points_coarse = ray_origin + ray_direction * centers_coarse[:, None]
    
    # Evaluate coarse NeRF
    sigma_coarse, _ = coarse_nerf(points_coarse, ray_direction)
    
    # Compute interval lengths
    delta_t = t_coarse[1:] - t_coarse[:-1]
    
    # Compute probability mass for each interval
    # p_i = sigma_i * delta_t_i / sum(sigma_j * delta_t_j)
    weights = sigma_coarse * delta_t
    total_weight = np.sum(weights)
    if total_weight == 0:
        # Degenerate case: empty scene
        return np.zeros(3)
    probs = weights / total_weight
    
    # Sample N_coarse points from the discrete distribution
    # Using stratified sampling for better coverage
    u = np.random.uniform(0, 1, N_coarse)
    cum_probs = np.cumsum(probs)
    # Map uniform samples to indices
    indices = np.searchsorted(cum_probs, u)
    # Clamp indices to valid range
    indices = np.clip(indices, 0, N_coarse - 1)
    
    # Sample points uniformly within the selected intervals
    t_sampled = []
    for idx in indices:
        t_min = t_coarse[idx]
        t_max = t_coarse[idx + 1]
        t_val = np.random.uniform(t_min, t_max)
        t_sampled.append(t_val)
    
    t_sampled = np.sort(np.array(t_sampled))
    
    # Step 2: Midpoint Resampling
    # Define fine interval endpoints using midpoints of adjacent coarse samples
    # and reflection for boundaries
    
    # Internal endpoints: midpoints between adjacent sampled points
    if N_coarse > 1:
        midpoints = (t_sampled[:-1] + t_sampled[1:]) / 2
    else:
        midpoints = np.array([])
        
    # Boundary handling: reflect first and last samples
    # Reflect first sample around t_near
    t_reflected_start = 2 * t_near - t_sampled[0]
    # Reflect last sample around t_far
    t_reflected_end = 2 * t_far - t_sampled[-1]
    
    # Combine boundaries and midpoints to form all endpoints
    # The number of fine intervals is N_fine - 1.
    # We have N_coarse - 1 midpoints.
    # We need to distribute the N_fine - 1 intervals.
    # The standard approach in Mip-NeRF 360 is to use the midpoints as the 
    # internal boundaries and the reflected points as the outer boundaries.
    
    # Construct the full list of endpoints for N_fine intervals
    # We need N_fine - 1 internal boundaries.
    # If N_fine > N_coarse, we might need to interpolate or duplicate.
    # However, the core innovation is the midpoint definition.
    # Let's assume N_fine is roughly proportional to N_coarse.
    
    # A robust implementation:
    # 1. Start with the reflected boundaries.
    # 2. Insert the midpoints.
    # 3. If more intervals are needed, subdivide existing intervals.
    
    endpoints = [t_reflected_start]
    endpoints.extend(midpoints)
    endpoints.append(t_reflected_end)
    
    # Sort endpoints (they should already be sorted due to logic)
    endpoints = np.sort(np.array(endpoints))
    
    # If we have more endpoints than needed for N_fine intervals, 
    # we might need to select a subset or interpolate. 
    # The paper states "sample n sorted values... use midpoints... as endpoints".
    # This implies the number of fine intervals is determined by the number of coarse samples.
    # Specifically, N_coarse samples create N_coarse - 1 midpoints.
    # Plus 2 boundaries = N_coarse + 1 endpoints -> N_coarse intervals.
    # If N_fine != N_coarse, we must adjust.
    
    # For simplicity, if N_fine == N_coarse, we use all endpoints.
    # If N_fine > N_coarse, we can subdivide.
    # If N_fine < N_coarse, we can merge.
    
    # Let's assume the standard case where N_fine is a multiple or similar to N_coarse.
    # The key is that the intervals are defined by these specific endpoints.
    
    # Generate sample points within the defined intervals
    t_fine = []
    num_intervals = len(endpoints) - 1
    
    # If we need exactly N_fine samples, and we have num_intervals intervals,
    # we can sample uniformly within each interval.
    # If num_intervals != N_fine - 1, we need to adjust.
    
    # The paper's "modified resampling" primarily affects the *definition* of intervals.
    # Let's assume we create N_fine intervals by subdividing the coarse intervals 
    # based on the midpoints.
    
    # Simplified: Use the midpoints as the primary structure.
    # If N_fine is large, we can sample multiple points per interval.
    
    # For the purpose of this pseudocode, we will generate N_fine points 
    # by stratified sampling within the intervals defined by the endpoints.
    
    # If the number of endpoints doesn't match N_fine - 1, we interpolate.
    # But the core method is the midpoint definition.
    
    # Let's stick to the exact description: 
    # "sample n sorted values... use midpoints... as endpoints"
    # This creates N_coarse - 1 internal intervals.
    # Plus the two boundary intervals defined by reflection.
    # Total intervals = N_coarse + 1? No, N_coarse samples create N_coarse-1 midpoints.
    # Boundaries are t_near and t_far? No, reflected points.
    
    # Let's assume the fine sampling uses the intervals defined by:
    # [t_reflected_start, midpoint_1], [midpoint_1, midpoint_2], ..., [midpoint_last, t_reflected_end]
    # This gives N_coarse intervals.
    
    # If N_fine is different, we can sample multiple points per interval.
    
    interval_starts = endpoints[:-1]
    interval_ends = endpoints[1:]
    
    # Stratified sampling within each interval
    for i in range(len(interval_starts)):
        t_min = interval_starts[i]
        t_max = interval_ends[i]
        # Distribute N_fine samples across intervals proportionally to length?
        # Or uniformly? The paper doesn't specify the distribution across intervals,
        # only the interval definition.
        # Let's assume uniform distribution of samples across the ray length.
        pass
        
    # To keep it simple and compliant with the "midpoint" definition:
    # We will sample N_fine points uniformly within the intervals defined by the endpoints.
    
    # Recalculate endpoints if N_fine != N_coarse + 1
    # This is a complex edge case not fully detailed in the notes.
    # We will assume N_fine is chosen such that it fits the structure.
    
    # Generate points
    t_fine_points = []
    for i in range(len(interval_starts)):
        t_min = interval_starts[i]
        t_max = interval_ends[i]
        # Sample uniformly in this interval
        t_val = np.random.uniform(t_min, t_max)
        t_fine_points.append(t_val)
        
    # If we need more points, repeat or subdivide.
    # If we need fewer, select a subset.
    
    t_fine_points = np.sort(np.array(t_fine_points))
    
    # Ensure we have N_fine points
    if len(t_fine_points) < N_fine:
        # Subdivide intervals
        while len(t_fine_points) < N_fine:
            # Pick an interval and split it
            idx = np.random.randint(len(interval_starts))
            t_min = interval_starts[idx]
            t_max = interval_ends[idx]
            t_new = np.random.uniform(t_min, t_max)
            t_fine_points = np.append(t_fine_points, t_new)
            t_fine_points = np.sort(t_fine_points)
    elif len(t_fine_points) > N_fine:
        t_fine_points = t_fine_points[:N_fine]
        
    t_fine_points = np.sort(t_fine_points)
    
    # Step 3: Fine Rendering
    points_fine = ray_origin + ray_direction * t_fine_points[:, None]
    sigma_fine, color_fine = fine_nerf(points_fine, ray_direction)
    
    # Volume Rendering
    # Alpha = 1 - exp(-sigma * delta_t)
    delta_t_fine = np.diff(np.concatenate(([t_near], t_fine_points, [t_far])))
    alpha = 1.0 - np.exp(-sigma_fine * delta_t_fine[:, None])
    
    # Transmittance
    T = np.cumprod(1.0 - alpha, axis=0)
    T = np.concatenate([np.ones((1, 3)), T[:-1]], axis=0)
    
    # Color
    rendered_color = np.sum(T * alpha * color_fine, axis=0)
    
    return rendered_color
```

### Parameters
*   **$N_c$ (Coarse Samples)**: Typically 64 or 128 [S1].
*   **$N_f$ (Fine Samples)**: Typically 128 or 256 [S1].
*   **Truncation**: In Zip-NeRF implementations, the proposal NGP hierarchy is truncated at maximum grid sizes of 512 and 2048 [S2].
*   **Channels**: Proposal NGPs use a single channel of features (predicting density only) [S2].

### Edge Cases and Pitfalls
*   **Empty Scene**: If the coarse histogram has zero total weight, the sampling fails. A guard clause should return a default color (e.g., background) [S1].
*   **Boundary Conditions**: The reflection of the first and last samples is critical to ensure that the fine intervals cover the entire $[t_n, t_f]$ range without gaps or overlaps near the boundaries [S1].
*   **Aliasing**: Using sampled points directly as endpoints (standard hierarchical sampling) can "erode" the coarse histogram, leading to aliasing. The midpoint resampling mitigates this [S1].

## Subtopics

### [Dynamic Optimal Sampling Layer](8681-dynamic-optimal-sampling-layer.md)
The Dynamic Optimal Sampling Layer is a three-stage sampling strategy (coarse, optimal, fine) that inserts an optimal sampling layer between coarse and fine sampling to adaptively adjust the number and locations of new sample points based on priorities from the previous stage, targeting high-weight regions. It builds on [Mip-NeRF 360 Proposal Sampling](5408-mip-nerf-360-proposal-sampling.md) by replacing uniform or fixed-density refinement with a priority-driven allocation mechanism that uses dilation to expand coverage in high-density areas, thereby improving sampling efficiency and rendering quality in sparse reconstruction scenarios.

### [NeRF-Casting Optimization and Representation](11489-nerf-casting-optimization-and-representation.md)
NeRF-Casting is a differentiable volume rendering method that extends NeRF with explicit reflection modeling to handle glossy surfaces and consistent view-dependent appearance. It solves the problem of inaccurate specular highlights and inconsistent reflections in standard neural radiance fields by introducing a reflection-dependent color MLP and a modified proposal sampling scheme. It builds on [Mip-NeRF 360 Proposal Sampling](5408-mip-nerf-360-proposal-sampling.md) and [Zip-NeRF](https://arxiv.org/abs/2302.11423) by adapting their adaptive sampling and hash grid structures to support reflected rays.

## Operating characteristics
*   **Time Complexity**: The method adds an overhead of one additional NeRF evaluation pass (the coarse pass) per ray. The fine pass is then performed on a reduced set of intervals. The complexity is $O(N_c + N_f)$ per ray, where $N_c$ is the number of coarse samples and $N_f$ is the number of fine samples [S1, S2].
*   **Memory Complexity**: Requires storing the coarse histogram and the sampled points. The memory footprint is proportional to $N_c$ [S1].
*   **Performance**: The midpoint resampling technique qualitatively reduces aliasing with little quantitative effect on rendering quality metrics like PSNR [S1]. The use of truncated NGPs for proposal sampling improves performance by reducing the computational cost of the coarse pass [S2].
*   **Strengths**: Reduces aliasing artifacts; improves geometric accuracy by focusing samples on opaque regions; efficient due to truncated NGPs in proposal passes [S1, S2].
*   **Limitations**: Requires two passes (coarse and fine); the reflection logic for boundaries can be complex to implement correctly [S1].

## Background
Mip-NeRF 360 Proposal Sampling was introduced in the paper "Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields" [S1]. It was developed to address the issue of aliasing in neural radiance fields, particularly in unbounded scenes. The method builds on the original Mip-NeRF approach by introducing a modified resampling procedure that uses midpoints of coarse samples to define fine intervals [S1]. The technique has been adopted and extended by subsequent works such as Zip-NeRF, which uses truncated NGPs for the proposal sampling to improve performance [S2].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S3] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
