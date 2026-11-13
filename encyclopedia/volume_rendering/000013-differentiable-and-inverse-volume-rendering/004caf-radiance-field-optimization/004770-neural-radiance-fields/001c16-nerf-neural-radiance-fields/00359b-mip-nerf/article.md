# Mip-NeRF

Also called: Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, MipNeRF, canonical sampling, multum in parvo NeRF

## Overview
Mip-NeRF extends Neural Radiance Fields (NeRF) to represent the scene at a continuously-valued scale, rendering anti-aliased conical frustums instead of infinitesimal rays to reduce aliasing artifacts and improve the representation of fine details [S1]. Unlike traditional mipmapping which uses discrete downsampling scales, Mip-NeRF learns a single neural scene model that can be queried at arbitrary scales, effectively prefiltering the radiance field [S1]. This approach ameliorates the aliasing inherent in NeRF's point-sampling model while simultaneously allowing the merger of the separate "coarse" and "fine" MLPs used in standard NeRF into a single network [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A camera ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a pixel radius $\dot{r}$ at the image plane. The method outputs a color value for the pixel by integrating radiance over a conical frustum.
*   **Output:** A prefiltered pixel color $C(\mathbf{o}, \mathbf{d}, \dot{r})$.

**Data Structures:**
The core data structure is a 3D Gaussian distribution that approximates the conical frustum associated with a sampling interval. This Gaussian is characterized by:
*   Mean distance along the ray: $\mu_t$
*   Variance along the ray: $\sigma_t^2$
*   Variance perpendicular to the ray: $\sigma_r^2$
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (shared for the frustum).

**Parameters:**
*   $\dot{r}$: The radius of the cone at the image plane. It is set to the width of the pixel in world coordinates scaled by $2/\sqrt{12}$ [S1]. This scaling ensures the cone's section on the image plane has a variance in $x$ and $y$ that matches the variance of the pixel's footprint.
*   MLP Configuration: 8 layers with a skip connection concatenating the input to the fifth layer [S2].

**Numbered Steps:**

1.  **Cone Casting:** For each pixel, instead of casting a single ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, cast a cone with apex at $\mathbf{o}$ and direction $\mathbf{d}$. The radius of the cone at distance $t$ is determined by $\dot{r}$.
2.  **Frustum Division:** Divide the cone into a series of conical frustums (cones cut perpendicular to their axis) corresponding to sampling intervals $[t_0, t_1]$ along the ray [S1].
3.  **Gaussian Approximation:** Approximate each conical frustum with a multivariate Gaussian. Compute the mean $\mu_t$ and variances $\sigma_t^2, \sigma_r^2$ for each frustum [S1].
4.  **Integrated Positional Encoding (IPE):** For each frustum, compute the IPE feature. This is the expected positional encoding over the volume of the frustum, approximated using the Gaussian parameters [S1].
5.  **Neural Query:** Query the single MLP with the IPE features and the view direction to obtain volume density $\sigma$ and color $\mathbf{c}$ [S1].
6.  **Volume Rendering:** Perform standard volume rendering (alpha compositing) using the sampled densities and colors, integrating over the conical frustums [S1].

**Formulas:**

The set of positions $\mathbf{x}$ within a conical frustum between distances $t_0$ and $t_1$ is defined by the indicator function $F$:
$$
F(\mathbf{x}, \mathbf{o}, \mathbf{d}, \dot{r}, t_0, t_1) = \mathbb{1}\left\{ \left( t_0 < \frac{\mathbf{d}^T(\mathbf{x} - \mathbf{o})}{\|\mathbf{d}\|_2^2} < t_1 \right) \wedge \left( \frac{\mathbf{d}^T(\mathbf{x} - \mathbf{o})}{\|\mathbf{d}\|_2 \|\mathbf{x} - \mathbf{o}\|_2} > \frac{1}{\sqrt{1 + (\dot{r}/\|\mathbf{d}\|_2)^2}} \right) \right\}
$$
where $\mathbb{1}\{\cdot\}$ is 1 if the condition is true and 0 otherwise [S1].

