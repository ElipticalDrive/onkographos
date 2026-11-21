# Variance Shadow Mapping

Also called: VSM

## Overview
Variance Shadow Mapping (VSM) is a shadow mapping technique that smoothens depth values to calculate the probability of shadows using Chebyshev's inequality, producing soft shadow edges instead of the abrupt 0/1 values of naive shadow mapping [S1]. It sets itself apart from its siblings by trading the binary occlusion test of [Shadow Mapping](2934-shadow-mapping.md) for a statistical bound on visibility, allowing for efficient soft shadows with minimal overhead compared to standard shadow mapping [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A depth map $D$ (or two maps for moments), a query point $z_q$ (depth of the fragment from the light's perspective), and a bias $b$.
*   **Output:** A shadow factor $s \in [0, 1]$ representing the probability that the fragment is lit.

**Data Structures:**
VSM requires storing two depth maps instead of one:
1.  **Mean Map ($M_1$):** Stores the first moment (mean depth) $E[z]$.
2.  **Second Moment Map ($M_2$):** Stores the second moment $E[z^2]$.

These maps are typically stored in floating-point formats (e.g., `GL_RGBA32F` or similar) to preserve precision for the variance calculation.

**Steps:**
1.  **Render Depth Moments:** Render the scene from the light's perspective. For each pixel, compute the mean depth $z$ and the squared depth $z^2$. Store $z$ in $M_1$ and $z^2$ in $M_2$.
2.  **Filtering (Optional but Recommended):** Apply a low-pass filter (e.g., Gaussian blur) to $M_1$ and $M_2$ to anti-alias the shadow edges. Let the filtered values be $\mu = M_1(x,y)$ and $\mu_2 = M_2(x,y)$. [G] If filtering is skipped, $\mu = z$ and $\mu_2 = z^2$ of the nearest depth.
3.  **Calculate Variance:** Compute the variance $\sigma^2$ of the depth distribution at the query location:
    $$ \sigma^2 = \mu_2 - \mu^2 $$
    [G] Clamp $\sigma^2$ to a small positive value (e.g., $10^{-6}$) to prevent division by zero or numerical instability when $\mu_2 \approx \mu^2$.
4.  **Calculate Distance:** Compute the distance between the query depth $z_q$ and the mean depth $\mu$:
    $$ d = z_q - \mu $$
5.  **Apply Bias:** Adjust the distance by a bias term $b$ to prevent self-shadowing artifacts (shadow acne):
    $$ d' = d + b $$
6.  **Compute Probability Bound:** Use Chebyshev's inequality to find the upper bound on the probability that the fragment is in shadow. Let $p$ be the confidence level (typically $p=0.8$ or similar, but often simplified to a fixed threshold in implementations). The standard VSM formula uses a single-sided Chebyshev bound:
    $$ s = \frac{\sigma^2}{\sigma^2 + d'^2} $$
    [S1] This formula provides the probability that the depth is *greater* than $z_q$ (i.e., occluded). If $d' \le 0$, the fragment is definitely lit (or the bias handles it), so $s=0$.
    [G] In practice, if $d' < 0$, return 1.0 (lit). Otherwise, return $\min(1.0, \frac{\sigma^2}{\sigma^2 + d'^2})$.

