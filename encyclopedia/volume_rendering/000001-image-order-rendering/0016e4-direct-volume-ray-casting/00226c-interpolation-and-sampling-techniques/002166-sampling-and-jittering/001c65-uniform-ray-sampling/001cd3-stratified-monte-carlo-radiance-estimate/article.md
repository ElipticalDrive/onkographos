# Stratified Monte Carlo Radiance Estimate

Also called: RVS Stratified Estimate, Stratified Sampling

## Overview
The Stratified Monte Carlo Radiance Estimate is an unbiased radiance estimation method that reduces variance compared to uniform sampling, particularly for degenerate distributions [S1, unofficial]. It sets itself apart from its parent [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by dividing the ray interval into bins and sampling one point per bin, ensuring better coverage of the integration domain [S1, unofficial]. It differs from [Evenly Spaced Sampling with Preprocessed Rays and Depth Indicators](2335-evenly-spaced-sampling-with-preprocessed-rays-and-depth-indi.md) by introducing stochastic jitter within bins rather than fixed positions [S3, unofficial].

## How it works
**Inputs:**
- Ray $\mathbf{r}$ with near clipping plane $t_n$ and far clipping plane $t_f$ [S1, unofficial].
- Number of samples $k$ [S1, unofficial].
- Boolean flag `randomize` indicating whether to jitter samples within bins [S3, unofficial].

**Outputs:**
- Radiance estimate $\hat{C}_{\text{IW}}(\mathbf{r})$ [S1, unofficial].
- Sample points stored in `RayBundle.sample_points` [S2, unofficial].
- Sample lengths (distances from ray origin) stored in `RayBundle.sample_lengths` [S2, unofficial].

**Data Structures:**
- `bins`: Array of size $k+1$ containing bin boundaries.
- `samples`: Array of size $k$ containing sample locations within bins.
- `RayBundle.sample_lengths`: Tensor of size $k$ storing $t_i$ values.
- `RayBundle.sample_points`: Tensor of size $k \times 3$ storing $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$.

**Steps:**
1. Define bin boundaries by dividing the interval $[t_n, t_f]$ into $k$ equal parts:
   $$ t_i = t_n + i \cdot \frac{t_f - t_n}{k}, \quad i = 0, \dots, k $$
   Let $t_0 = t_n$ and $t_k = t_f$ [S1, unofficial].
2. Extract lower and upper bounds for each bin $i \in \{1, \dots, k\}$:
   $$ \text{lower}_i = t_{i-1}, \quad \text{upper}_i = t_i $$ [S3, unofficial].
3. Generate sample locations $\tau_i$ for each bin:
   - If `randomize` is true:
     $$ \tau_i = \text{lower}_i + u_i \cdot (\text{upper}_i - \text{lower}_i) $$
     where $u_i \sim U[0, 1]$ is a uniform random number [S3, unofficial].
   - If `randomize` is false:
     $$ \tau_i = 0.5 \cdot (\text{lower}_i + \text{upper}_i) $$ [S3, unofficial].
4. Compute the radiance estimate using the stratified formula [S1, unofficial]:
   $$ \hat{C}_{\text{IW}}(\mathbf{r}) = \sum_{i=1}^k (t_i - t_{i-1}) c_{\mathbf{r}}(\tau_i) \frac{dF_{\mathbf{r}}}{dt} \bigg|_{t=\tau_i} $$
   where $c_{\mathbf{r}}(\tau_i)$ is the color contribution at sample $\tau_i$ and $\frac{dF_{\mathbf{r}}}{dt}$ is the differential transmittance/opacity derivative at $\tau_i$ [S1, unofficial].
5. Store $\tau_i$ in `RayBundle.sample_lengths` and compute $\mathbf{x}_i = \mathbf{o} + \tau_i \mathbf{d}$ in `RayBundle.sample_points` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stratified_monte_carlo_radiance_estimate(ray_origin, ray_direction, near, far, k, randomize, color_func, dF_dt_func):
    """
    Compute stratified Monte Carlo radiance estimate.
    
    Args:
        ray_origin: (3,) array, origin of the ray.
        ray_direction: (3,) array, normalized direction of the ray.
        near: float, near clipping plane distance.
        far: float, far clipping plane distance.
        k: int, number of samples.
        randomize: bool, if True, jitter samples within bins; if False, use midpoints.
        color_func: callable, returns color contribution c_r(tau) at distance tau.
        dF_dt_func: callable, returns dF_r/dt at distance tau.
        
    Returns:
        radiance_estimate: float, the estimated radiance.
        sample_points: (k, 3) array, 3D coordinates of sample points.
        sample_lengths: (k,) array, distances along the ray.
    """
    # Step 1: Generate bin boundaries
    bins = np.linspace(near, far, k + 1)
    
    # Step 2: Extract lower and upper bounds
    lower = bins[:-1]
    upper = bins[1:]
    
    # Step 3: Generate sample locations
    if randomize:
        # Sample uniformly within each bin
        uniform_samples = np.random.uniform(0, 1, size=k)
        samples = lower + uniform_samples * (upper - lower)
    else:
        # Use midpoint of each bin
        samples = 0.5 * (lower + upper)
        
    # Step 4: Compute radiance estimate
    radiance_estimate = 0.0
    for i in range(k):
        bin_width = upper[i] - lower[i]
        tau_i = samples[i]
        c_r = color_func(tau_i)
        dF_dt = dF_dt_func(tau_i)
        radiance_estimate += bin_width * c_r * dF_dt
        
    # Step 5: Store results
    sample_lengths = samples
    sample_points = ray_origin + sample_lengths[:, np.newaxis] * ray_direction
    
    return radiance_estimate, sample_points, sample_lengths
```

**Parameters:**
- $k$: Number of samples. Typical values: $k=32$ for challenging scenes (e.g., foggy fields), $k=4$ for simpler scenes (e.g., walls) [S1, unofficial].
- `randomize`: Boolean flag. Default is often true to reduce aliasing [S3, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Ensure $t_n < t_f$ to avoid division by zero or invalid intervals [S1, unofficial].
- Degenerate distributions: Stratified sampling is particularly effective when the integrand is degenerate, but if the function is constant, uniform sampling may suffice [S1, unofficial].
- Boundary conditions: Samples must lie within $[t_n, t_f]$ to avoid out-of-bounds memory access or invalid physical interpretations [S2, unofficial].

## Subtopics

### [Antithetic Sampling](7196-antithetic-sampling.md)
A variance reduction technique for estimating the derivative of a BSDF, exploiting symmetry of NDFs. Choose when derivatives are odd functions.

### [Decomposed sampling](1374-decomposed-sampling.md)
Addresses cases where derivatives are even functions. Choose when symmetry is even.

## Operating characteristics
**Time Complexity:** $O(k)$ per ray, where $k$ is the number of samples [S1, unofficial].
**Memory Complexity:** $O(k)$ to store sample points and lengths [S2, unofficial].
**Measured Performance:** In a 'Foggy' field experiment, $k=32$ stratified samples match the variance of $k=256$ uniform samples [S1, unofficial]. In a 'Wall' field experiment, $k=4$ stratified samples match the baseline [S1, unofficial].
**Strengths:** Significantly reduces variance compared to uniform sampling, especially for degenerate distributions [S1, unofficial]. Unbiased estimate [S1, unofficial].
**Limitations:** Requires evaluation of $dF_{\mathbf{r}}/dt$ at each sample point, which may be computationally expensive if not precomputed [S1, unofficial].
**When to Use:** When variance reduction is critical and the integrand is degenerate or highly variable [S1, unofficial].
**When Not to Use:** When computational budget is extremely tight and $k$ must be very small, or when the integrand is constant [S1, unofficial].

## Background
The method originates from differentiable rendering research, specifically in the context of volume sampling [S1, unofficial]. It was developed to improve the efficiency of radiance estimation in neural radiance fields and other differentiable rendering pipelines [S1, unofficial]. The technique builds on the principle of stratified sampling to reduce variance in Monte Carlo integration [S1, unofficial].

## Sources
[S1] Differentiable Rendering with Reparameterized Volume Sampling, https://doi.org/10.48550/arxiv.2302.10970, fetched 2026-10-26 (unofficial)
[S2] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S3] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
