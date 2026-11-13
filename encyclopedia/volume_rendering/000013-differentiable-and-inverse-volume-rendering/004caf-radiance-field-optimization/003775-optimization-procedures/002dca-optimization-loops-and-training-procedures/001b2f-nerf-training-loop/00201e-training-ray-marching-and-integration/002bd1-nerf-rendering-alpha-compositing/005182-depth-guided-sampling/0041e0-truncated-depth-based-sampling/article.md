# Truncated Depth-Based Sampling


## Overview
Truncated Depth-Based Sampling (TDBS) is a ray sampling strategy for neural radiance fields that utilizes coarse monocular depth priors to concentrate sample points near estimated surface locations using a truncated normal distribution [S1]. It differs from sibling methods like [Pixel Region and Depth Guided Ray Sampling](8866-pixel-region-and-depth-guided-sampling.md) and [Surface-Based Sampling](11356-surface-based-sampling.md) by explicitly modeling the probability density of surface proximity via a truncated normal distribution, whereas alternatives often rely on uniform sampling within fixed depth intervals or heuristic region masks [S1]. This approach improves convergence speed and pose estimation accuracy in joint optimization scenarios by fully exploiting the potential of monocular depth maps, unlike [NoPe-NeRF](link-to-nopenerf), which considers overall depth distortion but does not fully exploit depth priors for sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images with unknown camera poses; a pre-trained monocular depth estimation network (DPT) [S1]; the radiance field MLP $F_\theta$ [S1].
*   **Outputs:** Rendered RGB images, rendered depth images, and optimized camera poses and radiance field parameters [S1].

**Data Structures:**
*   **Depth Map:** A 2D array of depth values $D(u, v)$ for each pixel $(u, v)$ in the input image, estimated by DPT [S1].
*   **Ray:** Defined as $r(t) = o + td$, where $o$ is the camera origin and $d$ is the viewing direction [S1].
*   **Sampling Points:** A set of 1D distances $t_i$ along the ray, sampled from the truncated normal distribution [S1].
*   **MLP Input:** 3D points $x = r(t_i)$ and viewing directions $d$ [S1].

**Parameters:**
*   **Number of samples per ray:** 128 [S1].
*   **Truncation interval $[a, b]$:** $[0.001, 1.0]$ [S1].
*   **Variance $\bar{\sigma}$:** 0.1 [S1].
*   **Mean $\bar{\mu}$:** The depth value from the coarse monocular depth map for the corresponding pixel [S1].

**Steps:**
1.  **Depth Prior Estimation:** Use a pre-trained monocular depth network (DPT) to estimate a coarse depth map for each input image [S1].
2.  **Ray Casting:** For each pixel in the input image, cast a camera ray $r(t) = o + td$ into the scene [S1].
3.  **Truncated Normal Distribution Definition:** Define the probability density function (PDF) $\psi(\bar{\mu}, \bar{\sigma}, a, b; x)$ for the truncated normal distribution [S1]:
    $$
    \psi(\bar{\mu}, \bar{\sigma}, a, b; x) = 
    \begin{cases} 
    0 & \text{if } x \le a \\
    \frac{\phi(\bar{\mu}, \bar{\sigma}^2; x)}{\Phi(\bar{\mu}, \bar{\sigma}^2; b) - \Phi(\bar{\mu}, \bar{\sigma}^2; a)} & \text{if } a < x < b \\
    0 & \text{if } b \le x
    \end{cases}
    $$
    where $\phi(\cdot)$ and $\Phi(\cdot)$ denote the probability density and cumulative distribution functions of the normal distribution with mean $\bar{\mu}$ and variance $\bar{\sigma}^2$, respectively [S1].
4.  **Inverse Distribution Sampling:** Use inverse distribution sampling to obtain 128 sampling points $t_i$ along the ray from the truncated normal distribution defined by mean $\bar{\mu}$ (the depth value from the prior), variance $\bar{\sigma}=0.1$, and interval $[a, b]=[0.001, 1.0]$ [S1].
5.  **Radiance Field Query:** Feed the sampled 3D points $x = r(t_i)$ and viewing directions $d$ into the MLP $F_\theta$ to estimate color $c$ and volume density $\sigma$ [S1].
6.  **Volume Rendering:** Render RGB and depth images by integrating color $c$ and density $\sigma$ along the ray using standard volume rendering [S1]. The color $C(r)$ is computed as:
    $$
    C(r) = \int_{t_n}^{t_f} T(t)\sigma(r(t))c(r(t), d) dt
    $$
    where $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s)) ds\right)$ is the accumulated transmittance [S1].
