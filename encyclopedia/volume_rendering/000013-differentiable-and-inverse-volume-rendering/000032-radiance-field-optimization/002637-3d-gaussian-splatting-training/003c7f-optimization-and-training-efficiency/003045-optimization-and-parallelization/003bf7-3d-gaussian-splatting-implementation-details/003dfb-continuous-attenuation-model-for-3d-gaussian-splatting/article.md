# Continuous Attenuation Model for 3D Gaussian Splatting

Also called: Attenuation-based 3DGS, Equivalent Ray-Marching Formulation

## Overview
The Continuous Attenuation Model reformulates the rasterization-based image formation of 3D Gaussian Splatting into a volumetric ray-marching framework by mapping Gaussian primitives to a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. This equivalence allows the application of theoretical results from Other-at-Volume (OaV) for surface extraction, ensuring that transmittance isosurfaces remain unbiased and ellipsoidal. It distinguishes itself from prior attenuation approximations like those in GGGS [57] by enforcing geometric constraints that prevent surface erosion and multi-view inconsistency.

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$, where each Gaussian $G_i$ is defined by its mean $\boldsymbol{\mu}_i$, covariance matrix $\boldsymbol{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), opacity $\alpha_i$, and view-dependent color $\mathbf{c}_i(\mathbf{w})$. A ray defined by origin $\mathbf{o}$ and direction $\mathbf{w}$.
*   **Output:** The continuous attenuation coefficient $\sigma(\mathbf{x}, \mathbf{w})$ and the transmittance function $T_{\mathbf{o},\mathbf{w}}(t)$, which are used to compute the rendered color and depth via volumetric integration.

**Data Structures**
The Gaussian primitive $G(\mathbf{x})$ is evaluated as a multivariate Gaussian density function. For a single Gaussian centered at $\boldsymbol{\mu}$ with covariance $\boldsymbol{\Sigma}$:
$$ G(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})\right) $$
The attenuation model relies on the gradient of the log-transmittance. No additional data structures are required beyond the standard Gaussian parameters.

**Numbered Steps**
1.  **Identify Maximum Contribution Point:** For a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{w}$, find the flight distance $t_{\mathbf{o},\mathbf{w}}^G$ where the Gaussian contribution is maximized:
    $$ t_{\mathbf{o},\mathbf{w}}^G := \arg \max_{t \geq 0} \{G(\mathbf{o} + t\mathbf{w})\} $$
    This point corresponds to the intersection of the ray with the Gaussian's principal axis closest to the camera, or the point where the ray passes closest to the Gaussian center projected onto the ray direction.

2.  **Define Continuous Transmittance:** Construct the transmittance function $T_{\mathbf{o},\mathbf{w}}(t)$ for a single Gaussian:
    $$ T_{\mathbf{o},\mathbf{w}}(t) = 1 - G(\mathbf{o} + \min(t, t_{\mathbf{o},\mathbf{w}}^G)\mathbf{w}) $$
    This formulation ensures that transmittance decreases from 1 (at $t=0$) to $1 - G(t_{\mathbf{o},\mathbf{w}}^G)$ (at $t \geq t_{\mathbf{o},\mathbf{w}}^G$), matching the alpha-blending behavior where the maximum opacity is applied at the point of maximum density.

3.  **Derive Attenuation Coefficient:** Compute the view-dependent attenuation coefficient $\sigma(\mathbf{x}, \mathbf{w})$ from the transmittance. The correct formulation is:
    $$ \sigma(\mathbf{x}, \mathbf{w}) = \max(0, -\mathbf{w} \cdot \nabla \log(1 - G(\mathbf{x}))) $$
    This ensures that the attenuation is non-negative and aligns with the gradient of the Gaussian density along the ray direction.

4.  **Render via Volumetric Integration:** The color $\mathbf{C}_{\mathbf{o},\mathbf{w}}$ is computed by integrating the density-based model:
    $$ \mathbf{C}_{\mathbf{o},\mathbf{w}} = \mathbf{c}(\mathbf{w}) \int_0^{+\infty} -\frac{d}{dt} T_{\mathbf{o},\mathbf{w}}(s) ds = \mathbf{c}(\mathbf{w}) G(\mathbf{o} + t_{\mathbf{o},\mathbf{w}}^G \mathbf{w}) $$
    This matches the standard alpha-blending result for a single Gaussian.

5.  **Composite Multiple Gaussians:** Extend to $N$ Gaussians under two assumptions:
    *   (a) Statistical independence of Gaussians [S1].
    *   (b) Non-overlapping Gaussian primitives [S1].
    Under these assumptions, the total transmittance is the product of individual transmittances, and the total color is the sum of individual contributions weighted by the accumulated transmittance.

**Formulas**
*   **Gaussian Density:** $G(\mathbf{x}) = \exp(-\frac{1}{2} D^2(\mathbf{x}))$, where $D^2(\mathbf{x}) = (\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})$.
*   **Attenuation:** $\sigma(\mathbf{x}, \mathbf{w}) = \max(0, -\mathbf{w} \cdot \nabla \log(1 - G(\mathbf{x})))$.
*   **Gradient of Log-Transmittance:** $\nabla \log(1 - G(\mathbf{x})) = \frac{-\nabla G(\mathbf{x})}{1 - G(\mathbf{x})}$.
*   **Gradient of Gaussian:** $\nabla G(\mathbf{x}) = -G(\mathbf{x}) \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})$.
*   **Combined Attenuation:** $\sigma(\mathbf{x}, \mathbf{w}) = \max\left(0, \mathbf{w} \cdot \frac{G(\mathbf{x}) \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})}{1 - G(\mathbf{x})}\right)$.