The Integrated Positional Encoding (IPE) $\gamma^*$ is the expected positional encoding $\gamma(\mathbf{x})$ over the frustum:
$$
\gamma^*(\mathbf{o}, \mathbf{d}, \dot{r}, t_0, t_1) = \frac{\int \gamma(\mathbf{x}) F(\mathbf{x}, \mathbf{o}, \mathbf{d}, \dot{r}, t_0, t_1) d\mathbf{x}}{\int F(\mathbf{x}, \mathbf{o}, \mathbf{d}, \dot{r}, t_0, t_1) d\mathbf{x}}
$$
Since the numerator has no closed-form solution, the frustum is approximated by a multivariate Gaussian [S1].

The Gaussian parameters for a frustum defined by interval $[t_\mu - t_s, t_\mu + t_s]$ (where $t_\mu$ is the midpoint and $t_s$ is the half-width) are:
$$
\mu_t = t_\mu + \frac{2t_\mu t_s^2}{3t_\mu^2 + t_s^2}
$$
$$
\sigma_t^2 = \frac{t_s^2}{3} - \frac{4t_s^4(12t_\mu^2 - t_s^2)}{15(3t_\mu^2 + t_s^2)^2}
$$
$$
\sigma_r^2 = r^2 \left( \frac{t_\mu^2}{4} + \frac{5t_s^2}{12} - \dots \right)
$$
Note: The formula for $\sigma_r^2$ is truncated in the source [S1]. The term $r$ represents the radius of the cone at the mean distance $t_\mu$, which is $r = \dot{r} \frac{t_\mu}{\|\mathbf{d}\|_2}$ [S1, G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_gaussian_params(t_mu, t_s, r_dot, d_norm):
    """
    Compute Gaussian parameters for a conical frustum.
    
    Args:
        t_mu: Midpoint distance of the frustum along the ray.
        t_s: Half-width of the frustum along the ray.
        r_dot: Radius of the cone at the image plane (world units).
        d_norm: Norm of the ray direction vector ||d||_2.
        
    Returns:
        mu_t: Mean distance along the ray.
        sigma_t_sq: Variance along the ray.
        sigma_r_sq: Variance perpendicular to the ray.
    """
    # Mean distance
    mu_t = t_mu + (2 * t_mu * t_s**2) / (3 * t_mu**2 + t_s**2)
    
    # Variance along the ray
    sigma_t_sq = (t_s**2 / 3) - (4 * t_s**4 * (12 * t_mu**2 - t_s**2)) / (15 * (3 * t_mu**2 + t_s**2)**2)
    
    # Radius of the cone at the mean distance t_mu
    # r = r_dot * (t_mu / ||d||_2)
    r = r_dot * (t_mu / d_norm)
    
    # Variance perpendicular to the ray
    # Formula truncated in source [S1]
    # sigma_r_sq = r**2 * (t_mu**2 / 4 + 5 * t_s**2 / 12 - ...)
    # Implementation note: The full formula is not provided in the sources.
    # A common approximation or the full derivation from the original paper is required here.
    # Placeholder for the missing term from source [S1].
    sigma_r_sq = r**2 * (t_mu**2 / 4 + 5 * t_s**2 / 12) # [G] Placeholder based on visible terms
    
    return mu_t, sigma_t_sq, sigma_r_sq

def integrated_positional_encoding(mu_t, sigma_t_sq, sigma_r_sq, d, o, r_dot, num_freqs=15):
    """
    Compute the Integrated Positional Encoding (IPE) for a frustum.
    
    Args:
        mu_t: Mean distance along the ray.
        sigma_t_sq: Variance along the ray.
        sigma_r_sq: Variance perpendicular to the ray.
        d: Ray direction vector.
        o: Ray origin vector.
        r_dot: Radius of the cone at the image plane.
        num_freqs: Number of frequency bands for positional encoding.
        
    Returns:
        ipe: The integrated positional encoding vector.
    """
    # The IPE is the expected positional encoding over the Gaussian volume.
    # For a Gaussian distribution, the expectation of sin/cos of linear functions
    # can be computed analytically.
    # The positional encoding typically involves sin(2^k * pi * x) for x in [x, y, z].
    # The exact computation of IPE from Gaussian moments is complex and 
    # relies on the property that E[cos(a^T x)] = cos(a^T mu) * exp(-0.5 * a^T Sigma a).
    
    # Construct the Gaussian covariance matrix Sigma in 3D space.
    # The Gaussian is aligned with the ray direction d.
    # Sigma = sigma_r_sq * (I - d_hat * d_hat^T) + sigma_t_sq * (d_hat * d_hat^T)
    # where d_hat = d / ||d||
    
    d_hat = d / np.linalg.norm(d)
    I = np.eye(3)
    Sigma = sigma_r_sq * (I - np.outer(d_hat, d_hat)) + sigma_t_sq * np.outer(d_hat, d_hat)
    
    # The mean position of the frustum is mu_pos = o + mu_t * d_hat
    mu_pos = o + mu_t * d_hat
    
    ipe = []
    for k in range(num_freqs):
        freq = 2**k * np.pi
        # Basis functions for positional encoding: sin and cos for x, y, z
        # We need E[cos(freq * (mu_pos + delta)^T e_i)] and E[sin(...)]
        # Using the Gaussian expectation formula:
        # E[cos(a^T x)] = cos(a^T mu) * exp(-0.5 * a^T Sigma a)
        # E[sin(a^T x)] = sin(a^T mu) * exp(-0.5 * a^T Sigma a)
        
        for i in range(3):
            a = np.zeros(3)
            a[i] = freq
            # a^T Sigma a
            a_Sigma_a = a @ Sigma @ a
            exp_term = np.exp(-0.5 * a_Sigma_a)
            
            # a^T mu
            a_mu = a @ mu_pos
            
            ipe.append(np.cos(a_mu) * exp_term)
            ipe.append(np.sin(a_mu) * exp_term)
            
    return np.array(ipe)

def mipnerf_render(o, d, r_dot, near, far, num_samples, mlp):
    """
    Render a pixel using Mip-NeRF.
    
    Args:
        o: Ray origin.
        d: Ray direction.
        r_dot: Radius of the cone at the image plane.
        near: Near clipping plane distance.
        far: Far clipping plane distance.
        num_samples: Number of sampling intervals.
        mlp: The Mip-NeRF MLP model.
        
    Returns:
        color: Rendered pixel color.
    """
    d_norm = np.linalg.norm(d)
    t_vals = np.linspace(near, far, num_samples + 1)
    t_mus = (t_vals[:-1] + t_vals[1:]) / 2
    t_ss = (t_vals[1:] - t_vals[:-1]) / 2
    
    colors = []
    alphas = []
    
    for i in range(num_samples):
        mu_t, sigma_t_sq, sigma_r_sq = compute_gaussian_params(t_mus[i], t_ss[i], r_dot, d_norm)
        
        # Compute IPE
        ipe = integrated_positional_encoding(mu_t, sigma_t_sq, sigma_r_sq, d, o, r_dot)
        
        # Query MLP
        # The MLP takes IPE and view direction as input
        # Output: density sigma and color c
        sigma, c = mlp(ipe, d)
        
        # Compute alpha for this interval
        # alpha = 1 - exp(-sigma * delta_t)
        # However, for Mip-NeRF, the integration is over the frustum volume.
        # The standard NeRF alpha compositing is adapted.
        # The effective thickness is related to the frustum geometry.
        # For simplicity, we use the standard alpha formula with delta_t = t_ss[i] * 2
        # Note: The exact integration weight for the frustum volume is non-trivial.
        # Standard NeRF uses alpha = 1 - exp(-sigma * delta_t).
        # Mip-NeRF uses the same compositing but with IPE features.
        
        delta_t = t_ss[i] * 2 # Approximation
        alpha = 1 - np.exp(-sigma * delta_t)
        
        colors.append(c)
        alphas.append(alpha)
        
    # Alpha compositing
    # C = sum(T_i * alpha_i * c_i)
    # T_i = product(1 - alpha_j) for j < i
    T = 1.0
    final_color = np.zeros(3)
    
    for i in range(num_samples):
        alpha = alphas[i]
        color = colors[i]
        final_color += T * alpha * color
        T *= (1 - alpha)
        
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Medical Images:** Mip-NeRF's regional (canonical) sampling smooths points, which adversely affects capturing fine details and complex spatial relationships in medical images, leading to lower performance compared to other NeRF variants in this domain [S2].
*   **Precision:** The approximation of the conical frustum with a Gaussian is an approximation; the exact integral has no closed form [S1].
*   **Boundary Conditions:** The radius $\dot{r}$ must be scaled correctly to match the pixel footprint variance [S1].

## Subtopics

### NeRF (Neural Radiance Fields)
The base method that Mip-NeRF extends. Mip-NeRF replaces point sampling with cone sampling and merges coarse/fine MLPs. [NeRF (Neural Radiance Fields)](1185-nerf-neural-radiance-fields.md)

### NeRF Training Configuration
Mip-NeRF uses a single MLP, simplifying the training configuration compared to the coarse/fine setup of NeRF. [NeRF Training Configuration](1203-nerf-training-configuration.md)

### NeRF-SH
Uses spherical harmonics for color. Mip-NeRF uses IPE for spatial encoding. [NeRF-SH](1348-nerf-sh.md)

### Recursive-NeRF
Uses recursive sampling. Mip-NeRF uses integrated encoding. [Recursive-NeRF](3029-recursive-nerf.md)

### NeMF (Neural Microflake Field)
Uses microflakes. Mip-NeRF uses continuous scale representation. [NeMF (Neural Microflake Field)](3387-nemf-neural-microflake-field.md)

### EgoNeRF
For ego-motion. Mip-NeRF is general view synthesis. [EgoNeRF](4096-egonerf.md)

### Intrinsic NeRF
Decomposes intrinsic properties. Mip-NeRF focuses on anti-aliasing. [Intrinsic NeRF](4623-intrinsic-nerf.md)

### Sparse Neural Volume Representation
Uses sparse voxels. Mip-NeRF uses dense MLP. [Sparse Neural Volume Representation](7055-sparse-neural-volume-representation.md)

### Neural Radiance Distribution Field
Models distribution. Mip-NeRF models integrated features. [Neural Radiance Distribution Field](10216-neural-radiance-distribution-field.md)

### Incremental CONfidence (ICON)
Uses confidence. Mip-NeRF uses scale. [Incremental CONfidence (ICON)](11446-incremental-confidence-icon.md)

### Bayesian Uncertainty Quantification for SeaThru-NeRF
Uses Bayesian uncertainty. Mip-NeRF uses deterministic IPE. [Bayesian Uncertainty Quantification for SeaThru-NeRF](11861-bayesian-uncertainty-quantification-for-seathru-nerf.md)

### Control-NeRF
Uses control signals. Mip-NeRF uses geometric scale. [Control-NeRF](13260-control-nerf.md)

## Operating characteristics
*   **Time Complexity:** Mip-NeRF is approximately 7% faster than NeRF [S1]. It matches the accuracy of brute-force supersampled NeRF while being 22x faster [S1].
*   **Memory Complexity:** Mip-NeRF has half the number of parameters of NeRF due to the merger of coarse and fine MLPs [S1].
*   **Measured Performance:** On the original NeRF dataset, Mip-NeRF reduces average error rates by 17% compared to NeRF [S1]. On a challenging multiscale variant dataset, it reduces error rates by 60% [S1].
*   **Strengths:** Reduces aliasing artifacts, improves representation of fine details, allows for single MLP architecture, faster than supersampled NeRF.
*   **Limitations:** In medical image reconstruction, Mip-NeRF consistently shows the lowest performance across experiments due to its regional sampling smoothing out fine details [S2]. In the Abdominal dataset, Mip-NeRF achieved a PSNR of 17.3 and SSIM of 0.709, which was lower than other models [S2].
*   **When to Use:** When anti-aliasing is critical and scene content is observed at different resolutions.
*   **When Not to Use:** When fine detail preservation in complex spatial relationships is paramount, such as in medical image reconstruction, where other NeRF variants may perform better [S2].

## Background
Mip-NeRF was introduced in the paper "Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields" [S1]. The name "mip-NeRF" stands for "multum in parvo NeRF," analogous to "mipmap" [S1]. It addresses the limitation that coordinate-based neural representations could previously only be anti-aliased using expensive supersampling [S1]. The method was developed to enable efficient rendering of anti-aliased images during both training and testing, and the use of multiscale images during training [S1].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S2] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
