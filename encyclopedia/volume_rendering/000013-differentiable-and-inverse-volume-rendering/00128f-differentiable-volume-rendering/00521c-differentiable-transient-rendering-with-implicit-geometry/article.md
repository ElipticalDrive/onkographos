# Differentiable Transient Rendering with Implicit Geometry

Also called: Differentiable Transient Rendering, Path-space differentiable transient rendering

## Overview
This method computes time-resolved illumination at sensor points by simulating third-bounce light transport between a laser source and a non-line-of-sight (NLOS) sensor using implicit surface geometry [S1]. It differs from standard differentiable volume rendering by operating in path space for transient measurements, discretizing continuous time-of-flight into bins via Gaussian smoothing, and bypassing explicit binary visibility checks by relying on volumetric intensity as a visibility estimator [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Implicit geometry representation $G$ with parameters $\Theta_G$, laser parameters $\Theta_{ls}$, sensor parameters, laser position $x_l$, sensor position $x_s$, and transient bin index $\tau$.
*   **Outputs:** Sensed transient data matrix $H_R(x_l, x_s, \tau)$ representing time-resolved illumination at the sensor.

**Data Structures:**
*   **Implicit Surface:** Defined by parameters $\Theta_G$. Points $x_g$ and normals $n_g$ are estimated from volumetric intensities.
*   **Transient Bins:** Discrete time indices $\tau$. Continuous time $t$ is mapped to $\tau$ via Gaussian weighting.

**Steps:**

1.  **Ray Sampling:** Sample rays originating from the sensor points $x_s$ [S1].
2.  **Implicit Surface Estimation:** Estimate implicit surface points $x_g$ and normals $n_g$ along the sampled rays. The geometry is derived from volumetric intensities, which depend on illumination reaching the surface without occluders [S1].
3.  **Radiometric Contribution Calculation:** Compute the radiometric contribution $K(\bar{x}, t)$ for a path $\bar{x} \equiv x_l \to x_g \to x_s$ [S1]. This represents time-resolved transport in path space.
    *   *Note:* The method avoids computing the binary visibility term $V$ (typically required for path throughput $\mathbb{T}$) because the implicit surface estimation relies on volumetric intensities that inherently reflect unoccluded illumination [S1].
4.  **Time Discretization:** Discretize the continuous time $t$ of the path contribution $K(\bar{x}, t)$ into transient bins $\tau$ using a differentiable Gaussian distribution [S1]:
    $$
    \hat{K}(\bar{x}, \tau) = |K(\bar{x}, t)| \exp \left( - \frac{(\tau - t)^2}{2\sigma_t^2} \right)
    $$
    where $\tau$ is the transient bin index, $t$ is the continuous time, and $\sigma_t$ is the standard deviation for time bin discretization [S1].
5.  **Path Summation:** Approximate the time-resolved measurement $H_r(x_l, x_s, \tau)$ by summing the discrete path contributions $\hat{K}(\bar{x}, \tau)$ for all paths $\bar{x}$ starting at $x_l$ and ending at $x_s$, sampled via concentric disk mapping [S1]:
    $$
    H_r(x_l, x_s, \tau) \approx \sum_{\bar{x} \in X} \hat{K}(\bar{x}, \tau)
    $$
    where $X$ is the set of such paths [S1].
6.  **Laser-Sensor Model Application:** Apply the joint laser-sensor model to obtain the final sensed transient data $H_R$ by convolving with the system response $\Psi(\tau)$ and adding an intensity offset $\eta_s$ [S1]:
    $$
    H_R(x_l, x_s, \tau) = \Psi(\tau) * H_r(x_l, x_s, \tau) + \eta_s
    $$
    where $\eta_s$ accounts for ambient light and sensor dark count rate [S1].

**Parameters:**
*   $\sigma_t$: Standard deviation for Gaussian discretization. Set to 0.62 to ensure the Full Width at Half Maximum (FWHM) covers a unit time bin [S1].
*   $\eta_s$: Intensity offset parameter for ambient light and dark count rate [S1].

**Pitfalls:**
*   Differentiating the binary visibility function $V$ is challenging. The method mitigates this by assuming volumetric intensities are a good estimator of geometry visible from both laser and sensor positions, thus avoiding explicit visibility computation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_transient_rendering(implicit_geometry, laser_pos, sensor_pos, 
                                     sigma_t=0.62, eta_s=0.0, system_response=None):
    """
    Computes time-resolved illumination H_R using implicit geometry.
    
    Args:
        implicit_geometry: Object providing estimate_surface_points(rays) and 
                           estimate_normals(rays).
        laser_pos: 3D vector, position of the laser source x_l.
        sensor_pos: 3D vector, position of the sensor x_s.
        sigma_t: Float, standard deviation for time bin discretization (default 0.62).
        eta_s: Float, intensity offset for ambient light/dark count (default 0.0).
        system_response: Function or array, Psi(tau) convolution kernel.
        
    Returns:
        H_R: Array of sensed transient data indexed by tau.
    """
    # 1. Sample rays from sensor positions
    # In a full implementation, this involves sampling directions from sensor_pos
    # towards the relay wall or scene volume. Here we assume rays are provided
    # or generated by a sampling strategy.
    rays = sample_rays_from_sensor(sensor_pos)
    
    # 2. Estimate implicit surface points and normals
    # x_g are points on the implicit surface estimated from volumetric intensities
    surface_points = implicit_geometry.estimate_surface_points(rays)
    surface_normals = implicit_geometry.estimate_normals(rays)
    
    # 3. Compute radiometric contribution K(x_bar, t) for each path
    # Path x_bar = x_l -> x_g -> x_s
    # This step computes the physical transport contribution.
    # Note: Binary visibility V is avoided; intensity acts as visibility estimator.
    path_contributions = []
    path_times = []
    
    for i, (x_g, n_g) in enumerate(zip(surface_points, surface_normals)):
        # Compute path x_bar = laser_pos -> x_g -> sensor_pos
        # Calculate distance and time of flight
        dist_laser_to_surf = distance(laser_pos, x_g)
        dist_surf_to_sensor = distance(x_g, sensor_pos)
        total_dist = dist_laser_to_surf + dist_surf_to_sensor
        
        # Continuous time t (assuming speed of light c=1 for normalized units, 
        # or scale appropriately)
        t = total_dist / speed_of_light
        
        # Compute radiometric contribution K
        # This involves albedo rho, geometry terms, and Fresnel etc.
        # Estimated from physical observation in backward pass for albedo.
        # For forward pass, we assume K is computed based on current geometry params.
        k_val = compute_radiometric_contribution(laser_pos, x_g, sensor_pos, 
                                                 n_g, implicit_geometry.params)
        path_contributions.append(k_val)
        path_times.append(t)
        
    # 4. Discretize into transient bins using Gaussian distribution
    # H_r(tau) = sum over paths of K_hat(x_bar, tau)
    num_bins = get_num_transient_bins()
    H_r = zeros(num_bins)
    
    for k_val, t in zip(path_contributions, path_times):
        # Gaussian weighting for each bin tau
        for tau in range(num_bins):
            # Gaussian kernel: exp(-(tau - t)^2 / (2 * sigma_t^2))
            weight = exp(-((tau - t)**2) / (2 * sigma_t**2))
            H_r[tau] += abs(k_val) * weight
            
    # 5. Apply laser-sensor model
    # H_R = Psi * H_r + eta_s
    if system_response is not None:
        H_R = convolve(H_r, system_response) + eta_s
    else:
        # If no specific response, assume identity or default
        H_R = H_r + eta_s
        
    return H_R
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of sampled paths and transient bins. Summation over paths is linear in the number of samples.
*   **Memory Complexity:** Requires storing intermediate path contributions and transient bins. Avoids storing large visibility buffers.
*   **Strengths:** Enables gradient-based optimization of NLOS scene parameters (geometry, albedo, system parameters) by providing differentiable transient measurements. Handles implicit geometry efficiently.
*   **Limitations:** Relies on the assumption that volumetric intensities approximate visibility, which may fail in highly occluded scenes. Discretization error depends on $\sigma_t$.
*   **When to use:** For inverse rendering problems involving NLOS transient measurements where implicit geometry representation is preferred or necessary.

## Background
Originated in the context of Non-Line-of-Sight (NLOS) inverse rendering, specifically for self-calibrating and fully differentiable NLOS reconstruction [S1]. The method addresses the challenge of differentiating through transient rendering pipelines by using implicit surfaces and Gaussian time discretization.

## Sources
[S1] Self-Calibrating, Fully Differentiable NLOS Inverse Rendering, https://doi.org/10.1145/3610548.3618140, fetched 2026-10-27
