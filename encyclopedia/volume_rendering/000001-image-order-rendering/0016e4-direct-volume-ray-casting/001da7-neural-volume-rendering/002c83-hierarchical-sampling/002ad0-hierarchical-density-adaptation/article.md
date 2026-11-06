# Hierarchical Density Adaptation

Also called: On-the-fly coarse sampling

## Overview
Hierarchical Density Adaptation is a sampling strategy for direct volume ray casting that generates a coarse density estimate from a dense feature grid on the fly to guide importance sampling for fine samples. It solves the problem of missing thin or high-frequency surface details when sampling directly from a dense grid, which can occur if coarse sample points fall between grid cells containing significant geometry. Unlike sibling methods that maintain a dedicated, pre-computed coarse grid or train a separate coarse network, this method derives the coarse density by convolving the existing dense density grid with a non-learnable kernel, thereby avoiding additional memory allocation for coarse structures while ensuring robust surface detection.

## How it works
The method operates in two stages: coarse sampling to estimate density distribution, and fine sampling to render color. It relies on a dense density feature grid $G_\sigma$ and a dense appearance feature grid $G_a$.

**Inputs and Outputs:**
*   **Input:** Dense density grid $G_\sigma$, dense appearance grid $G_a$, camera ray origin $o$, ray direction $\mathbf{d}$, number of coarse samples $N_c$, number of fine samples $N_f$.
*   **Output:** Pixel color $\hat{C}$.

**Data Structures:**
*   $G_\sigma$: A 3D grid storing volume density values $\sigma$. In the primary implementation (EgoNeRF), this is a balanced spherical grid where resolution increases exponentially along the radial direction [S1].
*   $K$: A non-learnable convolution kernel, specifically an average pooling kernel [S1].

**Steps:**

1.  **On-the-fly Coarse Density Estimation:**
    To prevent skipping important surface regions, the density at coarse sample locations is not read directly from $G_\sigma$. Instead, it is computed by convolving $G_\sigma$ with kernel $K$ [S1].
    $$ \sigma(\mathbf{x}_{\text{coarse}}) = \mathcal{T}(K * G_\sigma, \mathbf{x}_{\text{coarse}}) $$
    where $\mathcal{T}$ denotes trilinear (or appropriate spherical) interpolation [S1]. The kernel $K$ acts as an average pooling operation, smoothing the density field to capture broader geometric extents [S1].

2.  **Coarse Sampling:**
    Sample $N_c$ points $\mathbf{x}_i$ along the ray. In the standard spherical grid implementation, these samples are distributed exponentially along the ray rather than uniformly, to match the exponential increase in grid resolution along the radial direction [S1]. Let $\delta_i$ be the distance between consecutive coarse samples [S1].

3.  **Weight Calculation:**
    Calculate importance sampling weights $w_i$ for each coarse sample based on the estimated density $\sigma_i = \sigma(\mathbf{x}_i)$ and accumulated transmittance $\tau_i$ [S1]:
    $$ w_i = \tau_i (1 - e^{-\sigma_i \delta_i}) $$
    where the accumulated transmittance is:
    $$ \tau_i = e^{-\sum_{j=1}^{i-1} \sigma_j \delta_j} $$
    with $\tau_1 = 1$ [S1].

4.  **Fine Sampling:**
    Sample $N_f$ fine points from the probability distribution defined by the normalized weights $w_i$. These points are used to query the dense grids for precise density and color [S1].