**Parameters:**
*   **Bias ($b$):** A small positive value to prevent self-shadowing. Typical values depend on the scene scale and depth precision. [G] Common range: $10^{-4}$ to $10^{-2}$ in normalized device coordinates.
*   **Confidence Level ($p$):** The probability threshold. [S1] The source does not specify a fixed $p$, but the formula $\frac{\sigma^2}{\sigma^2 + d'^2}$ corresponds to a specific bound derivation. [G] Standard implementation often omits explicit $p$ and uses the raw bound.

**Edge Cases and Pitfalls:**
*   **Light Leaking:** VSM can suffer from light leaking (shadow appearing where it shouldn't) because Chebyshev's inequality is a loose bound. [S2]
*   **Precision:** Floating-point precision in the depth maps is critical. [S1]
*   **Bias Tuning:** Too little bias causes shadow acne; too much causes "Peter Panning" (shadows detached from objects).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def variance_shadow_mapping(mean_map, second_moment_map, z_query, bias):
    """
    Calculate shadow factor using Variance Shadow Mapping.
    
    Args:
        mean_map: 2D array of mean depths (E[z])
        second_moment_map: 2D array of second moments (E[z^2])
        z_query: float, depth of the fragment from light's perspective
        bias: float, bias to prevent self-shadowing
        
    Returns:
        float: Shadow factor in [0, 1] (0 = lit, 1 = shadow)
    """
    # Retrieve moments at the current pixel (assuming 2D lookup)
    mu = mean_map[current_pixel]
    mu2 = second_moment_map[current_pixel]
    
    # Calculate variance
    sigma2 = mu2 - mu * mu
    
    # Guard against numerical instability
    if sigma2 < 1e-6:
        sigma2 = 1e-6
        
    # Calculate distance with bias
    d = z_query - mu + bias
    
    # If the fragment is behind the mean depth (with bias), it is lit
    if d <= 0:
        return 0.0
        
    # Chebyshev's inequality bound
    # Probability that z > z_query is <= sigma^2 / (sigma^2 + d^2)
    shadow_prob = sigma2 / (sigma2 + d * d)
    
    return min(1.0, shadow_prob)
```

## Subtopics

### Moment Shadow Mapping
Moment Shadow Mapping (MSM) is a differentiable shadow mapping technique that uses higher-order moments (e.g., $n=2$ for 4 moments) to provide a sharper bound to visibility, reducing light leaking compared to VSM [S2]. It sets itself apart from VSM by using a polynomial bound (Chebyshev-Cantelli or similar higher-order bounds) which is tighter than the single-variance bound, resulting in more accurate albedo textures in inverse rendering pipelines [S2]. It is chosen when differentiable rendering requires precise visibility gradients and reduced light leaking. [S2] recommends a bias value of $\alpha = 5 \times 10^{-7}$ for $n=2$, but implementations often use at least twice this value to avoid singular Hankel matrices [S2]. MSM runtime is similar to VSM, but it requires slightly more memory [S2]. A PyTorch AD implementation of MSM may fail to converge due to NaN outputs near singularities, whereas C++/CUDA implementations are stable [S2].

### [Variance Occlusion Maps](17900-variance-occlusion-maps.md)
Variance Occlusion Maps (VOM) approximate the Cumulative Distribution Function (CDF) of voxel values within a spherical region using only the first two statistical moments (mean and variance) to determine the percentage of occluding voxels for ambient occlusion and soft shadows. Unlike naive shadow mapping or standard variance shadow mapping which operate on 2D depth buffers, VOM operates directly on 3D volume data, allowing for spatially varying soft shadows and ambient occlusion that respect the underlying geometry's density distribution. It sets itself apart from its sibling [Variance Shadow Mapping](5686-variance-shadow-mapping.md) by extending the moment-based approximation into volumetric space, enabling occlusion queries against isosurfaces rather than just depth planes.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment for the lookup and calculation, assuming pre-filtered maps. Rendering the depth maps is $O(N)$ where $N$ is the number of pixels.
*   **Memory Complexity:** $2 \times$ the size of a standard depth map (for mean and second moment). [S1]
*   **Measured Performance:** VSM produces soft shadows with minimal overhead compared to standard shadow mapping [S1]. MSM implementation runtime is similar to VSM [S2].
*   **Strengths:** Efficient soft shadows; differentiable (MSM); minimal overhead over hard shadow mapping.
*   **Limitations:** Light leaking (VSM); requires careful bias tuning; loose bounds can cause artifacts.
*   **When to use:** When soft shadows are desired and performance is critical, and light leaking is acceptable or mitigated by other means.
*   **When not to use:** When high-fidelity, artifact-free shadows are required (use ray tracing or MSM with careful tuning).

## Background
Variance Shadow Mapping was introduced as a method to smooth depth values and calculate shadow probabilities using Chebyshev's inequality [S1]. It is selected as a method for shadow generation in pipelines where soft shadows are needed with minimal overhead [S1]. Moment Shadow Mapping is a more recent development in the context of differentiable rendering, aiming to reduce light leaking and provide better gradients for inverse rendering tasks [S2].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S2] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
