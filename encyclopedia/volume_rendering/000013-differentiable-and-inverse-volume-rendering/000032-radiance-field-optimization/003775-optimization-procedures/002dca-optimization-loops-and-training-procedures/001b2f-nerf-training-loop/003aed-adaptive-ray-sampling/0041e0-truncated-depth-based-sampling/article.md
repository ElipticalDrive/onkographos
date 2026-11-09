# Truncated Depth-Based Sampling


## Overview
Truncated Depth-Based Sampling (TDBS) is a ray sampling strategy that utilizes monocular depth priors informed by a truncated normal distribution to optimize ray point sampling, improving convergence speed and accuracy of pose estimation in joint NeRF and camera pose optimization [S1]. It sets itself apart from alternatives like NoPe-NeRF by explicitly modeling the probability of surface location around the depth prior using a truncated normal distribution, rather than merely considering overall depth distortion [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of input images, camera poses (initially unknown or coarse), and a pre-trained monocular depth estimation network (DPT) [S1].
*   **Outputs:** Sampled 3D points along rays, which are fed into a radiance field MLP to estimate color $c$ and density $\sigma$, resulting in rendered RGB and depth images [S1].

**Data Structures:**
*   **Depth Map:** A 2D array of depth values estimated by DPT for each pixel [S1].
*   **Ray:** Defined as $r(t) = o + td$, where $o$ is the camera origin and $d$ is the viewing direction [S1].
*   **Sampling Points:** A set of $N$ points along the ray, where $N=128$ [S1].

**Parameters:**
*   **Truncation Interval $[a, b]$:** Set to $[0.001, 1.0]$ [S1].
*   **Variance $\bar{\sigma}$:** Set to $0.1$ [S1].
*   **Number of Samples:** 128 points per ray [S1].

**Method Steps:**
1.  **Depth Prior Estimation:** Use a pre-trained monocular depth network (DPT) to estimate a coarse depth map $D$ for each input image [S1].
2.  **Ray Casting:** For each pixel, cast a camera ray $r(t) = o + td$ into the scene [S1].
3.  **Truncated Normal Distribution Definition:** Define the probability density function (PDF) $\psi(\bar{\mu}, \bar{\sigma}, a, b; x)$ for sampling points along the ray, where $\bar{\mu}$ is the depth value from the prior, $\bar{\sigma}$ is the variance, and $[a, b]$ is the truncation interval [S1]:
    $$
    \psi(\bar{\mu}, \bar{\sigma}, a, b; x) = 
    \begin{cases} 
    0 & x \le a \\
    \frac{\phi(\bar{\mu}, \bar{\sigma}^2; x)}{\Phi(\bar{\mu}, \bar{\sigma}^2; b) - \Phi(\bar{\mu}, \bar{\sigma}^2; a)} & a < x < b \\
    0 & b \le x
    \end{cases}
    $$
    where $\phi(\cdot)$ and $\Phi(\cdot)$ denote the probability density and distribution functions of the normal distribution with mean $\bar{\mu}$ and variance $\bar{\sigma}^2$ [S1].
4.  **Inverse Distribution Sampling:** Use inverse distribution sampling to obtain $N$ sampling points along the ray from the truncated normal distribution defined by $\bar{\mu}$, $\bar{\sigma}$, and $[a, b]$ [S1].
5.  **Radiance Field Query:** Feed the sampled points into an MLP $F_\theta$ to estimate color $c$ and volume density $\sigma$ [S1].
6.  **Volume Rendering:** Render RGB and depth images by integrating color $c$ and density $\sigma$ via volume rendering [S1]. The color $C(r)$ is computed as:
    $$
    C(r) = \int_{t_n}^{t_f} T(t)\sigma(r(t))c(r(t), d)dt
    $$
    where $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s))ds\right)$ is the accumulated transmittance [S1].