**Parameters**
*   **Assumptions:** The model assumes statistical independence and non-overlapping Gaussians. In practice, this approximation is effective despite minor discrepancies in poorly supervised background regions [S1].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls**
*   **Incorrect Attenuation Formula:** Using the simpler attenuation $\sigma(\mathbf{x}, \mathbf{w}) = \frac{1}{2} |\mathbf{w} \cdot \nabla \log(1 - G(\mathbf{x}))|$ (as in GGGS [57]) produces non-spherical isosurfaces for a single Gaussian. This leads to non-multi-view consistent depth maps and surface erosion [S1].
*   **Boundary Conditions:** The ray origin $\mathbf{o}$ is assumed to be in empty space ($G(\mathbf{o}) \approx 0$) [S1].
*   **Overlap:** The non-overlapping assumption breaks down when Gaussians significantly overlap, leading to inaccuracies in the volumetric formulation compared to the rasterization-based alpha blending [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_density(x, mu, Sigma_inv):
    """
    Computes the value of a Gaussian at point x.
    x: (3,) array
    mu: (3,) array
    Sigma_inv: (3,3) array
    """
    diff = x - mu
    exponent = -0.5 * np.dot(diff, np.dot(Sigma_inv, diff))
    return np.exp(exponent)

def compute_attenuation(x, w, mu, Sigma_inv):
    """
    Computes the continuous attenuation coefficient sigma(x, w).
    x: (3,) array, point in space
    w: (3,) array, ray direction (normalized)
    mu: (3,) array, Gaussian center
    Sigma_inv: (3,3) array, inverse covariance
    """
    G = gaussian_density(x, mu, Sigma_inv)
    
    # Avoid division by zero if G is close to 1
    if G >= 1.0:
        return 0.0
        
    # Gradient of G: dG/dx = -G * Sigma_inv * (x - mu)
    diff = x - mu
    grad_G = -G * np.dot(Sigma_inv, diff)
    
    # Gradient of log(1 - G): d/dx log(1 - G) = -grad_G / (1 - G)
    grad_log_transmittance = -grad_G / (1.0 - G)
    
    # Attenuation: max(0, -w . grad_log_transmittance)
    # Note: The formula is sigma = max(0, -w . nabla log(1-G))
    # Since nabla log(1-G) = -grad_G / (1-G), 
    # sigma = max(0, w . grad_G / (1-G))
    
    dot_product = np.dot(w, grad_G) / (1.0 - G)
    sigma = max(0.0, dot_product)
    
    return sigma

def find_max_contribution_point(o, w, mu, Sigma_inv):
    """
    Finds t such that G(o + t*w) is maximized for t >= 0.
    For a Gaussian, this occurs at the projection of mu onto the ray,
    clamped to t >= 0.
    """
    # Vector from ray origin to Gaussian center
    v = mu - o
    
    # Project v onto w
    t = np.dot(v, w)
    
    # Clamp to t >= 0
    t = max(0.0, t)
    
    return t

def render_single_gaussian_ray(o, w, mu, Sigma_inv, color):
    """
    Renders a single Gaussian using the continuous attenuation model.
    """
    # 1. Find the point of maximum contribution
    t_max = find_max_contribution_point(o, w, mu, Sigma_inv)
    
    # 2. Evaluate Gaussian at that point
    x_max = o + t_max * w
    G_max = gaussian_density(x_max, mu, Sigma_inv)
    
    # 3. The color is simply the Gaussian color weighted by the max density
    # C = c(w) * G(o + t_max * w)
    # Since o is in empty space, G(o) ~ 0, so the integral result is c(w) * G_max
    return color * G_max
```

## Subtopics

### First-order Spherical Harmonics Distillation
Uses the attenuation model to distill SH coefficients for efficient rendering. [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)

### RoofGS
Applies the continuous attenuation model to enforce roof-like structures in surface extraction. [RoofGS](9830-roofgs.md)

### Inverse Multiquadric Splatting
An alternative primitive representation that may benefit from similar volumetric formulations. [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)

### Tile-Based Comparison-Free Sorting
Optimizes the rasterization pipeline, which the attenuation model seeks to replace for surface extraction tasks. [Tile-Based Comparison-Free Sorting](9991-tile-based-comparison-free-sorting.md)

### Explicit Camera Jacobian Derivation
Provides gradients for the camera parameters, which can be derived from the attenuation model. [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)

### Opacity-Independent Gaussian Truncation
Defines truncation bounds for Gaussians, which are relevant for efficient ray-marching. [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)

## Operating characteristics
*   **Time Complexity:** The attenuation model requires evaluating the Gaussian density and its gradient for each point along the ray during marching. This is generally more expensive than the rasterization-based alpha blending, which only evaluates Gaussians intersecting the screen tiles. However, it enables more robust surface extraction.
*   **Memory Complexity:** No additional memory is required beyond the Gaussian parameters.
*   **Strengths:** Ensures unbiased, spherical isosurfaces aligned with the visible part of the Gaussian. Prevents surface erosion and multi-view inconsistency.
*   **Limitations:** Relies on the assumption of non-overlapping Gaussians, which may not hold in complex scenes. Computationally more intensive than rasterization.
*   **When to Use:** When high-fidelity surface extraction and multi-view consistency are required.
*   **When Not to Use:** When real-time novel view synthesis is the primary goal and rasterization is sufficient.

## Background
The Continuous Attenuation Model was introduced in "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1]. It builds on the observation that while rasterization is efficient, it lacks the theoretical grounding for surface extraction provided by volumetric rendering. The model corrects the geometric inaccuracies of previous attenuation approximations like GGGS [57], which produced non-spherical isosurfaces.

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
