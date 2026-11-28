# Stratified Sampling
Also called: Discrete Volume Rendering, Fragmental Approximate Volume Rendering Method, Stratified Sampling for Volume Rendering
## Overview
Stratified Sampling partitions the ray depth interval $[t_n, t_f]$ into $N$ equal-sized bins and generates one sample per bin to approximate the continuous volume rendering integral [S1]. It sets itself apart from its parent [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by guaranteeing coverage of the entire depth range without clustering or gaps, and from its siblings by providing a structured variance reduction technique for the volume rendering integral [S1, S3].
## How it works
**Inputs:** ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, near depth $t_n$, far depth $t_f$, number of samples $N$.
**Outputs:** sampled 3D positions $\mathbf{p}_i$, densities $\sigma_i$, colors $c_i$, transmittance $T_i$, opacity $\alpha_i$, accumulated color $\hat{C}(\mathbf{r})$.
**Data structures:** Arrays of size $N+1$ for bin boundaries, size $N$ for samples, transmittance, and opacity. All values are stored as floating-point scalars or vectors depending on batch dimension.

**Numbered steps:**
1. Compute $N+1$ uniform parameter values $t_{\text{vals}} \in [0, 1]$.
2. Map parameters to physical distances $z_{\text{vals}}$ via linear interpolation between $t_n$ and $t_f$.
3. Compute midpoints of adjacent intervals.
4. Define lower and upper boundaries for each of the $N$ bins.
5. Generate $N$ independent uniform random numbers $u_i \sim U[0, 1]$.
6. Perturb sample positions within each bin: $z_i = \text{lower}_i + u_i (\text{upper}_i - \text{lower}_i)$.
7. Compute 3D positions: $\mathbf{p}_i = \mathbf{o} + z_i \mathbf{d}$.
8. Query neural field or volume for density $\sigma_i$ and color $c_i$ at $\mathbf{p}_i$.
9. Compute step sizes $\delta_i = z_{i+1} - z_i$ (or $t_{i+1} - t_i$ in parameter space) [S1].
10. Compute opacity $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and transmittance $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1].
11. Accumulate color: $\hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i c_i$ [S1].

**Formulas:**
$$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i c_i $$
$$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
$$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
$$ \delta_i = t_{i+1} - t_i $$
where $\sigma_i$ is the density at sample $i$, $c_i$ is the color contribution, and $\delta_i$ is the distance between neighboring sampling points [S1].

**Parameters:**
- $N$: Number of sampling points. Typical values range from 32 to 128 depending on scene complexity and hardware constraints [S3].
- $\delta_i$: Distance between neighboring sampling points along the ray [S1].

