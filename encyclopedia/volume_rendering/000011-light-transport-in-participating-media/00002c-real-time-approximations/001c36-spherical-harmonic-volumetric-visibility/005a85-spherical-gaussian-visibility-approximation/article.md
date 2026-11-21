# Spherical Gaussian Visibility Approximation

Also called: SG Visibility Approximation, SG-based Visibility Approximation, Visibility Approximation via Spherical Gaussians

## Overview
Spherical Gaussian Visibility Approximation is a real-time technique for approximating the integral of visibility multiplied by a Spherical Gaussian (SG) lobe, yielding another SG [S1]. It sets itself apart from [Spherical Harmonic Volumetric Visibility](1225-spherical-harmonic-volumetric-visibility.md) by using SG basis functions instead of spherical harmonics, which allows for the preservation of the lobe's center and a simple scaling of amplitude to account for occlusion [S1]. This method is particularly effective for handling narrow specular lobes where visibility varies rapidly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D location $\mathbf{x}$, a visibility function $V(\mathbf{x}, \boldsymbol{\omega}_i)$, and an incoming SG lobe defined by center $\boldsymbol{\xi}$, sharpness $\lambda$, and amplitude $\boldsymbol{\mu}$ [S1].
*   **Output:** An approximated SG lobe with the same center $\boldsymbol{\xi}$ and sharpness $\lambda$, but with a scaled amplitude $\gamma \boldsymbol{\mu}$ [S1].

**Data Structures:**
*   **SG Lobe:** Defined by parameters $(\boldsymbol{\xi}, \lambda, \boldsymbol{\mu})$. The SG function is $G(\boldsymbol{\omega}_i; \boldsymbol{\xi}, \lambda, \boldsymbol{\mu}) = \boldsymbol{\mu} e^{\lambda (\boldsymbol{\omega}_i \cdot \boldsymbol{\xi} - 1)}$ [S1, G]. Note: The exact functional form of the SG is not explicitly provided in the source text, but the convolution result implies a standard exponential form where $\lambda$ controls sharpness and $\boldsymbol{\mu}$ is the amplitude [S1, G].
*   **Visibility Query:** A function or data structure that returns $V(\mathbf{x}, \boldsymbol{\omega}_i)$ for a given direction [S1].

**Steps:**
1.  **Sample Directions:** Randomly sample $S$ directions $\boldsymbol{\omega}_k$ within the support of the SG lobe. The source specifies $S=32$ [S1].
2.  **Query Visibility:** For each sampled direction $\boldsymbol{\omega}_k$, query the visibility $V(\mathbf{x}, \boldsymbol{\omega}_k)$ [S1].
3.  **Compute Weighted Average:** Calculate the visibility ratio $\gamma$ using the weighted average of the queried visibility, weighted by the SG value at each sample direction [S1]:
    $$ \gamma = \frac {\sum ^S_{k=1} G(\boldsymbol{\omega }_k; \boldsymbol{\xi }, \lambda, \boldsymbol{\mu}) V(\mathbf{x}, \boldsymbol{\omega }_k)}{\sum ^S_{k=1} G(\boldsymbol{\omega }_k; \boldsymbol{\xi }, \lambda, \boldsymbol{\mu})} $$
    Note: The source formula uses $G(\boldsymbol{\omega}_k)$ in the numerator and denominator. Since the SG amplitude $\boldsymbol{\mu}$ is common to all terms in the sum, it cancels out. Thus, the weights are effectively proportional to $e^{\lambda (\boldsymbol{\omega }_k \cdot \boldsymbol{\xi } - 1)}$ [S1, G].
