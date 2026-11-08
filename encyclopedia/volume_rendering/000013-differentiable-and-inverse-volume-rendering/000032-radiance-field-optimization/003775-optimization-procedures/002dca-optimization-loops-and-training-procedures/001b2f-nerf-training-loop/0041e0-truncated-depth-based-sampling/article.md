# Truncated Depth-Based Sampling
Also called: TDBS

## Overview
Truncated Depth-Based Sampling (TDBS) is a ray sampling strategy for neural radiance fields that utilizes coarse monocular depth priors to guide point sampling via a truncated normal distribution, improving convergence speed and pose estimation accuracy in joint optimization scenarios [S1]. It sets itself apart from uniform sampling and methods like NoPe-NeRF by explicitly modeling the probability of the true surface being near the estimated depth, rather than treating depth merely as a distortion constraint or ignoring the prior's distributional nature [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images, camera poses (initially unknown or coarse), and a pre-trained monocular depth estimation network (specifically DPT) [S1].
*   **Outputs:** Sampled 3D points along rays, which are fed into an MLP to estimate color $c$ and density $\sigma$, resulting in rendered RGB and depth images [S1].

**Data Structures:**
*   **Depth Map:** A 2D array of depth values estimated by DPT for each pixel [S1].
*   **Ray:** Defined as $r(t) = o + td$, where $o$ is the camera origin and $d$ is the viewing direction [S1].
*   **Sampling Points:** A list of 128 depth values $t$ sampled per ray [S1].

**Parameters:**
*   **Mean ($\bar{\mu}$):** The depth value from the monocular prior for the current pixel [S1].
*   **Variance ($\bar{\sigma}$):** Set to 0.1 [S1].
*   **Truncation Interval $[a, b]$:** Set to $[0.001, 1.0]$ [S1].
*   **Number of Samples:** 128 points per ray [S1].

**Algorithm Steps:**
1.  **Depth Prior Estimation:** Use a pre-trained monocular depth network (DPT) to estimate a coarse depth map for each input image [S1].
2.  **Ray Casting:** For each pixel in the input image, cast a camera ray $r(t) = o + td$ into the scene [S1].
3.  **Truncated Normal Distribution Definition:** Define the probability density function (PDF) $\psi(\bar{\mu}, \bar{\sigma}, a, b; x)$ for the truncated normal distribution:
    $$
    \psi(\bar{\mu}, \bar{\sigma}, a, b; x) = 
    \begin{cases} 
    0 & x \le a \\
    \frac{\phi(\bar{\mu}, \bar{\sigma}^2; x)}{\Phi(\bar{\mu}, \bar{\sigma}^2; b) - \Phi(\bar{\mu}, \bar{\sigma}^2; a)} & a < x < b \\
    0 & b \le x
    \end{cases}
    $$
    where $\phi(\cdot)$ and $\Phi(\cdot)$ denote the probability density and distribution functions of the normal distribution with mean $\bar{\mu}$ and variance $\bar{\sigma}^2$, respectively [S1].
4.  **Inverse Distribution Sampling:** Use inverse distribution sampling to obtain 128 sampling points along the ray from the truncated normal distribution defined by $\bar{\mu}$, $\bar{\sigma}$, and $[a, b]$ [S1].
5.  **Neural Field Evaluation:** Feed the sampled 3D points into the radiance field MLP $F_\theta$ to estimate color $c$ and volume density $\sigma$ [S1].
6.  **Volume Rendering:** Render RGB and depth images by integrating $c$ and $\sigma$ along the ray using standard volume rendering:
    $$
    C(r) = \int_{t_n}^{t_f} T(t)\sigma(r(t))c(r(t), d)dt
    $$
    where $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s))ds\right)$ is the accumulated transmittance [S1].
