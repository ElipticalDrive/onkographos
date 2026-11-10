# Flash Cache

Also called: Fast Cache, Fast Cache approximation

## Overview
Flash Cache is a volumetric inverse rendering method that reduces bias and variance in radiance estimation by using a high-capacity Neural Radiance Field (NeRF) as a control variate for incoming illumination [S1]. Unlike prior radiance caches that use low-capacity models (introducing bias via inaccurate reflection estimation) or render NeRFs at single evaluation points (introducing bias via volume rendering integral approximation), Flash Cache provides an unbiased, low-variance estimate of the rendering equation [S1]. It achieves this by replacing the recursive incoming radiance term with a NeRF-based cache output, evaluated via importance sampling, while maintaining computational tractability through a "Fast Cache" approximation [S1].

## How it works
Flash Cache combines volumetric rendering with physically-based Monte Carlo light transport to optimize scene parameters $\Phi$ (geometry, materials, lighting) [S1]. The method avoids the recursive evaluation cost of standard path tracing by approximating the volume rendering integral with a single sample point per ray and using a NeRF as a radiance cache for incoming light [S1].

### Inputs and Outputs
*   **Inputs:** A set of input images $I(\mathbf{o}, \boldsymbol{\omega}_o)$, where $\mathbf{o}$ is the camera origin and $\boldsymbol{\omega}_o$ is the viewing direction [S1]. Scene representation parameters $\Phi$ [S1].
*   **Outputs:** Optimized parameters $\Phi$ that minimize the difference between measured images and rendered incident radiance [S1].

### Data Structures
*   **NeRF Radiance Cache:** A neural network (MLP or grid-based) that maps a 3D position $\mathbf{x}$ and direction $\boldsymbol{\omega}_i$ to outgoing radiance $L_{\text{cache}}(\mathbf{x}, \boldsymbol{\omega}_i)$ [S1]. This cache acts as a proxy for incoming radiance $L_i$.
*   **Mixture of von-Mises Fisher (vMF) Distributions:** A spatially-varying probability distribution used to importance-sample incoming directions $\boldsymbol{\omega}_i$ based on the expected distribution of incoming radiance [S1].

### Steps
1.  **Ray Sampling:** For each camera ray $(\mathbf{o}, \boldsymbol{\omega}_o)$, sample a single point $\mathbf{x}(t_k)$ along the ray [S1]. This approximates the volume rendering integral with one sample [S1].
2.  **Weight Calculation:** Compute the quadrature weight $w_k$ for the sample point using the density $\sigma$ along the ray [S1]:
    $$
    w_k = \left(1 - e^{-\sigma(\mathbf{x}(t_k)) (t_{k+1} - t_k)}\right) \exp \left( - \sum_{j=1}^{k-1} \sigma(\mathbf{x}(t_j))(t_{j+1} - t_j) \right)
    $$
    where $t_k$ is the distance along the ray, and $t_{k+1} - t_k$ is the step size [S1].
3.  **Importance Sampling:** At the sample point $\mathbf{x}(t_k)$, use the spatially-varying mixture of vMF distributions to sample $M$ incoming directions $\boldsymbol{\omega}_i^{(j)}$ [S1]. The distribution $p(\boldsymbol{\omega}_i)$ is parameterized by a neural field to capture spatially-varying illumination effects like occlusion and interreflections [S1].
4.  **Radiance Cache Query:** For each sampled direction $\boldsymbol{\omega}_i^{(j)}$, query the NeRF radiance cache to obtain the estimated incoming radiance $L_{\text{cache}}(\mathbf{x}(t_k), \boldsymbol{\omega}_i^{(j)})$ [S1]. This replaces the recursive evaluation of $L_i$ [S1].
5.  **Outgoing Radiance Estimation:** Compute the unbiased estimator for outgoing radiance $\hat{L}_o(\mathbf{x}(t_k), \boldsymbol{\omega}_o)$ using the BRDF $f$, the cache output, and the sampling distribution [S1]:
    $$
    \hat{L}_o(\mathbf{x}(t_k), \boldsymbol{\omega}_o) = \frac{1}{M} \sum_{j=1}^{M} f(\mathbf{x}(t_k), \boldsymbol{\omega}_i^{(j)}, \boldsymbol{\omega}_o) L_{\text{cache}}(\mathbf{x}(t_k), \boldsymbol{\omega}_i^{(j)}) \frac{(\mathbf{n} \cdot \boldsymbol{\omega}_i^{(j)})}{p(\boldsymbol{\omega}_i^{(j)})}
    $$
    where $\mathbf{n}$ is the surface normal at $\mathbf{x}(t_k)$ [S1].
6.  **Incident Radiance Approximation:** Approximate the incident radiance $L_i(\mathbf{o}, \boldsymbol{\omega}_o)$ for the camera ray using the single sample and its weight [S1]:
    $$
    \hat{L}_i(\mathbf{o}, \boldsymbol{\omega}_o) = w_k \hat{L}_o(\mathbf{x}(t_k), \boldsymbol{\omega}_o)
    $$
    Note: The source describes approximating the volume rendering integral with a single sample point [S1]. The full integral is $\int L_o T \sigma dt$, approximated here as $w_k L_o$ [S1].