7.  **Optimization:** Optimize the radiance field parameters $\theta$ and camera poses by supervising the rendered depth and RGB against target images [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.stats import norm

def truncated_normal_pdf(x, mu, sigma, a, b):
    """
    Calculate the PDF of a truncated normal distribution.
    """
    # Standardize
    z_a = (a - mu) / sigma
    z_b = (b - mu) / sigma
    
    # Calculate normalization constant
    # Phi is CDF, phi is PDF
    norm_const = norm.cdf(z_b) - norm.cdf(z_a)
    
    if norm_const == 0:
        return 0.0
        
    # Calculate unnormalized PDF
    pdf_val = norm.pdf((x - mu) / sigma) / sigma
    
    return pdf_val / norm_const

def sample_truncated_normal(mu, sigma, a, b, num_samples):
    """
    Sample points from a truncated normal distribution using inverse CDF.
    """
    # Calculate CDF bounds
    z_a = (a - mu) / sigma
    z_b = (b - mu) / sigma
    
    cdf_a = norm.cdf(z_a)
    cdf_b = norm.cdf(z_b)
    
    # Generate uniform random numbers in [cdf_a, cdf_b]
    u = np.random.uniform(cdf_a, cdf_b, num_samples)
    
    # Inverse CDF (PPF)
    # We need to map u back to the truncated domain
    # Since u is already in the truncated CDF range, we can use the standard PPF
    # and then scale/shift? No, PPF takes the probability value.
    # The PPF of the standard normal gives z, then x = mu + z*sigma.
    # However, norm.ppf(u) works directly if u is the CDF value.
    z_samples = norm.ppf(u)
    x_samples = mu + z_samples * sigma
    
    return x_samples

def truncated_depth_based_sampling(ray_origin, ray_direction, depth_prior, 
                                   sigma_var=0.1, interval_a=0.001, interval_b=1.0, 
                                   num_samples=128):
    """
    Generate sample points along a ray using Truncated Depth-Based Sampling.
    
    Args:
        ray_origin: 3D point (o)
        ray_direction: Normalized 3D vector (d)
        depth_prior: Scalar depth value from monocular prior (mu_bar)
        sigma_var: Variance of the truncated normal distribution (sigma_bar)
        interval_a: Lower bound of truncation
        interval_b: Upper bound of truncation
        num_samples: Number of points to sample
        
    Returns:
        points: Array of 3D points along the ray
    """
    # The depth prior is assumed to be the mean mu_bar of the distribution
    mu_bar = depth_prior
    
    # Sample depths along the ray direction
    # Note: The depth prior is a scalar distance along the ray.
    # We sample t values from the truncated normal distribution.
    t_samples = sample_truncated_normal(mu_bar, sigma_var, interval_a, interval_b, num_samples)
    
    # Convert 1D samples to 3D points
    # r(t) = o + t * d
    points = ray_origin + np.outer(t_samples, ray_direction)
    
    return points
```

**Edge Cases and Pitfalls:**
*   **Coarse Depth Inaccuracy:** Pre-trained monocular depth networks (e.g., DPT) produce coarse depth maps that may be inaccurate or incorrect in fine details. Using TDBS directly in a full-stage manner with such erroneous priors can degrade rendering quality [S1].
*   **Local Minima:** Relying solely on TDBS with inaccurate priors may cause the optimization to fall into local minima [S1]. This is mitigated by the Coarse-to-Fine Training Strategy, which alternates between depth-prior-guided sampling and uniform sampling [S1].

## Subtopics

### [Coarse-to-Fine Training Strategy](10617-coarse-to-fine-training-strategy.md)
A training strategy that progressively improves depth precision to circumvent local minima and refine depth geometry. It sets itself apart by alternating between depth-prior-guided sampling and uniform sampling, whereas TDBS relies solely on truncated normal distributions throughout training. This alternation accelerates convergence and enhances pose estimation accuracy.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** TD-NeRF with TDBS demonstrates superior performance in the joint optimization of camera pose and NeRF, surpassing prior works on three datasets [S1].
*   **Strengths:** Improves convergence speed and accuracy of pose estimation by fully exploiting monocular depth priors [S1].
*   **Limitations:** Sensitive to the accuracy of the coarse depth prior; erroneous priors can degrade rendering quality if not managed via a coarse-to-fine strategy [S1].
*   **When to Use:** When joint optimization of camera pose and NeRF is required, and a monocular depth prior is available.
*   **When Not to Use:** When high-precision depth priors are not available and a coarse-to-fine strategy is not employed, as inaccurate priors may lead to poor results.

## Background
*   **Origin:** Proposed in TD-NeRF by [S1] (2024).
*   **History:** TDBS was introduced to address the challenge of simultaneously optimizing neural radiance fields and camera poses without given camera poses in indoor and outdoor scenes [S1]. It builds on the idea that coarse depth maps can provide a prior for ray sampling, assuming the real surface is near the estimated depth [S1].
*   **Usage:** Used in TD-NeRF for joint camera pose and neural radiance field optimization [S1].

## Sources
[S1] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
