# Uniform Ray Sampling

Also called: Uniform Sampling, Uniform Point Sampling, Uniform Point Sampling along Rays

## Overview
Uniform Ray Sampling generates a fixed number of 3D points along a camera ray by distributing sample depths evenly or randomly within the interval defined by the near and far clipping planes [S2][S5, S7, unofficial]. It serves as the baseline discretization method for volume rendering and neural radiance fields (NeRF) when no prior information about scene density or transmittance is available [S2, S10]. Unlike [Stratified Ray Sampling](3014-stratified-ray-sampling.md) or [Importance Sampling for SPH Rendering](3624-importance-sampling-for-sph-rendering.md), which concentrate samples in high-contribution regions, uniform sampling treats all depth intervals equally, assuming a constant probability density function (PDF) [S2]. This makes it the simplest to implement but often the least efficient for scenes with large empty spaces or complex geometry, as it may oversample empty regions and undersample dense surfaces [S2].

## How it works
**Inputs:**
*   `origins`: Tensor of ray origins, shape $(N, 3)$, where $N$ is the number of rays.
*   `directions`: Tensor of unit-norm ray directions, shape $(N, 3)$.
*   `near`: Scalar or tensor of near clipping plane distances.
*   `far`: Scalar or tensor of far clipping plane distances.
*   `num_samples` (or `n_pts_per_ray`): Integer $N_s$, the number of points to sample per ray.

**Outputs:**
*   `sample_points`: Tensor of 3D coordinates, shape $(N, N_s, 3)$.
*   `sample_lengths`: Tensor of segment lengths, shape $(N, N_s, 1)$ or $(N, N_s, 3)$ depending on implementation, representing the distance between consecutive samples or the depth value itself [S6][S8, unofficial].

**Data Structures:**
*   **RayBundle**: A container object holding `origins`, `directions`, `sample_points`, and `sample_lengths` [S6][S8, unofficial].
*   **t_values**: A 1D tensor of linearly spaced values between 0 and 1, or depth values between `near` and `far` [S9, S12, unofficial].

**Steps:**

1.  **Generate Depth Values**: Create $N_s$ depth values $d_i$ uniformly distributed between `near` and `far`.
    *   *Linear Depth*: Use linear spacing:
        $$ d_i = \text{near} + i \cdot \frac{\text{far} - \text{near}}{N_s - 1}, \quad i = 0, \dots, N_s-1 $$
        Or using `torch.linspace` in $[0, 1]$ space mapped to depth:
        $$ t_i = \text{linspace}(0, 1, N_s) $$
        $$ d_i = \text{near} \cdot (1 - t_i) + \text{far} \cdot t_i $$
        [S7, S9, S12, S14, unofficial]
    *   *Random Uniform*: Generate $N_s$ random variables $u_i \sim U(0, 1)$ for each ray:
        $$ d_i = \text{near} + (\text{far} - \text{near}) \cdot u_i $$
        [S2][S4, unofficial]
    *   *Perturbation (Stratified)*: To reduce aliasing, add uniform noise to linearly spaced samples. Compute midpoints $m_i = 0.5(d_i + d_{i+1})$, define intervals $[l_i, u_i]$, and sample:
        $$ d_i = l_i + (u_i - l_i) \cdot \epsilon_i, \quad \epsilon_i \sim U(0, 1) $$
        [S9, S14, unofficial]

2.  **Broadcast Ray Parameters**: Reshape `origins` to $(N, 1, 3)$ and `directions` to $(N, 1, 3)$ to allow broadcasting with the depth tensor [S5, S8, S12, unofficial].

3.  **Compute 3D Coordinates**: Calculate sample positions $\mathbf{x}_i$ using the ray equation:
    $$ \mathbf{x}_i = \mathbf{o} + d_i \cdot \mathbf{r} $$
    where $\mathbf{o}$ is the origin, $\mathbf{r}$ is the direction, and $d_i$ is the depth [S7, unofficial][S10][S13, unofficial].

4.  **Compute Sample Lengths**: Calculate the distance between consecutive samples for transmittance estimation. For linear spacing, $\Delta d_i = d_{i+1} - d_i$. The last sample's length is often set to a large value or the distance to the far plane [S7, S12, unofficial].

