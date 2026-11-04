# Integrated Positional Encoding

Also called: Mip-NeRF Positional Encoding, anti-aliased positional encoding

## Overview
Integrated Positional Encoding (IPE) is a feature representation that generalizes standard positional encoding to allow a region of space to be compactly featurized, rather than encoding a single point [S1]. It solves the aliasing artifacts that occur in neural radiance fields when scene content is observed at multiple resolutions by integrating sine and cosine features over an interval (cone) instead of evaluating them at a single point [S1]. This effectively anti-aliases high-frequency features by shrinking them towards zero when the frequency period is small compared to the interval size [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A mean position vector $\mu \in \mathbb{R}^3$ and a covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ representing a multivariate Gaussian approximation of a 3D conical frustum. A set of frequencies $\omega$.
*   **Output:** A vector of integrated Fourier features.

**Data Structures:**
*   **Frequencies:** A set of base frequencies. In the reference implementation, these are defined as $2^i$ for $i \in \{0, \dots, n\_freqs-1\}$ [S2, unofficial].
*   **Covariance:** The diagonal elements of the covariance matrix, $\sigma^2$, are used for the attenuation calculation [S2, unofficial].

**Parameters:**
*   **$n\_freqs$:** Number of frequency octaves. Default value is 16 [S2, unofficial].
*   **$L$:** The maximum frequency for IPE features. Unlike standard PE, $L$ can be set to an extremely large value and never tuned, as IPE naturally suppresses high frequencies [S1].

**Steps:**
1.  Approximate the 3D conical frustum defined by a camera pixel with a multivariate Gaussian distribution [S1].
2.  For each pixel, cast a cone instead of a ray [S1].
3.  Sample $n + 1$ values for $t_k$ along the cone axis [S1].
4.  Compute IPE features for the interval spanning each adjacent pair of sampled $t_k$ values [S1].
5.  For each frequency $\omega$ in the frequency set, compute the expected feature under the isotropic Gaussian assumption [S2, unofficial].

**Formulas:**
The expected value of the sine and cosine features under an isotropic Gaussian with mean $\mu$ and variance $\sigma^2$ is given by [S2, unofficial]:

$$
E[\sin(\omega x)] = \sin(\omega \mu) \exp(-\omega^2 \sigma^2 / 2)
$$

$$
E[\cos(\omega x)] = \cos(\omega \mu) \exp(-\omega^2 \sigma^2 / 2)
$$

Where:
*   $\omega$ is the frequency.
*   $\mu$ is the mean position.
*   $\sigma^2$ is the variance (derived from the covariance matrix diagonal).

**Pseudocode:**
The following Python-like pseudocode implements the feature generation for a batch of positions and covariances.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def integrated_positional_encoding(means, covs_diag, n_freqs=16):
    """
    Compute Integrated Positional Encoding features.
    
    Args:
        means: Tensor of shape (*, 3), mean positions.
        covs_diag: Tensor of shape (*, 3), diagonal of the covariance matrix.
        n_freqs: Number of frequency octaves.
        
    Returns:
        Tensor of shape (*, 3 * 2 * n_freqs), the integrated features.
    """
    # Generate base frequencies: 2^0, 2^1, ..., 2^(n_freqs-1)
    # Note: The reference code uses freq * math.pi, implying freqs are integers
    # representing powers of 2, or the formula incorporates pi differently.
    # Based on [S2, unofficial] code: freqs are likely 2**i.
    freqs = torch.tensor([2**i for i in range(n_freqs)], device=means.device, dtype=means.dtype)
    
    parts = []
    
    for freq in freqs:
        # Calculate omega * pi * mean
        # The reference code uses freq * math.pi * means. 
        # This implies the base frequency is scaled by pi in the argument.
        omega_mean = freq * math.pi * means  # (*, 3)
        
        # Calculate attenuation factor: exp(-omega^2 * sigma^2 / 2)
        # In reference: w2_sigma2 = (freq * math.pi) ** 2 * covs_diag
        w2_sigma2 = (freq * math.pi) ** 2 * covs_diag  # (*, 3)
        decay = torch.exp(-0.5 * w2_sigma2)  # (*, 3)
        
        # Compute sine and cosine features attenuated by the decay factor
        parts.append(decay * torch.sin(omega_mean))
        parts.append(decay * torch.cos(omega_mean))
        
    return torch.cat(parts, dim=-1)
```

**Edge Cases and Pitfalls:**
*   **Precision:** When the variance $\sigma^2$ is large relative to the frequency $\omega$, the decay factor $\exp(-\omega^2 \sigma^2 / 2)$ approaches zero. This is the intended anti-aliasing behavior, but extreme values may lead to underflow if not handled carefully [S2, unofficial].
*   **Cone Approximation:** The method relies on approximating the conical frustum with a Gaussian. This approximation is valid for the purpose of integration but introduces geometric simplification errors [S1].

## Subtopics

### [Off-Axis Positional Encoding](3503-off-axis-positional-encoding.md)
An alternative approach to handling directional information in neural fields.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of frequency octaves $n\_freqs$ and the batch size. The computation involves basic trigonometric and exponential operations per frequency.
*   **Memory Complexity:** Linear with respect to $n\_freqs$ and the batch size, storing the resulting feature vectors.
*   **Performance:** Mip-NeRF using IPE reduces error rates relative to standard NeRF by 60% on average on a challenging multiresolution benchmark [S1].
*   **Strengths:** Eliminates the need to tune the maximum frequency hyperparameter $L$; effectively handles multi-resolution scenes by suppressing high frequencies when the sampling interval is large [S1].
*   **Limitations:** Requires casting cones and sampling multiple points along the axis, which is more complex than standard ray casting [S1]. The Gaussian approximation of the cone frustum is an approximation [S1].

## Background
Integrated Positional Encoding was introduced in the Mip-NeRF paper by Barron et al., presented at ICCV 2021 [S2, unofficial]. It was developed to address the aliasing artifacts in NeRF renderings that occur when training and testing images observe scene content from different distances or resolutions [S1]. The method generalizes the positional encoding used in NeRF by integrating the features over a region of space rather than evaluating them at a single point [S1].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S2] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