7.  **Optimization:** Optimize the radiance field parameters $\theta$ and camera poses by supervising the rendered depth and RGB images against target values [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.stats import norm

def truncated_normal_pdf(x, mu, sigma, a, b):
    """
    Probability density function of a truncated normal distribution.
    """
    # Standardize
    z_a = (a - mu) / sigma
    z_b = (b - mu) / sigma
    z_x = (x - mu) / sigma
    
    # PDF of standard normal
    phi = norm.pdf(z_x)
    
    # CDF of standard normal
    Phi_a = norm.cdf(z_a)
    Phi_b = norm.cdf(z_b)
    
    # Normalization constant
    norm_const = Phi_b - Phi_a
    
    if norm_const == 0:
        return 0.0
        
    if x <= a or x >= b:
        return 0.0
    else:
        return phi / norm_const

def inverse_transform_sampling_truncated_normal(mu, sigma, a, b, num_samples):
    """
    Sample points from a truncated normal distribution using inverse transform sampling.
    """
    # CDF of standard normal
    z_a = (a - mu) / sigma
    z_b = (b - mu) / sigma
    
    Phi_a = norm.cdf(z_a)
    Phi_b = norm.cdf(z_b)
    
    # Generate uniform random numbers in [Phi_a, Phi_b]
    u = np.random.uniform(Phi_a, Phi_b, num_samples)
    
    # Inverse CDF (percent point function) of standard normal
    z_samples = norm.ppf(u)
    
    # Transform back to original distribution
    t_samples = mu + sigma * z_samples
    
    return t_samples

def tr_depth_based_sampling(ray_origin, ray_direction, depth_prior, num_samples=128, a=0.001, b=1.0, sigma=0.1):
    """
    Generate sampling points for a ray using Truncated Depth-Based Sampling.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        depth_prior: float, the depth value from the monocular depth map for the corresponding pixel.
        num_samples: int, number of points to sample along the ray.
        a: float, lower bound of truncation interval.
        b: float, upper bound of truncation interval.
        sigma: float, standard deviation of the normal distribution.
        
    Returns:
        points: Array of 3D points along the ray.
    """
    # Step 1: Sample distances t from truncated normal distribution
    # The mean mu is the depth_prior. The distribution is defined over the interval [a, b].
    t_samples = inverse_transform_sampling_truncated_normal(
        mu=depth_prior, 
        sigma=sigma, 
        a=a, 
        b=b, 
        num_samples=num_samples
    )
    
    # Step 2: Convert 1D distances to 3D points
    # r(t) = o + t * d
    points = ray_origin + t_samples[:, np.newaxis] * ray_direction
    
    return points
```

**Edge Cases and Pitfalls:**
*   **Coarse Depth Inaccuracy:** Pre-trained monocular depth networks (e.g., DPT) produce coarse depth maps that fail to capture fine details and may be inaccurate or incorrect in many regions [S1]. Using these directly in a full-stage manner can degrade rendering quality [S1].
*   **Local Minima:** Relying solely on TDBS with potentially erroneous depth priors can cause the model to fall into local minima during training [S1]. This is mitigated by the [Coarse-to-Fine Training Strategy](10617-coarse-to-fine-training-strategy.md), which alternates between depth-prior-guided sampling and uniform sampling [S1].
*   **Truncation Bounds:** The choice of $[a, b]$ and $\sigma$ affects the concentration of samples. If the depth prior is significantly outside $[a, b]$, the sampling distribution may be truncated to zero or near-zero probability, leading to poor coverage [S1].

## Subtopics

### [Coarse-to-Fine Training Strategy](10617-coarse-to-fine-training-strategy.md)
A training strategy that progressively improves depth precision to circumvent local minima and refine depth geometry. It sets itself apart by alternating between depth-prior-guided sampling and uniform sampling, whereas TDBS relies solely on truncated normal distributions throughout training. This alternation accelerates convergence and enhances pose estimation accuracy.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** TD-NeRF with TDBS achieves superior performance in the joint optimization of camera pose and NeRF, surpassing prior works, and generates more accurate depth geometry [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Improves convergence speed and accuracy of pose estimation by explicitly utilizing monocular depth priors through the truncated normal distribution [S1]. It fully exploits the potential of monocular depth maps, unlike NoPe-NeRF which considers overall depth distortion [S1].
*   **Limitations:** Relies on the quality of coarse monocular depth priors, which may be inaccurate or lack fine details, potentially degrading rendering quality if used without the coarse-to-fine strategy [S1].
*   **When to Use:** When training NeRF from unknown camera poses and monocular depth priors are available, especially when joint optimization of pose and radiance field is required [S1].
*   **When Not to Use:** When high-precision depth priors are not available and the coarse-to-fine strategy cannot be employed, as erroneous depth values may degrade quality [S1].

## Background
Truncated Depth-Based Sampling (TDBS) was proposed in the TD-NeRF paper [S1]. It addresses the challenge of simultaneously optimizing neural radiance fields and camera poses without given camera poses in indoor and outdoor scenes [S1]. The method explicitly utilizes monocular depth priors through three key advancements: TDBS, a coarse-to-fine training strategy, and a Gaussian Point Constraint (GPC) [S1]. The implementation of TD-NeRF has been released [S1].

## Sources
[S1] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