**Parameters:**
*   `num_samples` ($N_s$): Typically 64 for coarse NeRF training, up to 256 or more for high-quality rendering [S9, S14, unofficial].
*   `near`, `far`: Defined by the camera frustum or scene bounding box [S14, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Ray Direction**: If a ray direction is zero, replace with a small epsilon (e.g., $10^{-6}$) to avoid division by zero or invalid geometry [S14, unofficial].
*   **Precision**: When using `float32` for interpolation and arithmetic, cast back to `float16` only at storage to maintain precision [S1, unofficial].
*   **Boundary Conditions**: Ensure samples lie strictly within $[near, far]$. If using NDC sampling, filter points outside the AABB bounding box [S14, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def uniform_ray_sampling(origins, directions, near, far, num_samples, perturb=0.0):
    """
    Generate uniform samples along camera rays.
    
    Args:
        origins: Tensor of shape (N, 3)
        directions: Tensor of shape (N, 3), unit-norm
        near: Scalar or Tensor of shape (N,)
        far: Scalar or Tensor of shape (N,)
        num_samples: int
        perturb: float, amount of random noise to add for stratification
        
    Returns:
        sample_points: Tensor of shape (N, num_samples, 3)
        sample_lengths: Tensor of shape (N, num_samples, 1)
    """
    # 1. Generate linearly spaced t values in [0, 1]
    t_vals = torch.linspace(0.0, 1.0, num_samples, device=origins.device) # (num_samples,)
    
    # 2. Map t values to depth values [near, far]
    # Handle broadcasting for batched near/far
    near_expanded = near.unsqueeze(1) # (N, 1)
    far_expanded = far.unsqueeze(1)   # (N, 1)
    
    z_vals = near_expanded * (1.0 - t_vals) + far_expanded * t_vals # (N, num_samples)
    
    # 3. Optional: Add perturbation for stratified sampling
    if perturb > 0.0:
        # Compute midpoints
        mids = 0.5 * (z_vals[:, 1:] + z_vals[:, :-1]) # (N, num_samples - 1)
        
        # Define intervals
        upper = torch.cat([mids, far_expanded], dim=-1) # (N, num_samples)
        lower = torch.cat([near_expanded, mids], dim=-1) # (N, num_samples)
        
        # Generate random noise
        rand_noise = perturb * torch.rand(z_vals.shape, device=origins.device)
        
        # Sample within intervals
        z_vals = lower + (upper - lower) * rand_noise
        
    # 4. Broadcast origins and directions
    origins_expanded = origins.unsqueeze(1) # (N, 1, 3)
    directions_expanded = directions.unsqueeze(1) # (N, 1, 3)
    z_vals_expanded = z_vals.unsqueeze(2) # (N, num_samples, 1)
    
    # 5. Compute 3D points
    sample_points = origins_expanded + directions_expanded * z_vals_expanded # (N, num_samples, 3)
    
    # 6. Compute sample lengths (deltas)
    # Delta is distance between consecutive samples
    deltas = torch.diff(z_vals, dim=1, prepend=torch.zeros_like(near_expanded)) # (N, num_samples)
    # Note: The first delta is near - near = 0, which might need adjustment depending on integration scheme
    # Often, the last delta is set to a large value or far - last_sample
    deltas[:, -1] = far_expanded.squeeze(1) - z_vals[:, -1]
    
    sample_lengths = deltas.unsqueeze(2) # (N, num_samples, 1)
    
    return sample_points, sample_lengths
```

## Subtopics

### [Stratified Monte Carlo Radiance Estimate](1373-stratified-monte-carlo-radiance-estimate.md)
An unbiased radiance estimation method that reduces variance compared to uniform sampling by dividing the ray interval into strata and sampling one point uniformly at random within each stratum. It ensures coverage of the entire interval, making it effective for degenerate distributions where uniform sampling requires significantly more samples.

### [Evenly Spaced Sampling with Preprocessed Rays and Depth Indicators](2335-evenly-spaced-sampling-with-preprocessed-rays-and-depth-indi.md)
Ensures even spacing along rays within a voxel cell using preprocessed ray data and a 4-bit depth-sampling indicator stored in pixel opacity. It maintains sampling continuity across cells, differing from uniform sampling by relying on shared preprocessed rays and stateful indicators.

### [Uniform NDC Point Sampling](2395-uniform-ndc-point-sampling.md)
Transforms camera rays into an average camera frame and uniformly samples points within the projected Normalized Device Coordinates (NDC) space. It provides linear sampling in disparity, combining benefits of uniform and logarithmic sampling, but is limited to scenes where all samples lie in the front hemisphere of the average camera frame.

### [Batch Stratified Sampling](2730-batch-stratified-sampling.md)
Generates $N_c$ evenly-spaced depth bins in $[near, far]$ and perturbs sample positions within each bin to reduce aliasing. It extends uniform sampling by stratifying the depth interval for more even distribution.

### [Stratified Ray Sampling](3014-stratified-ray-sampling.md)
Divides the ray interval into $N$ evenly-spaced bins and draws one sample uniformly at random from each bin. It introduces controlled randomness within structured intervals to break up regular sampling patterns, improving training data quality.

### [Local Sampling](5532-local-sampling.md)
Concentrates samples around a known surface representation (e.g., depth texture) rather than distributing them uniformly along the entire ray. It reduces computational cost by avoiding empty space evaluations, requiring an oracle for surface localization.

### [Non-linear (Logarithmic) Sampling](5814-non-linear-logarithmic-sampling.md)
Distributes points along a view ray using a logarithmic function to concentrate density near the camera. It addresses the limitation of uniform sampling in large scenes by balancing spatial footprint to approximate cubic volumes in view space.

### [Pixel-guided ray sampling](7551-pixel-guided-ray-sampling.md)
Calculates a prior probability distribution based on color variation within pixel neighborhoods to identify high-information regions. It concentrates computational resources on pixels with sharp color differences, replacing constant PDF with spatially varying distributions.

### Forward Sampling Inside Surface
Samples points forward along the ray from a known surface to get SDF values inside the object, used when latent space is not pretrained [S3, unofficial]. It initializes samples at the surface depth and adds grid offsets to probe the interior [S3, unofficial].

### Camera Ray Sampling for Translucent Materials
Samples the ray randomly with a small number of samples (as low as one) after the camera ray reaches translucent material [S15]. It uses a specific formula involving the extinction coefficient $\sigma_t$ and maximum depth $d_{\max}$ to place $k_{\max}$ samples, ensuring coverage even for small object widths [S15].

### Context-based Probability Ray Sampling
Non-uniformly samples rays based on image context, calculating a prior probability density function as the standard deviation of color in pixel neighborhoods [S16]. It samples more rays in complex areas and fewer in trivial areas, unlike uniform sampling [S16].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot N_s)$, where $N$ is the number of rays and $N_s$ is the number of samples per ray. The computation is highly parallelizable on GPUs [S1, S12, unofficial].
*   **Memory Complexity**: $O(N \cdot N_s \cdot 3)$ for storing sample points, plus $O(N \cdot N_s)$ for depths/lengths [S6][S8, unofficial].
*   **Performance**: On an RTX PRO 6000 Blackwell, sampling 1M rays for the dragon dataset takes ~2.8ms (fp32) [S1, unofficial]. Optimizations like hoisting `stepSize` out of inner loops can reduce register usage from 80 to 74 [S1, unofficial].
*   **Strengths**: Simple to implement; deterministic (if no perturbation); unbiased estimator of the integral if the number of samples is sufficient [S2].
*   **Limitations**: Inefficient for scenes with large empty regions or complex geometry; requires many samples to achieve low variance compared to importance sampling methods [S2].
*   **When to Use**: Baseline for NeRF training; scenes with uniform density; when no transmittance estimator is available [S2, S10].
*   **When Not to Use**: Large unbounded scenes (use [Non-linear (Logarithmic) Sampling](5814-non-linear-logarithmic-sampling.md)); scenes with sparse geometry (use [Spatial Skipping](2516-sampling-and-jittering.md) or [Importance Sampling](3624-importance-sampling-for-sph-rendering.md)); real-time rendering with limited compute.

## Background
Uniform sampling is the default raymarching strategy in the original NeRF paper [16] and remains a standard baseline in neural radiance fields [S2, S10]. It assumes that every point along the ray contributes equally to the final rendering, leading to a constant PDF and linearly decaying transmittance [S2]. The method is widely used in academic benchmarks and open-source implementations due to its simplicity [S5, S12, S14, unofficial].

## Sources
[S1] `ray_implicit_intersection` improvements, https://github.com/openvdb/fvdb-core/pull/663, fetched 2026-10-25 (unofficial)
[S2] NerfAcc: Efficient Sampling Accelerates NeRFs, https://doi.org/10.1109/iccv51070.2023.01699, fetched 2026-10-26
[S3] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S4] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S5] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S6] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S7] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S8] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S9] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S10] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
[S11] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S12] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S13] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
[S14] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S15] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
[S16] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