4.  **Construct Output SG:** The approximated SG has center $\boldsymbol{\xi}$, sharpness $\lambda$, and amplitude $\gamma \boldsymbol{\mu}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def spherical_gaussian_visibility_approximation(x, visibility_func, xi, lambda_val, mu, num_samples=32):
    """
    Approximates the convolution of visibility with an SG lobe.
    
    Args:
        x: 3D location.
        visibility_func: Function V(x, omega) returning visibility in [0, 1].
        xi: SG center direction (unit vector).
        lambda_val: SG sharpness parameter.
        mu: SG amplitude.
        num_samples: Number of random samples S (default 32).
        
    Returns:
        gamma_mu: The scaled amplitude of the output SG.
    """
    numerator = 0.0
    denominator = 0.0
    
    for _ in range(num_samples):
        # Sample a random direction omega_k on the sphere
        # Note: The source does not specify the sampling distribution.
        # Uniform sampling on the sphere is a standard assumption [G].
        omega_k = random_direction_on_sphere()
        
        # Calculate SG weight for this direction
        # G(omega_k) = exp(lambda * (omega_k . xi - 1))
        # Amplitude mu cancels out in the ratio, so we omit it [G]
        dot_product = dot(omega_k, xi)
        sg_weight = math.exp(lambda_val * (dot_product - 1.0))
        
        # Query visibility
        v = visibility_func(x, omega_k)
        
        numerator += sg_weight * v
        denominator += sg_weight
        
    if denominator == 0:
        return 0.0
        
    gamma = numerator / denominator
    return gamma * mu

def random_direction_on_sphere():
    """Generates a uniformly random unit vector on the sphere [G]."""
    u = random.uniform(0, 1)
    v = random.uniform(0, 1)
    theta = 2 * math.pi * u
    phi = math.acos(2 * v - 1)
    x = math.sin(phi) * math.cos(theta)
    y = math.sin(phi) * math.sin(theta)
    z = math.cos(phi)
    return (x, y, z)
```

**Parameters:**
*   $S$: Number of random samples. Typical value: 32 [S1].

**Edge Cases and Pitfalls:**
*   **Specular Components:** For narrow specular lobes, the visibility ratio $\gamma$ must be determined by sampling the specular SG specifically to accurately approximate the final integral [S1].
*   **Denominator Zero:** If the denominator in the $\gamma$ calculation is zero (e.g., if all sampled directions have zero SG weight), the result is undefined. The pseudocode handles this by returning 0 [G].

## Subtopics

### Spherical Gaussian Light-Field and Visibility Representation
A rendering technique that represents light fields and illumination-weighted environment visibility as sets of Spherical Gaussians (SGs), optimized via neural network backpropagation and stored in 2D textures for hardware-accelerated lookups [S2, unofficial]. It computes illumination-weighted environment visibility for each texel by taking the ratio of the pre-convolved environment map with visibility to the pre-convolved environment map without visibility [S2, unofficial]. The visibility field is approximated as a set of SGs, evaluated in the direction of interest, and multiplied by the pre-convolved environment map to estimate reflected light [S2, unofficial]. SG parameters (axis and sharpness) are treated as extra degrees of freedom and optimized via backpropagation to predict the light-field image [S2, unofficial]. Texture resolution for storing SG parameters and light-field data is 128x128 half-float RGB images per direction [S2, unofficial]. This method achieves full HD global-illumination rendering in 1-2ms on an RTX 2080 graphics card for static scenes [S2, unofficial]. It uses fewer SG lobes than previous methods while significantly improving rendering quality and performance for static scenes [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The core approximation step requires $O(S)$ visibility queries, where $S=32$ [S1]. The light-field variant uses neural network inference and texture lookups, achieving 1-2ms for full HD on an RTX 2080 [S2, unofficial].
*   **Memory Complexity:** Depends on the storage of visibility data. The light-field variant uses 128x128 half-float RGB images per direction [S2, unofficial].
*   **Strengths:** Efficient real-time approximation; preserves lobe center; effective for narrow specular lobes [S1].
*   **Limitations:** Approximation error depends on the number of samples $S$ and the sampling distribution [S1]. The light-field variant is limited to static scenes [S2, unofficial].
*   **When to use:** Real-time rendering where visibility varies rapidly (e.g., specular highlights) and SGs are used for lighting representation [S1].
*   **When not to use:** Dynamic scenes with complex, time-varying visibility if the light-field precomputation is not feasible [S2, unofficial].

## Background
The method originates from research on inverse rendering and indirect illumination modeling [S1]. It was developed to efficiently compute the integral of visibility multiplied by SG lobes, supporting real-time rendering of indirect illumination [S1]. The light-field variant was developed to optimize SG parameters for static scenes using neural networks [S2, unofficial].

## Sources
[S1] Modeling Indirect Illumination for Inverse Rendering, https://doi.org/10.1109/cvpr52688.2022.01809, fetched 2026-10-26
[S2] PDF On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27 (unofficial)
