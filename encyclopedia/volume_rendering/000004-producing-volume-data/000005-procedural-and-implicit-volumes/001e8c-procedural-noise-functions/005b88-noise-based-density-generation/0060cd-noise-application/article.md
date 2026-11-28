# Noise Application

Also called: Poisson/Normal Noise Injection, Poisson Noise Modeling for Projection Data

## Overview
Noise Application simulates realistic detector noise in projection data, primarily for X-ray computed tomography (CT) and point cloud processing. It differs from its siblings, which often focus on procedural density generation or visual effects, by modeling statistical photon counting statistics (Poisson) or electrical noise (Normal/Gaussian) to validate reconstruction algorithms against realistic artifacts [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Projection data (attenuation values on a negative logarithmic scale) or point cloud vertices; noise mode (`'poisson'` or `'normal'`); noise parameter (norm factor or standard deviation).
- **Outputs:** Noisy projection data or displaced point cloud; optional noise vector $e$ for error analysis [S1][S2, S3, unofficial].

**Data Structures:**
- **Projection Data:** 2D array of floating-point values representing line integrals.
- **Point Cloud:** Tensor of shape $(N, 3)$ representing vertex coordinates [S3, unofficial].

**Steps:**

1. **Mode Selection:** Determine if noise is `'poisson'` or `'normal'` [S2, unofficial].

2. **Poisson Noise Application (X-ray Projections):**
   - Convert projection data from the negative logarithmic scale to photon counts. Let $p$ be the projection data. The photon count $c$ is derived via exponentiation: $c = \exp(-p)$ [S1].
   - Apply the floor function to ensure integer values: $c_{int} = \lfloor c \rfloor$ [S1].
   - Sample a Poisson distribution using each pixel's intensity $c_{int}$ as the mean. Let $n_i$ be the sampled value for pixel $i$: $n_i \sim \text{Poisson}(c_{int, i})$ [S1].
   - Scale the sampled data back to the negative logarithmic scale: $p_{noisy} = -\ln(n_i)$ [S1].
   - Calculate the noise vector $e$ as the difference between the noisy and clean projections in the forward projection space, or directly in the log domain depending on implementation specifics. The relative noise level $\rho$ is calculated as:
     $$ \rho = \frac{\|e\|_2}{\|A\bar{x}\|_2} $$
     where $e$ is the noise vector added to the forward projection, $A$ is the forward projection operator, and $\bar{x}$ is the ground truth volume [S1].

3. **Normal Noise Application (General/Point Cloud):**
   - If mode is `'normal'`, sample from a Normal distribution with the input image/points as mean $\mu$ and parameter $\sigma$ as standard deviation: $y \sim \mathcal{N}(\mu, \sigma^2)$ [S2, unofficial].
   - For point clouds, calculate the diagonal length $d$ of the oriented bounding box. Generate random offsets bounded by $\pm d \times \text{coeff}$, where $\text{coeff}$ is sampled uniformly from $[0.003, 0.01]$ [S3, unofficial].
   - Apply the offset to the point cloud vertices [S3, unofficial].

**Parameters:**
- **Noise Levels (Poisson):** 0.25% (high-quality scan baseline), 1% (moderate), 5% (high) [S1].
- **Mode:** String, `'poisson'` or `'normal'` [S2, unofficial].
- **Parameter:** Float, norm factor for Poisson noise or standard deviation for Normal noise [S2, unofficial].
- **Coeff (Point Cloud):** Scaling factor for noise magnitude, typically $[0.003, 0.01]$ [S3, unofficial].

**Pitfalls:**
- Poisson noise modeling assumes a single energy spectrum. Real laboratory X-ray sources provide a wider spectrum of energies, leading to multiple Poisson distributions with different means. Additionally, detectors exhibit electrical (Gaussian) noise, making the single Poisson model an approximation [S1].
- Precision issues may arise when converting between log scale and photon counts for very low intensity values [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.stats import poisson, norm

def apply_noise(projection_data, mode='poisson', param=0.01):
    """
    Apply noise to projection data or point cloud.
    
    Args:
        projection_data: np.ndarray, projection data (log scale) or point cloud (N,3)
        mode: str, 'poisson' or 'normal'
        param: float, noise level or standard deviation
        
    Returns:
        noisy_data: np.ndarray
        noise_vector: np.ndarray (only for Poisson)
    """
    if mode == 'poisson':
        # Convert from negative log scale to photon counts
        photon_counts = np.exp(-projection_data)
        
        # Floor to ensure integers
        photon_counts_int = np.floor(photon_counts)
        
        # Sample Poisson distribution
        noisy_counts = poisson.rvs(photon_counts_int)
        
        # Convert back to negative log scale
        # Handle zero counts to avoid log(0)
        noisy_counts = np.maximum(noisy_counts, 1)
        noisy_projection = -np.log(noisy_counts)
        
        # Calculate noise vector e for relative noise calculation
        # e is the difference in the forward projection space (photon counts)
        noise_vector = noisy_counts - photon_counts_int
        
        return noisy_projection, noise_vector
        
    elif mode == 'normal':
        # Sample from Normal distribution
        noisy_data = norm.rvs(loc=projection_data, scale=param)
        return noisy_data, None
    else:
        raise ValueError("Mode must be 'poisson' or 'normal'")
```

## Subtopics

### Point Cloud Noise Injection
A data augmentation technique that adds random displacement noise to point cloud vertices to simulate misalignment, noise, or outliers for training robustness. It differs from X-ray noise by operating in 3D geometric space rather than projection intensity space. Choose this when training point cloud classification or segmentation models to improve robustness to sensor noise. It uses an oriented bounding box diagonal to scale the noise magnitude [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels or points, as each element is processed independently.
- **Memory Complexity:** $O(N)$ to store the noisy data and noise vector.
- **Measured Performance:** Noise levels of 0.25%, 1%, and 5% are used to simulate high-quality, moderate, and high noise respectively, based on qualitative comparison with real CT scanner data [S1].
- **Strengths:** Provides realistic noise models for validating reconstruction algorithms; simple to implement.
- **Limitations:** Poisson model is an approximation; does not account for multi-energy spectra or electrical noise fully [S1].

## Background
Originated in the context of X-ray computed tomography (CT) reconstruction, where realistic noise models are essential for evaluating reconstruction algorithms. The Poisson noise model is based on the statistical nature of photon counting in X-ray detectors [S1]. The method is implemented in various CT reconstruction frameworks and research papers [S1][S2, unofficial].

## Sources
[S1] Improved dynamic imaging of multiphase flow by constrained tomographic reconstruction., https://doi.org/10.1038/s41598-021-91776-1, fetched 2026-10-26
[S2] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
[S3] pbonazzi/guassian-diffusion, https://github.com/pbonazzi/guassian-diffusion, fetched 2026-10-28 (unofficial)
