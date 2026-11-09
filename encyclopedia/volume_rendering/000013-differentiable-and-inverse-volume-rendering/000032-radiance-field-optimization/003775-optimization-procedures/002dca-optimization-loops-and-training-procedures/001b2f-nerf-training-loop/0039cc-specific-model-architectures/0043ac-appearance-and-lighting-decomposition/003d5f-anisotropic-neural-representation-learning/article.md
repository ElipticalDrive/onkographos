# Anisotropic Neural Representation Learning

Also called: Anisotropic NeRF, SH-guided anisotropic features

## Overview
Anisotropic Neural Representation Learning is a method that models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity in neural rendering, replacing the isotropic point-sampled features of standard NeRF [S1]. It differs from anti-aliasing approaches like Mip-NeRF, which sample cones to approximate pre-filtered representations, by modifying the MLP to predict view-dependent SH coefficients directly, thereby avoiding the need for shape-based sampling parameterization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray origin $\mathbf{o}$, viewing direction $\mathbf{d}$, and near/far bounds $t_n, t_f$.
*   **Output:** Pixel color $\hat{C}(\mathbf{r})$.

**Data Structures:**
The core data structure is a Multilayer Perceptron (MLP) that replaces the initial geometry/feature network of a standard NeRF.
*   **MLP Architecture:** The first MLP takes spatial coordinates $\mathbf{x}$ as input. Instead of outputting scalar density $\sigma$ and latent feature vectors $\mathbf{f}$, it outputs coefficients for spherical harmonic basis functions.
*   **SH Coefficients:** Let $K$ be the number of SH basis functions used. The MLP outputs a vector of size $K$ for density and a vector of size $K \times D$ for latent features (where $D$ is the feature dimension).

**Numbered Steps:**
1.  **Ray Sampling:** Sample $N$ points along the ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ within the near and far bounds $[t_n, t_f]$. Let the sampled distances be $t_1, \dots, t_N$, defining intervals $I_i = [t_i, t_{i+1}]$ for $i=0 \dots N$, where $t_0=t_n$ and $t_{N+1}=t_f$ (or $\infty$) [S1].
2.  **View-Dependent Feature Prediction:** For each sample point $i$ at location $\mathbf{x}_i = \mathbf{r}(t_i)$ and viewing direction $\mathbf{d}$:
    a. Pass $\mathbf{x}_i$ through the anisotropic MLP to obtain SH coefficients for density $\mathbf{a}_i^\sigma$ and features $\mathbf{a}_i^c$.
    b. Evaluate the SH basis functions at direction $\mathbf{d}$ to obtain view-dependent values.
    c. Compute the view-dependent density $\sigma_i$ and latent feature vector $\mathbf{c}_i$ (color/appearance) by dotting the coefficients with the basis evaluations.
3.  **Volume Rendering:** Compute the accumulated color using the discrete Riemann sum approximation [S1]:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=0}^{N} \mathbf{c}_i T_i (1 - \exp(-\sigma_i \Delta t_i)) $$
    where $\Delta t_i = t_{i+1} - t_i$, and $T_i = \exp\left(-\sum_{j=0}^{i-1} \sigma_j \Delta t_j\right)$ is the accumulated transmittance [S1].
4.  **Regularization (Training):** To prevent over-fitting to anisotropic artifacts (shape-radiance ambiguity), decompose the anisotropic elements from the predicted density and latent features. Apply a point-wise operation to penalize the energy of these anisotropic components in the geometry representation [S1].

**Formulas:**
*   **Volume Rendering Integral:**
    $$ \hat{C}(\mathbf{r})=\int _{0}^{\infty } T(t)\sigma (\mathbf {r}(t))\mathbf {c}(t)\, dt $$
    where $T (t) = \exp \left(- \int _{0}^{t} \sigma (\mathbf {r}(s)) \, ds\right)$ [S1].
*   **Discrete Approximation:**
    $$ \hat{C}(\mathbf{r}) =\sum _{i=0}^{N}\mathbf {c}_{i}T_i(1-\exp (-\sigma _{i}(t_{i+1}-t_{i}))) $$
    where $\sigma_i$ and $\mathbf{c}_i$ are view-dependent [S1].

**Parameters:**
*   $N$: The number of samples on the ray within near and far bounds ($t_n$ and $t_f$) [S1].
*   SH Degree: Not given in the sources.

