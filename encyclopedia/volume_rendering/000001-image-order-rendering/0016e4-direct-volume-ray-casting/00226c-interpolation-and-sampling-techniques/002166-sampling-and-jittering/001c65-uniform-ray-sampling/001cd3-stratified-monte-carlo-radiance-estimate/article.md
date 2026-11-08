# Stratified Monte Carlo Radiance Estimate

Also called: RVS Stratified Estimate, Stratified Sampling

## Overview
The Stratified Monte Carlo Radiance Estimate is an unbiased radiance estimation method that reduces variance compared to uniform sampling, particularly for degenerate distributions where standard Monte Carlo methods struggle [S1, unofficial]. It sets itself apart from its parent [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by dividing the ray interval into discrete bins and sampling one point per bin, ensuring better coverage of the integration domain [S1, unofficial]. Unlike [Evenly Spaced Sampling with Preprocessed Rays and Depth Indicators](2335-evenly-spaced-sampling-with-preprocessed-rays-and-depth-indi.md), which uses fixed deterministic spacing, this method introduces stochastic jitter within bins to maintain unbiasedness while achieving lower variance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray $\mathbf{r}$, near clipping plane $t_n$, far clipping plane $t_f$, number of samples $k$.
*   **Output:** Radiance estimate $\hat{C}_{\text{IW}}(\mathbf{r})$.
*   **Intermediate Data:** Sample points stored in `RayBundle.sample_points` and sample lengths in `RayBundle.sample_lengths` [S2, unofficial].

**Data Structures:**
*   `bins`: A 1D array of size $k+1$ containing the boundaries of the stratified intervals.
*   `samples`: A 1D array of size $k$ containing the sampled distances along the ray.

**Steps:**
1.  **Define Interval:** Set the integration interval $[t_n, t_f]$ [S3, unofficial].
2.  **Create Bin Boundaries:** Divide the interval into $k$ equal parts. Compute boundaries:
    $$ t_i = t_n + i \cdot \frac{t_f - t_n}{k}, \quad \text{for } i = 0, \dots, k $$
    Let $\text{bins} = [t_0, t_1, \dots, t_k]$ [S3, unofficial].
3.  **Extract Bounds:** Identify lower and upper bounds for each bin $i$:
    $$ \text{lower}_i = t_{i-1}, \quad \text{upper}_i = t_i, \quad \text{for } i = 1, \dots, k $$
    [S3, unofficial].
4.  **Sample within Bins:** For each bin $i$, generate a sample $\tau_i$.
    *   **Randomized (Unbiased):** Draw a uniform random number $u_i \sim U[0, 1]$. Compute:
        $$ \tau_i = \text{lower}_i + u_i \cdot (\text{upper}_i - \text{lower}_i) $$
        [S3, unofficial].
    *   **Deterministic (Midpoint):** Use the center of the bin:
        $$ \tau_i = 0.5 \cdot (\text{lower}_i + \text{upper}_i) $$
        [S3, unofficial].
5.  **Store Samples:** Store the distances $\tau_i$ in `RayBundle.sample_lengths` and compute the 3D positions $\mathbf{p}_i = \mathbf{o} + \tau_i \mathbf{d}$ (where $\mathbf{o}$ is ray origin and $\mathbf{d}$ is direction) in `RayBundle.sample_points` [S2, unofficial].
6.  **Compute Radiance Estimate:** Calculate the stratified estimate using the formula below [S1, unofficial].

**Formula:**
The stratified estimate $\hat{C}_{\text{IW}}(\mathbf{r})$ is defined as [S1, unofficial]:
$$ \hat{C}_{\text{IW}}(\mathbf{r}) = \sum_{i=1}^k (t_i - t_{i-1}) c_{\mathbf{r}}(\tau_i) \frac{dF_{\mathbf{r}}}{dt} \bigg|_{t=\tau_i} $$
Where:
*   $k$ is the number of samples.
*   $t_i$ are the bin boundaries defined in Step 2.
*   $c_{\mathbf{r}}(\tau_i)$ is the color (or radiance contribution) at sample point $\tau_i$.
*   $\frac{dF_{\mathbf{r}}}{dt} \big|_{t=\tau_i}$ is the derivative of the transmittance (or related distribution function) with respect to distance at $\tau_i$.
*   $\tau_i \sim U[t_{i-1}, t_i]$ are independent uniform bin samples [S1, unofficial].

**Parameters:**
*   $k$ (number of samples): Typical values vary by scene complexity. In a 'Foggy' field experiment, $k=32$ matched the performance of $k=256$ uniform samples. In a 'Wall' field experiment, only $k=4$ samples were needed to match the baseline [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure $t_n$ and $t_f$ are correctly defined to avoid degenerate bins (zero width).
*   **Unbiasedness:** The randomized version is unbiased. The deterministic midpoint version is biased but may be used for stability or specific variance reduction properties if bias is acceptable [S3, unofficial].
*   **Degenerate Distributions:** The method is specifically noted to be effective for degenerate distributions where uniform sampling performs poorly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stratified_monte_carlo_radiance_estimate(ray_origin, ray_direction, near, far, k, sample_color_func, sample_transmittance_deriv_func):
    """
    Computes the stratified Monte Carlo radiance estimate.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        near: float, near clipping plane distance.
        far: float, far clipping plane distance.
        k: int, number of samples.
        sample_color_func: function(t) -> color, returns color at distance t.
        sample_transmittance_deriv_func: function(t) -> float, returns dF/dt at distance t.
        
    Returns:
        radiance_estimate: float or color, the estimated radiance.
        sample_points: list of 3D vectors, positions of samples.
        sample_lengths: list of floats, distances of samples from origin.
    """
    # Step 1 & 2: Create bin boundaries
    # bins = linspace(near, far, k + 1)
    step_size = (far - near) / k
    bins = [near + i * step_size for i in range(k + 1)]
    
    radiance_estimate = 0.0
    sample_points = []
    sample_lengths = []
    
    # Step 3, 4, 5 & 6: Sample within bins and compute estimate
    for i in range(1, k + 1):
        lower = bins[i - 1]
        upper = bins[i]
        
        # Randomized sampling (unbiased)
        # u_i = random.uniform(0, 1)
        # tau_i = lower + u_i * (upper - lower)
        
        # Deterministic midpoint sampling (biased)
        tau_i = 0.5 * (lower + upper)
        
        # Store sample lengths and points
        sample_lengths.append(tau_i)
        sample_points.append(ray_origin + ray_direction * tau_i)
        
        # Compute contribution
        # c_r(tau_i)
        color = sample_color_func(tau_i)
        # dF_r/dt |_{t=tau_i}
        transmittance_deriv = sample_transmittance_deriv_func(tau_i)
        
        # Bin width
        bin_width = upper - lower
        
        # Add to estimate
        # radiance_estimate += bin_width * color * transmittance_deriv
        radiance_estimate += bin_width * color * transmittance_deriv
        
    return radiance_estimate, sample_points, sample_lengths
```

## Subtopics

### [Antithetic Sampling](7196-antithetic-sampling.md)
Antithetic sampling is a variance reduction technique for estimating the derivative of a BSDF with respect to scene parameters, specifically when the integrand approximates an odd function. It exploits the symmetry of Normal Distribution Functions (NDFs) to make the derivative term an odd function, allowing paired samples to cancel out noise. It sets itself apart from general stratified sampling by targeting the specific symmetry of microfacet derivatives, whereas its sibling [Decomposed sampling](1374-decomposed-sampling.md) addresses cases where derivatives are even functions.

### Ratio Estimator with Equidistant Sampling (REES)
REES is a stochastic integrator that distributes samples at equal distance, jittered by noise, combined with a ratio estimator using analytic unoccluded radiance [S4]. It uses the basic framework of stratified/equidistant sampling paired with analytic unoccluded radiance computation [S4]. REES yields better convergence in unoccluded parts of the scene but leads to overall poor distribution of samples in occluded parts compared to MCWVDS [S4].

### Sample Trimmed Mean Estimator
This technique reduces variance from rare outliers in importance sampling by using a sample trimmed mean over each pixel, discarding extreme samples [S5]. It stores multiple independent, intermediate (low sample count) results. For each pixel, it discards the top 1-5% and bottom 1-5% of samples, then averages the remaining samples [S5]. The parameter is to discard 1% to 5% of the highest and lowest samples per pixel [S5]. A pitfall is that rare outliers can be generated through occasional bad sampling decisions; the trimmed mean trades a slight bias for a large reduction in variance, reducing overall error. The estimator remains consistent because the distribution approaches Gaussian for large sample counts, making the bias zero for symmetric distributions [S5].

## Operating characteristics
*   **Time Complexity:** $O(k)$ per ray, where $k$ is the number of samples. Generating bins and sampling is linear in $k$.
*   **Memory Complexity:** $O(k)$ to store sample points and lengths per ray [S2, unofficial].
*   **Measured Performance:** In a 'Foggy' field experiment, $k=32$ stratified samples matched the performance of $k=256$ uniform samples. In a 'Wall' field experiment, $k=4$ stratified samples matched the baseline [S1, unofficial]. This indicates a significant variance reduction, allowing for fewer samples to achieve the same quality.
*   **Strengths:** Unbiased (when randomized), significantly lower variance than uniform sampling for degenerate distributions, efficient convergence in unoccluded regions [S1, unofficial][S4].
*   **Limitations:** Performance degrades in occluded parts of the scene compared to whole-volume distance sampling methods like MCWVDS [S4]. Requires computation of transmittance derivatives [S1, unofficial].
*   **When to use:** When variance reduction is critical and the distribution is degenerate or complex, and computational budget is limited.
*   **When not to use:** In highly occluded scenes where whole-volume distance sampling (MCWVDS) is superior [S4].

## Background
The Stratified Monte Carlo Radiance Estimate was introduced in the context of differentiable rendering with reparameterized volume sampling [S1, unofficial]. It builds on the concept of inverse transform sampling, noting that it requires only a few points for degenerate distributions [S1, unofficial]. The method was compared against baseline uniform estimates and inverse transform sampling, showing consistent variance improvement [S1, unofficial].

## Sources
[S1] Differentiable Rendering with Reparameterized Volume Sampling, https://doi.org/10.48550/arxiv.2302.10970, fetched 2026-10-26 (unofficial)
[S2] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S3] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S4] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
[S5] A forward scattering dipole model from a functional integral approximation, https://doi.org/10.1145/3072959.3073681, fetched 2026-10-26
