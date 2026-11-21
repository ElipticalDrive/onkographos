# Integrated Positional Encoding

Also called: IPE, Mip-NeRF Encoding, Mip-NeRF Positional Encoding, anti-aliased positional encoding

## Overview
Integrated Positional Encoding (IPE) is a feature representation that generalizes standard positional encoding to allow a region of space to be compactly featurized, rather than encoding a single point in space [S1]. It solves the aliasing artifacts inherent in neural radiance fields when scene content is observed at multiple resolutions or distances by reasoning about the 3D conical frustum defined by a camera pixel [S2]. Unlike [Fourier Positional Encoding](1267-fourier-positional-encoding.md), which evaluates sine and cosine functions at discrete points, IPE computes the closed-form integral of these features over a Gaussian approximation of the frustum, effectively anti-aliasing high-frequency details [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D positions (means $\mu$) and their associated covariance matrices ($\Sigma$) representing the spatial uncertainty or volume of samples along a ray/cone.
*   **Output:** A tensor of integrated Fourier features where high-frequency components are attenuated based on the spatial extent of the input region.

**Data Structures:**
*   **Means ($\mu$):** Tensor of shape $(..., 3)$, representing the center of the Gaussian approximation for each sample interval.
*   **Covariance ($\Sigma$):** Tensor of shape $(..., 3, 3)$ for full covariance, or $(..., 3)$ for diagonal covariance, representing the spread of the Gaussian.
*   **Frequencies ($\omega$):** A set of base frequencies, typically defined as $2^i$ for $i \in [0, n\_freqs-1]$.

**Steps:**
1.  **Gaussian Approximation:** Approximate the 3D conical frustum defined by a camera pixel as a multivariate Gaussian distribution [S1, S2].
2.  **Interval Sampling:** For rendering, sample $n+1$ depth values $t_k$ along the cone axis [S1].
3.  **Feature Integration:** Compute the expected value of the positional encoding over the Gaussian distribution defined by the mean $\mu$ and covariance $\Sigma$ of the interval between adjacent samples [S1].

**Formulas:**
The standard Fourier positional encoding for a coordinate $x$ involves terms like $\sin(\omega \pi x)$ and $\cos(\omega \pi x)$. IPE computes the expectation $E[\gamma(x)]$ where $x \sim \mathcal{N}(\mu, \Sigma)$ [S1].

For an isotropic Gaussian with mean $\mu$ and variance $\sigma^2$, the expected feature for a frequency $\omega$ is:
$$ E[\sin(\omega x)] = \sin(\omega \mu) \exp\left(-\frac{\omega^2 \sigma^2}{2}\right) $$
$$ E[\cos(\omega x)] = \cos(\omega \mu) \exp\left(-\frac{\omega^2 \sigma^2}{2}\right) $$
where $\omega$ is the frequency, $\mu$ is the mean position, and $\sigma^2$ is the variance [S3, unofficial].

In the general multivariate case with covariance $\Sigma$, the attenuation factor is derived from the characteristic function of the Gaussian. For a frequency vector $\mathbf{k}$, the decay is $\exp(-\frac{1}{2} \mathbf{k}^T \Sigma \mathbf{k})$ [S1][S4, unofficial].

**Parameters:**
*   **$L$ (Max Frequency):** The maximum frequency for IPE features. Unlike standard PE, $L$ can be set to an extremely large value and does not require tuning, as high frequencies are naturally suppressed by the integration [S1].
*   **$n\_freqs$:** Number of frequency octaves. Default value is 16 [S3, unofficial].

**Pseudocode:**
The following implementation assumes diagonal covariance (isotropic variance per axis) as described in [S3, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def integrated_positional_encoding(means, covs_diag, n_freqs=16):
    """
    Compute Integrated Positional Encoding for diagonal covariance.
    
    Args:
        means: Tensor of shape (*, 3) representing the mean positions.
        covs_diag: Tensor of shape (*, 3) representing the diagonal variances.
        n_freqs: Number of frequency octaves.
        
    Returns:
        Tensor of shape (*, 3 * 2 * n_freqs) containing the integrated features.
    """
    # Generate base frequencies: 2^0, 2^1, ..., 2^(n_freqs-1)
    # Note: Standard PE often uses base 2. The source [S3, unofficial] uses freq * math.pi,
    # implying the base frequencies are integers 1, 2, 4... or similar scaling.
    # We follow the standard definition where freqs are powers of 2.
    freqs = 2.0 ** torch.arange(0, n_freqs, device=means.device, dtype=means.dtype)
    
    # Expand dimensions for broadcasting
    # means: (*, 3) -> (*, 1, 3)
    # freqs: (n_freqs,) -> (1, n_freqs, 1)
    # covs_diag: (*, 3) -> (*, 1, 3)
    
    means_exp = means.unsqueeze(-2)       # (*, 1, 3)
    freqs_exp = freqs.unsqueeze(0).unsqueeze(-1) # (1, n_freqs, 1)
    covs_exp = covs_diag.unsqueeze(-2)    # (*, 1, 3)
    
    # Calculate the attenuation factor: exp(-0.5 * (omega * sigma)^2)
    # omega corresponds to freqs * pi in the sine/cosine argument if we assume
    # the standard PE definition sin(2^i * pi * x).
    # Source [S3, unofficial] uses: w2_sigma2 = (freq * math.pi) ** 2 * covs_diag
    
    omega = freqs_exp * math.pi           # (1, n_freqs, 1)
    w2_sigma2 = (omega ** 2) * covs_exp   # (*, n_freqs, 3)
    decay = torch.exp(-0.5 * w2_sigma2)   # (*, n_freqs, 3)
    
    # Calculate the phase: omega * x
    phase = omega * means_exp             # (*, n_freqs, 3)
    
    # Compute sine and cosine features
    sin_features = decay * torch.sin(phase) # (*, n_freqs, 3)
    cos_features = decay * torch.cos(phase) # (*, n_freqs, 3)
    
    # Concatenate along the last dimension
    # Output shape: (*, n_freqs * 3 * 2)
    features = torch.cat([sin_features, cos_features], dim=-1)
    
    return features
```

**Edge Cases and Pitfalls:**
*   **Precision:** When the variance $\sigma^2$ is large relative to the frequency $\omega$, the decay term $\exp(-\omega^2 \sigma^2 / 2)$ approaches zero. This is the desired anti-aliasing behavior, but extreme values may lead to underflow if not handled by the floating-point format [S1].
*   **Covariance Definition:** The formula differs significantly between diagonal and full covariance. The diagonal assumption simplifies the computation to element-wise operations but may be less accurate for highly anisotropic frustums [S3, unofficial]. The full covariance variant requires matrix multiplication [S4, unofficial].

## Subtopics

### Integrated Positional Encoding (Full Covariance)
This variant assumes the covariance matrix is diagonal, meaning the variance is independent along each axis. It is computationally efficient and used in the original Mip-NeRF implementation [S3, unofficial]. It is chosen when performance is critical and the anisotropy of the frustum is not extreme.

### Integrated Positional Encoding (Full Covariance)
This variant uses the full $3 \times 3$ covariance matrix to capture correlations between axes. It constructs a basis matrix $B$ of shape $[3, 3L]$ by concatenating scaled identity matrices $[2^i I_3]$ for $i \in [0, L-1]$ [S4, unofficial]. The encoded mean is computed as $y = \mu B$, and the encoded variance is computed by matrix multiplying the covariance $\Sigma$ with the basis $B$, followed by element-wise multiplication with $B$ and summation [S4, unofficial]. It is chosen when high accuracy in anisotropic scenes is required, at the cost of higher computational complexity [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The diagonal variant is $O(N \cdot F \cdot D)$ where $N$ is the number of samples, $F$ is the number of frequencies, and $D$ is the dimension (3). The full covariance variant involves matrix multiplications, increasing complexity to $O(N \cdot F \cdot D^3)$ or $O(N \cdot F \cdot D^2)$ depending on implementation [S4, unofficial].
*   **Memory Complexity:** Proportional to the input size and the number of frequencies. No additional large buffers are required compared to standard PE.
*   **Performance:** Mip-NeRF reduces error rates relative to NeRF by 60% on average on challenging multiresolution benchmarks [S1].
*   **Strengths:** Eliminates the need to tune the maximum frequency hyperparameter $L$; robust to varying camera distances and resolutions; reduces aliasing artifacts [S1].
*   **Limitations:** Requires the computation of covariance matrices for samples, which adds overhead to the sampling step; the full covariance variant is more expensive [S4, unofficial].

## Background
Integrated Positional Encoding was introduced in the paper "Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields" by Barron et al., presented at ICCV 2021 [S3, unofficial]. It was developed to address the limitation of NeRF, which exhibits significant artifacts when training and testing images observe scene content from different distances or resolutions [S1]. The method generalizes the positional encoding used in NeRF by integrating over the spatial extent of the camera pixel's frustum [S2].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