**Pitfalls:**
*   **Shape-Radiance Ambiguity:** Training with only color reconstruction loss causes strong anisotropy and shape-radiance ambiguity [S1]. This is mitigated by the specific regularization term described in Step 4.
*   **Directional Ambiguity:** Standard isotropic features ignore direction information in point intervals, leading to ambiguous features and limited reconstruction quality [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def evaluate_sh(direction, coefficients):
    """
    Evaluates spherical harmonics at a given direction using provided coefficients.
    direction: (3,) array-like, viewing direction d.
    coefficients: (K,) array-like, SH coefficients for a single property (e.g., density).
    Returns: Scalar value.
    """
    # Implementation of SH basis evaluation required.
    # Standard SH basis functions Y_l^m(theta, phi) derived from direction.
    # value = sum(coefficients[k] * Y_k(direction))
    # [G] Placeholder for actual SH basis evaluation logic.
    pass

def anisotropic_nerf_forward(ray_origin, ray_direction, near, far, mlp, n_samples):
    """
    Forward pass for Anisotropic Neural Representation Learning.
    """
    # 1. Sample points along the ray
    t_vals = np.linspace(near, far, n_samples)
    points = ray_origin + t_vals[:, np.newaxis] * ray_direction # (N, 3)
    
    # 2. Predict SH coefficients via MLP
    # mlp takes (N, 3) and outputs SH coefficients
    # Assume mlp outputs: density_coeffs (N, K) and color_coeffs (N, K, D)
    density_coeffs, color_coeffs = mlp(points) 
    
    # 3. Evaluate view-dependent density and color
    # Evaluate SH basis at ray_direction for all samples
    # basis_evals: (N, K)
    basis_evals = evaluate_sh_basis(ray_direction, n_samples) 
    
    # Compute view-dependent sigma and c
    sigma = np.dot(basis_evals, density_coeffs.T) # (N,)
    # For color, dot product of basis and color coefficients
    # color_coeffs shape: (N, K, D). basis_evals: (N, K).
    # Result: (N, D)
    c = np.einsum('nk, nkj -> nj', basis_evals, color_coeffs)
    
    # 4. Volume Rendering
    # Compute interval lengths
    dt = np.diff(np.concatenate(([near], t_vals, [far]))) # (N+1,)
    
    # Compute transmittance T_i
    # T_i = exp(-sum_{j=0}^{i-1} sigma_j * dt_j)
    # Accumulate optical depth
    optical_depth = np.cumsum(sigma * dt[:-1]) # (N,)
    T = np.exp(-optical_depth)
    T = np.concatenate(([1.0], T)) # T_0 = 1
    
    # Compute contribution
    # Contribution_i = c_i * T_i * (1 - exp(-sigma_i * dt_i))
    # Note: dt needs to align with sigma. sigma is defined at points t_i.
    # Interval i corresponds to [t_i, t_{i+1}].
    # The formula in source uses sum i=0 to N.
    
    # Re-align for standard NeRF discrete sum:
    # sigma_i is density at t_i. Interval i is [t_i, t_{i+1}].
    # We have N samples. Let's assume standard NeRF indexing where we integrate over N intervals.
    # The source defines N samples t_1...t_N.
    # Let's stick to the source's notation: N samples, intervals I_i.
    
    # Simplified standard NeRF accumulation:
    weights = (1 - np.exp(-sigma * dt)) * T
    color = np.sum(weights[:, np.newaxis] * c, axis=0)
    
    return color

def anisotropic_loss(pred_color, gt_color, density_coeffs, color_coeffs):
    """
    Loss function including anisotropy regularization.
    """
    # 1. Color Reconstruction Loss
    color_loss = np.mean((pred_color - gt_color) ** 2)
    
    # 2. Anisotropy Regularization
    # Decompose anisotropic elements and penalize energy.
    # [G] Specific decomposition formula not provided in sources.
    # Typically involves measuring variance or magnitude of higher-order SH coefficients.
    # We assume a penalty on the magnitude of the SH coefficients beyond the 0th order (isotropic).
    # isotropic_energy = density_coeffs[:, 0]**2
    # anisotropic_energy = np.sum(density_coeffs[:, 1:]**2, axis=1)
    # reg_loss = np.mean(anisotropic_energy)
    
    # [G] Placeholder for specific regularization term described in [S1].
    reg_loss = 0.0 
    
    return color_loss + reg_loss
```

## Operating characteristics
*   **Time Complexity:** Similar to standard NeRF per forward pass, but with additional overhead for SH basis evaluation and MLP output size (K coefficients vs 1 scalar).
*   **Memory Complexity:** Higher than standard NeRF because the MLP must output $K$ coefficients per property instead of 1.
*   **Strengths:** Eliminates directional ambiguity; flexible and plug-and-play into various NeRF frameworks; improves rendering quality on synthetic and real-world scenes [S1].
*   **Limitations:** Requires careful regularization to prevent over-fitting to anisotropic artifacts; computational cost increases with the number of SH coefficients $K$ [S1].
*   **When to use:** When high-quality reconstruction is needed and directional ambiguity is a concern, especially in scenes with complex lighting or reflective surfaces.
*   **When not to use:** When computational resources are extremely limited and isotropic approximation is sufficient.

## Background
Originating from the paper "Improving Neural Volume Rendering via Learning View-Dependent Integral Approximation" (2025), this method addresses the limitations of standard NeRF where direction information is ignored in point intervals [S1]. It builds on the concept of anti-aliasing NeRFs but diverges by using learnable view-dependent features parameterized by MLPs and SH bases, rather than shape-based sampling parameterization [S1]. It is designed to be a sub-model that can replace the first MLP in various NeRFs [S1].

## Sources
[S1] Improving Neural Volume Rendering via Learning View-Dependent Integral Approximation, https://doi.org/10.1109/tvcg.2025.3554692, fetched 2026-10-26