**Edge cases and pitfalls:**
- When $N=1$, stratification degenerates to uniform sampling [S1].
- If $\sigma_i \delta_i$ exceeds $\approx 700$, $\exp(-\sigma_i \delta_i)$ underflows to zero, causing $\alpha_i$ to saturate at 1 and $T_i$ to drop to 0 prematurely [G].
- Boundary precision errors can cause gaps or overlaps if $z_{\text{vals}}$ are not computed with consistent interpolation [S2, unofficial].
- Degenerate rays (zero direction magnitude) must be normalized before sampling [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stratified_sample(ray_origin, ray_direction, t_min, t_max, N):
    # 1. Compute uniform parameter values
    t_vals = [i / N for i in range(N + 1)]
    
    # 2. Map to physical distances
    z_vals = [t_min * (1.0 - t) + t_max * t for t in t_vals]
    
    # 3. Compute midpoints
    mids = [0.5 * (z_vals[i] + z_vals[i + 1]) for i in range(N)]
    
    # 4. Define interval boundaries
    lower = [z_vals[0]] + mids[:-1]
    upper = mids[1:] + [z_vals[-1]]
    
    # 5. Generate random noise
    import random
    u = [random.uniform(0.0, 1.0) for _ in range(N)]
    
    # 6. Sample within intervals
    z_samples = [lower[i] + u[i] * (upper[i] - lower[i]) for i in range(N)]
    
    # 7. Compute 3D positions
    pts = [(ray_origin[0] + z * ray_direction[0],
            ray_origin[1] + z * ray_direction[1],
            ray_origin[2] + z * ray_direction[2]) for z in z_samples]
    
    # 8. Query densities and colors (placeholder for neural field/volume query)
    # sigma = [query_density(p) for p in pts]
    # color = [query_color(p) for p in pts]
    
    # 9. Compute step sizes
    delta = [z_vals[i + 1] - z_vals[i] for i in range(N)]
    
    # 10. Compute opacity and transmittance
    alpha = [1.0 - math.exp(-sigma[i] * delta[i]) for i in range(N)]
    T = [math.exp(-sum(sigma[j] * delta[j] for j in range(i))) for i in range(N)]
    
    # 11. Accumulate color
    C = sum(T[i] * alpha[i] * color[i] for i in range(N))
    return pts, sigma, color, T, alpha, C
```
## Subtopics

### [Stratified Monte Carlo Radiance Estimate](1373-stratified-monte-carlo-radiance-estimate.md)
An unbiased radiance estimation technique that reduces variance compared to uniform sampling, particularly effective for degenerate distributions. It sets itself apart from its sibling [Batch Stratified Sampling](2730-batch-stratified-sampling.md) by focusing on the specific integration of radiance along a single ray's path segment using stratified bins, and from [Stratified Ray Sampling](3014-stratified-ray-sampling.md) by operating on the distance parameter $t$ along the ray rather than stratifying the ray origins or directions themselves. Choose this variant when precise per-ray radiance integration is required for physically accurate synthesis.

### [Batch Stratified Sampling](2730-batch-stratified-sampling.md)
Partitions the ray depth interval $[t_n, t_f]$ into $N_c$ evenly-spaced bins and perturbs sample positions within each bin to reduce aliasing artifacts during volume rendering. It builds on [Stratified Sampling](17389-stratified-sampling.md) by vectorizing the bin boundary and sampling operations to process multiple rays simultaneously. Unlike pure uniform sampling, it ensures one sample per stratum, avoiding clustering and gaps. Choose this variant for GPU-accelerated neural rendering pipelines where batch processing throughput is critical.

### [Stratified Ray Sampling](3014-stratified-ray-sampling.md)
Discretizes the integration interval along a camera ray by dividing it into equal-sized bins and drawing one sample uniformly at random from each bin. It is used to approximate acoustic integrals and to capture scene structure during the coarse training stage of neural radiance fields. Choose this variant when coarse scene structure capture or acoustic simulation is the primary objective.

### Ray Sampling Strategies
Compares surface rendering (PSNR 28.5), uniform sampling with 64 samples (PSNR 29.6), and proposed stratified sampling with 33 samples (PSNR 31.6) on novel view synthesis [S3]. Surface rendering produces severe geometric artifacts, while uniform sampling produces stratified artifacts due to discretized sampling [S3]. The proposed stratified approach avoids both artifacts and achieves better results with fewer samples [S3]. Choose this configuration when balancing reconstruction quality and sample efficiency is required.

### Importance Sampling via PDF Inversion
Computes weights for PDF estimation, normalizes them to form a PDF, computes the cumulative distribution function (CDF), pads the CDF with a zero, generates uniform samples $u$, finds corresponding indices via searchsorted, clamps denominators to prevent division by zero, and interpolates sample positions within selected bins [S2, unofficial]. Choose this variant when scene density is highly non-uniform and adaptive sample placement is needed to concentrate queries in high-density regions.

## Operating characteristics
**Time and memory complexity:** $O(N)$ time per ray and $O(N)$ memory for storing samples, transmittance, and opacity arrays [S1]. The algorithm is highly parallelizable, scaling linearly with the number of rays $R$ to $O(N \cdot R)$ time and memory when vectorized [S3].
**Measured performance:** On novel view synthesis benchmarks, stratified sampling with 33 samples achieves a PSNR of 31.6, outperforming uniform sampling with 64 samples (PSNR 29.6) and surface rendering (PSNR 28.5) [S3].
**Scaling:** Performance scales linearly with $N$; increasing $N$ reduces variance and aliasing at the cost of higher compute and memory bandwidth [S1].
**Strengths:** Guarantees coverage of the integration interval, reduces variance and aliasing compared to pure uniform random sampling, and is computationally lightweight [S1, S3].
**Limitations:** Can still exhibit stratified artifacts if $N$ is too low or bins are too coarse for high-frequency scene details; does not adapt sample density to local scene complexity without PDF inversion [S3].
**When to use:** Standard direct volume ray casting, neural radiance field training, and acoustic integration where uniform coverage and low variance are prioritized [S1].
**When not:** Highly anisotropic or sparse scenes requiring adaptive importance sampling, or when strict unbiased Monte Carlo convergence over many passes is required [S3].
## Background
Originating from the Neural Radiance Field (NeRF) framework, this method was proposed to approximate the continuous volume rendering integral by dividing the ray segment into parts and uniformly randomly sampling within each part [S1]. It addresses the limitation of fixed offline sampling locations by ensuring better coverage of the ray space [S1]. The technique has since been adopted in neural rendering pipelines, acoustic simulation, and batch-optimized GPU implementations [S3].
## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S2] magis-slac/gradoptics, https://github.com/magis-slac/gradoptics, fetched 2026-10-27 (unofficial)
[S3] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