7.  **Optimization:** Optimize the radiance field parameters and camera poses by supervising the rendered depth and RGB against target values [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.stats import norm

def truncated_normal_pdf(x, mu, sigma, a, b):
    """
    Calculates the PDF of a truncated normal distribution.
    """
    # Standardize
    z_a = (a - mu) / sigma
    z_b = (b - mu) / sigma
    
    # CDF difference for normalization
    cdf_diff = norm.cdf(z_b) - norm.cdf(z_a)
    
    if cdf_diff == 0:
        return 0.0
        
    # PDF of standard normal
    pdf_val = norm.pdf((x - mu) / sigma) / sigma
    
    return pdf_val / cdf_diff

def sample_truncated_normal(mu, sigma, a, b, num_samples):
    """
    Samples points from a truncated normal distribution using inverse CDF sampling.
    """
    # Calculate CDF at boundaries
    z_a = (a - mu) / sigma
    z_b = (b - mu) / sigma
    
    cdf_a = norm.cdf(z_a)
    cdf_b = norm.cdf(z_b)
    
    # Generate uniform random numbers in the truncated CDF range
    u = np.random.uniform(cdf_a, cdf_b, num_samples)
    
    # Inverse CDF (Percent Point Function)
    # Note: norm.ppf is the inverse of norm.cdf
    samples = norm.ppf(u) * sigma + mu
    
    return samples

def truncated_depth_based_sampling(ray_origin, ray_direction, depth_prior, 
                                   sigma=0.1, a=0.001, b=1.0, num_samples=128):
    """
    Generates sampling points along a ray using TDBS.
    
    Args:
        ray_origin: 3D vector (o)
        ray_direction: 3D vector (d)
        depth_prior: float, the estimated depth value from monocular prior (mu)
        sigma: float, variance parameter (default 0.1)
        a: float, lower bound of truncation (default 0.001)
        b: float, upper bound of truncation (default 1.0)
        num_samples: int, number of points to sample (default 128)
        
    Returns:
        points: Array of 3D points along the ray
    """
    # 1. Sample depths from truncated normal distribution
    # The depth prior serves as the mean (mu)
    sampled_depths = sample_truncated_normal(depth_prior, sigma, a, b, num_samples)
    
    # 2. Convert depth values to 3D points along the ray
    # r(t) = o + t * d
    points = ray_origin + sampled_depths[:, np.newaxis] * ray_direction
    
    return points
```

**Edge Cases and Pitfalls:**
*   **Coarse Depth Inaccuracy:** Pre-trained monocular depth networks (e.g., DPT) produce coarse maps that fail to capture fine details and may be incorrect in specific regions [S1]. Using these directly in a full-stage manner can degrade rendering quality [S1].
*   **Local Minima:** Relying solely on TDBS without a coarse-to-fine strategy may cause the model to fall into local minima during training [S1].

## Subtopics

### [Coarse-to-Fine Training Strategy](10617-coarse-to-fine-training-strategy.md)
A training strategy that progressively improves depth precision to circumvent local minima and refine depth geometry. It sets itself apart by alternating between depth-prior-guided sampling and uniform sampling, whereas TDBS relies solely on truncated normal distributions throughout training. This alternation accelerates convergence and enhances pose estimation accuracy.

## Operating characteristics
*   **Convergence:** TDBS improves convergence speed and accuracy of pose estimation compared to uniform sampling or methods that do not fully exploit depth priors [S1].
*   **Robustness:** It makes depth self-supervising more robust by explicitly modeling the depth prior's distribution [S1].
*   **Limitations:** The quality of sampling is dependent on the accuracy of the coarse monocular depth prior. If the prior is significantly incorrect, the truncated normal distribution may sample points away from the true surface, potentially degrading rendering quality if not mitigated by a coarse-to-fine strategy [S1].
*   **Performance:** The method uses 128 sampling points per ray [S1].

## Background
TDBS was introduced in the TD-NeRF paper as part of a method for jointly optimizing camera poses and neural radiance fields from unknown poses [S1]. It addresses the limitation of previous methods like NoPe-NeRF, which considered overall depth distortion but did not fully exploit the potential of monocular depth maps [S1]. The approach explicitly utilizes monocular depth priors through three key advancements: TDBS, a coarse-to-fine training strategy, and a Gaussian Point Constraint (GPC) [S1].

## Sources
[S1] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