7.  **Optimization:** Minimize the loss function with respect to $\Phi$ [S1]:
    $$
    \min_{\Phi} \sum_{\mathbf{o}, \boldsymbol{\omega}_o} \| I(\mathbf{o}, \boldsymbol{\omega}_o) - L_i(\mathbf{o}, \boldsymbol{\omega}_o; \Phi) \|^2 + \mathcal{L}_{\text{reg}}(\Phi)
    $$
    where $L_i$ is the estimated incident radiance [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flash_cache_render(ray_origin, ray_direction, nerf_cache, vmf_distribution, brdf, scene_params):
    """
    Renders a single ray using Flash Cache approximation.
    
    Args:
        ray_origin: 3D vector, camera origin o.
        ray_direction: 3D vector, camera direction omega_o.
        nerf_cache: Neural Radiance Field function L_cache(x, omega_i).
        vmf_distribution: Spatially-varying mixture of vMFs for sampling omega_i.
        brdf: Physically-based BRDF function f(x, omega_i, omega_o).
        scene_params: Parameters Phi for geometry, materials, lighting.
        
    Returns:
        incident_radiance: Scalar, estimated L_i(o, omega_o).
    """
    # Step 1: Sample a single point along the ray
    # In practice, t_k is determined by the scene geometry or a fixed step
    # For simplicity, assume a single sample point x_t is provided or derived
    # from the scene representation (e.g., intersection with implicit surface)
    x_t, t_k, t_next = sample_point_along_ray(ray_origin, ray_direction, scene_params)
    
    # Step 2: Calculate quadrature weight w_k
    sigma = nerf_cache.get_density(x_t) # Density sigma(x) from NeRF
    delta_t = t_next - t_k
    weight_k = (1 - exp(-sigma * delta_t)) * exp(-get_accumulated_transmittance(x_t, ray_origin, ray_direction, scene_params))
    
    # Step 3: Importance sample incoming directions
    M = 10 # Number of samples, typical value not specified in sources, assume small integer
    incoming_directions = vmf_distribution.sample(M, x_t)
    
    # Step 4 & 5: Estimate outgoing radiance using cache
    normal = nerf_cache.get_normal(x_t) # Normal n at x_t
    outgoing_radiance_sum = 0.0
    
    for omega_i in incoming_directions:
        # Query cache for incoming radiance L_cache(x_t, omega_i)
        # This approximates L_i(x_t, omega_i)
        l_cache = nerf_cache.query(x_t, omega_i)
        
        # Compute BRDF term
        f_val = brdf.evaluate(x_t, omega_i, ray_direction)
        
        # Cosine term
        cos_theta = dot(normal, omega_i)
        
        # PDF value
        p_val = vmf_distribution.pdf(omega_i, x_t)
        
        # Avoid division by zero
        if p_val < 1e-8:
            p_val = 1e-8
            
        # Add to sum
        term = f_val * l_cache * cos_theta / p_val
        outgoing_radiance_sum += term
        
    outgoing_radiance = outgoing_radiance_sum / M
    
    # Step 6: Approximate incident radiance for the camera ray
    incident_radiance = weight_k * outgoing_radiance
    
    return incident_radiance
```

### Parameters
*   **M:** Number of Monte Carlo samples for incoming directions. Not given in the sources.
*   **$\Phi$:** Scene representation parameters (geometry, materials, lighting). Optimized via gradient descent [S1].
*   **$\mathcal{L}_{\text{reg}}$:** Regularization term. Not given in the sources.

### Edge Cases and Pitfalls
*   **Recursive Evaluation:** Naïve use of the rendering equation requires recursive evaluation of $L_i$, leading to high computational cost [S1]. Flash Cache avoids this by using the cache as a control variate [S1].
*   **Bias:** Using low-capacity caches or single-point NeRF evaluation introduces bias [S1]. Flash Cache uses a high-capacity cache and control variate to avoid this [S1].
*   **Precision:** Division by $p(\boldsymbol{\omega}_i)$ requires care to avoid numerical instability when the PDF is near zero [G].

## Subtopics

### Radiance Cache
A neural field that approximates incoming radiance $L_i$ to avoid recursive path tracing. It is cheaper to evaluate than full volume rendering [S1].

### Control Variate
A variance reduction technique where the NeRF cache output is used to reduce the variance of the Monte Carlo estimator without introducing bias [S1].

### von-Mises Fisher Mixture
A spatially-varying distribution used for importance sampling incoming directions, capturing effects like occlusion and interreflections [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $M$ and the cost of querying the NeRF cache. It is cheaper than recursive path tracing but more expensive than single-point NeRF evaluation [S1].
*   **Memory Complexity:** Requires storing the NeRF cache parameters and the vMF distribution parameters [S1].
*   **Strengths:** Reduces bias and variance in inverse rendering; captures high-frequency and near-field illumination; preserves thin structures via volumetric representation [S1].
*   **Limitations:** Computationally more expensive than low-capacity cache methods; requires careful tuning of the vMF distribution [S1].
*   **When to Use:** When unbiased estimation of complex lighting effects (interreflections, specularities) is required in volumetric inverse rendering [S1].
*   **When Not to Use:** When computational resources are extremely limited and bias is acceptable [S1].

## Background
Flash Cache was introduced by Attal et al. in the paper "Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering" [S1]. It addresses limitations in prior radiance cache methods, which either introduced bias through low-capacity models or single-point evaluations [S1]. The method builds on the principles of Neural Radiance Fields and physically-based Monte Carlo rendering [S1].

## Sources
[S1] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