5.  **Rendering:**
    Render the pixel using the combined set of $N = N_c + N_f$ samples. The color is computed via volume rendering integration [S1]:
    $$ \hat{C} = \sum_{i=1}^{N} \tau_i (1 - e^{-\sigma(\mathbf{x}_i)\delta_i}) c(\mathbf{x}_i, \mathbf{d}) + \tau_{N+1} c_{\text{env}}(\mathbf{d}) $$
    where $c(\mathbf{x}_i, \mathbf{d})$ is the color obtained from the appearance grid $G_a$ and an optional decoding MLP, and $c_{\text{env}}(\mathbf{d})$ is the environment map color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_density_adaptation(ray_origin, ray_direction, G_sigma, G_appearance, 
                                    N_c, N_f, kernel_K, env_map=None):
    """
    Renders a single ray using Hierarchical Density Adaptation.
    
    Args:
        ray_origin: 3D vector (o)
        ray_direction: 3D vector (d)
        G_sigma: Dense density grid object with method query_and_smooth(x, kernel)
        G_appearance: Dense appearance grid object with method query(x, d)
        N_c: Number of coarse samples
        N_f: Number of fine samples
        kernel_K: Convolution kernel (average pooling)
        env_map: Optional environment map function c_env(d)
        
    Returns:
        pixel_color: Scalar or RGB vector
    """
    # Step 1 & 2: Coarse Sampling and Density Estimation
    # In spherical grids, samples are distributed exponentially along the ray
    coarse_points = []
    coarse_densities = []
    coarse_transmittances = []
    
    current_transmittance = 1.0
    # Determine step sizes based on exponential distribution logic specific to the grid
    # [G: Assuming a helper function exponential_ray_samples exists for the specific grid topology]
    distances = exponential_ray_samples(ray_origin, ray_direction, N_c)
    
    for i, dist in enumerate(distances):
        x_coarse = ray_origin + dist * ray_direction
        
        # On-the-fly coarse density: Convolve dense grid with kernel K
        # This avoids skipping thin surfaces present in G_sigma
        sigma_coarse = G_sigma.query_and_smooth(x_coarse, kernel_K)
        
        coarse_points.append(x_coarse)
        coarse_densities.append(sigma_coarse)
        coarse_transmittances.append(current_transmittance)
        
        # Update transmittance for next step
        # delta_i is the distance to the next sample. 
        # For the last sample, delta is effectively infinite or handled by termination
        if i < len(distances) - 1:
            delta_i = distances[i+1] - dist
        else:
            delta_i = 0.0 # Or a large value if ray continues, but weight becomes 0 if transmittance is 0
            
        current_transmittance *= math.exp(-sigma_coarse * delta_i)

    # Step 3: Calculate Importance Sampling Weights
    weights = []
    for i in range(N_c):
        sigma_i = coarse_densities[i]
        tau_i = coarse_transmittances[i]
        # Delta for the segment covered by this sample
        if i < N_c - 1:
            delta_i = distances[i+1] - distances[i]
        else:
            # For the last coarse sample, we need a delta to compute weight for fine sampling
            # The source defines w_i = tau_i * (1 - exp(-sigma_i * delta_i))
            # We assume a standard segment length or infinite for the tail
            delta_i = distances[i] - distances[i-1] if i > 0 else 1.0 
            
        w_i = tau_i * (1.0 - math.exp(-sigma_i * delta_i))
        weights.append(w_i)
        
    # Normalize weights for probability distribution
    total_weight = sum(weights)
    if total_weight > 0:
        probs = [w / total_weight for w in weights]
    else:
        probs = [1.0 / N_c] * N_c
        
    # Step 4: Fine Sampling
    # Sample N_f points based on the probability distribution of coarse segments
    fine_points = []
    # [G: Implementation of sampling from discrete distribution defined by probs]
    # For each fine sample, pick a coarse segment index k with probability probs[k],
    # then sample a point uniformly within that segment's distance range.
    fine_points = sample_from_distribution(probs, distances, N_f)
    
    # Combine coarse and fine points
    all_points = coarse_points + fine_points
    
    # Step 5: Rendering
    # Sort all points by distance from origin
    all_points.sort(key=lambda p: distance(ray_origin, p))
    
    pixel_color = 0.0
    accumulated_transmittance = 1.0
    
    for i, x in enumerate(all_points):
        # Query dense grids for precise values
        sigma = G_sigma.query(x)
        color = G_appearance.query(x, ray_direction)
        
        # Determine delta for this sample
        if i < len(all_points) - 1:
            delta = distance(all_points[i], all_points[i+1])
        else:
            delta = 0.0 # Or handle tail separately
            
        # Volume rendering contribution
        weight = accumulated_transmittance * (1.0 - math.exp(-sigma * delta))
        pixel_color += weight * color
        accumulated_transmittance *= math.exp(-sigma * delta)
        
    # Add environment map contribution if available
    if env_map is not None:
        pixel_color += accumulated_transmittance * env_map(ray_direction)
        
    return pixel_color
```

**Parameters:**
*   $N_c$: Number of coarse samples along the ray. [S1]
*   $N_f$: Number of fine samples along the ray. [S1]
*   $K$: Convolution kernel. Typically an average pooling kernel. [S1]

**Edge Cases and Pitfalls:**
*   **Skipping Surface Regions:** Evaluating density directly from the dense grid $G_\sigma$ at coarse points may skip important surface regions if the coarse sampling interval is larger than the feature thickness. This is mitigated by the on-the-fly convolution with $K$ [S1].
*   **Grid Resolution Mismatch:** In spherical grids, resolution increases exponentially with radius. Uniform sampling may under-sample distant regions. Coarse samples should be distributed exponentially to match the grid structure [S1].

## Subtopics

### Environment Map Integration
This variant integrates an equirectangular environment map $E \in \mathbb{R}^{H \times W \times 3}$ to handle outdoor or large indoor environments where the bounded feature grid (limited by $R_{max}$) cannot represent distant geometry. The environment map represents visibility at almost infinite distance. The rendering equation is extended to include a tail term: $\hat{C} = \sum_{i=1}^{N} \tau_i (1 - e^{-\sigma(\mathbf{x}_i)\delta_i}) c(\mathbf{x}_i, \mathbf{d}) + \tau_{N+1} c_{\text{env}}(\mathbf{d})$, where $c_{\text{env}}(\mathbf{d}) = \mathcal{E}(u, v; \mathbf{d})$ and $(u, v)$ are sampling positions dependent only on the viewing direction $\mathbf{d}$ [S1].

## Operating characteristics
*   **Memory:** Requires no additional memory for a dedicated coarse grid, as the coarse density is derived from the existing dense grid $G_\sigma$ [S1].
*   **Time Complexity:** Adds the cost of a convolution operation (average pooling) over the grid or per-sample during the coarse stage, plus the overhead of $N_c$ coarse samples. The total sample count is $N_c + N_f$.
*   **Strengths:** Prevents aliasing and missing thin structures that occur with direct dense grid sampling. Efficient as it reuses the primary density representation.
*   **Limitations:** The quality of the coarse estimate depends on the kernel $K$ and the density grid resolution. If the grid is too sparse, even the smoothed estimate may be inaccurate.
*   **When to use:** When using explicit feature grids (like balanced spherical grids) where maintaining a separate coarse grid is memory-prohibitive or unnecessary, and surface details are thin relative to the coarse sampling interval.

## Background
This method was introduced in the context of EgoNeRF, a radiance field representation using balanced spherical feature grids for egocentric view synthesis [S1]. It was proposed to address the limitations of standard hierarchical sampling when applied to explicit grids, where direct sampling might miss geometry. The approach leverages the physical meaning of volume density in the grid to allow simple convolution-based smoothing for coarse estimation [S1].

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
