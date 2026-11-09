# Anisotropic Neural Representation Learning

Also called: Anisotropic NeRF, SH-guided anisotropic features

## Overview
Anisotropic Neural Representation Learning is a method that models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity in neural rendering, replacing the isotropic point-sampled features of standard NeRF [S1]. Unlike Mip-NeRF which samples cones to approximate pre-filtered representations, this method modifies the MLP to predict view-dependent SH coefficients directly, avoiding the need for shape-based sampling parameterization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera origin $\mathbf{o}$, viewing direction $\mathbf{d}$, and ray bounds $t_n, t_f$.
*   **Outputs:** Rendered pixel color $\hat{C}(\mathbf{r})$.

**Data Structures:**
The core data structure is the first Multi-Layer Perceptron (MLP) of the NeRF architecture. Instead of outputting scalar density $\sigma$ and latent feature vectors $\mathbf{f}$, this MLP outputs coefficients for Spherical Harmonic basis functions.
*   **SH Coefficients:** For a given spatial location $\mathbf{x}$, the network outputs a vector of coefficients $\mathbf{a}_{\sigma}(\mathbf{x})$ for density and $\mathbf{a}_{c}(\mathbf{x})$ for color/radiance. The dimension of these vectors depends on the chosen SH degree $L$. For degree $L$, the number of coefficients is $(L+1)^2$.

**Numbered Steps:**
1.  **Ray Sampling:** Cast a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ from the camera origin. Sample $N$ points $t_i$ within the near $t_n$ and far $t_f$ bounds, defining intervals $I_i = [t_i, t_{i+1}]$ [S1].
2.  **Spatial Query:** For each sample point $\mathbf{x}_i = \mathbf{r}(t_i)$, query the first MLP. The MLP takes $\mathbf{x}_i$ as input and outputs the SH coefficients for density $\mathbf{a}_{\sigma}(\mathbf{x}_i)$ and color $\mathbf{a}_{c}(\mathbf{x}_i)$ [S1].
3.  **Directional Evaluation:** Evaluate the SH basis functions at the viewing direction $\mathbf{d}$ to convert coefficients into view-dependent values.
    *   Density: $\sigma_i = \sum_{l=0}^{L} \sum_{m=-l}^{l} a_{\sigma, l, m}(\mathbf{x}_i) Y_{l, m}(\mathbf{d})$
    *   Color: $\mathbf{c}_i = \sum_{l=0}^{L} \sum_{m=-l}^{l} a_{c, l, m}(\mathbf{x}_i) Y_{l, m}(\mathbf{d})$
    Where $Y_{l, m}$ are the spherical harmonic basis functions [S1].
4.  **Volume Rendering:** Compute the accumulated color using the discrete Riemann sum approximation [S1]:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=0}^{N} \mathbf{c}_i T_i (1 - \exp(-\sigma_i (t_{i+1} - t_i))) $$
    Where $T_i = \exp(-\sum_{j=0}^{i-1} \sigma_j (t_{j+1} - t_j))$ is the accumulated transmittance [S1].
5.  **Regularization (Training):** During training, decompose the anisotropic elements from the predicted density and latent features. Apply a point-wise operation to penalize the energy of the anisotropic components in the geometry representation to prevent over-fitting and shape-radiance ambiguity [S1].

**Formulas:**
*   **Volume Rendering Integral:**
    $$ \hat{C}(\mathbf{r}) = \int_{0}^{\infty} T(t)\sigma(\mathbf{r}(t))\mathbf{c}(t)\, dt $$
    $$ T(t) = \exp \left(- \int_{0}^{t} \sigma(\mathbf{r}(s)) \, ds\right) $$ [S1]
*   **Discrete Approximation:**
    $$ \hat{C}(\mathbf{r}) = \sum_{i=0}^{N} \mathbf{c}_i T_i (1 - \exp(-\sigma_i (t_{i+1} - t_i))) $$ [S1]

**Parameters:**
*   $N$: The number of samples on the ray within near and far bounds ($t_n$ and $t_f$) [S1].
*   $L$: The degree of the spherical harmonics. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Shape-Radiance Ambiguity:** Training with only color reconstruction loss causes strong anisotropy and shape-radiance ambiguity [S1]. This is handled by decomposing anisotropic elements and applying point-wise regularization [S1].
*   **Directional Ambiguity:** Standard NeRF ignores direction information in point intervals, resulting in ambiguous features. The SH-guided approach explicitly models this view-dependence [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherical_harmonic_basis(d, degree):
    """
    Evaluates spherical harmonic basis functions for direction d.
    Returns a vector of basis values.
    Note: Implementation details of Y_l,m are assumed to be available.
    """
    # Placeholder for actual SH basis evaluation
    # Returns vector of size (degree + 1)**2
    pass

def query_anisotropic_mlp(x, sh_coefficients_density, sh_coefficients_color):
    """
    Evaluates the view-dependent density and color at location x and direction d.
    """
    # In the actual model, x is input to the MLP to get coefficients.
    # Here we assume coefficients are pre-computed or passed.
    # The core mechanism is:
    # sigma = dot(coefficients_density, SH_basis(d))
    # color = dot(coefficients_color, SH_basis(d))
    pass

def render_anisotropic_ray(o, d, t_n, t_f, n_samples, sh_degree, mlp_query_fn):
    """
    Renders a single ray using Anisotropic Neural Representation Learning.
    """
    # 1. Sample points along the ray
    t_vals = np.linspace(t_n, t_f, n_samples)
    points = o + t_vals[:, np.newaxis] * d
    
    # 2. Query MLP for SH coefficients at each point
    # mlp_query_fn returns (coeffs_sigma, coeffs_color) for each point
    coeffs_sigma_list = []
    coeffs_color_list = []
    
    for p in points:
        c_s, c_c = mlp_query_fn(p, sh_degree)
        coeffs_sigma_list.append(c_s)
        coeffs_color_list.append(c_c)
        
    coeffs_sigma = np.array(coeffs_sigma_list) # Shape: (N, (L+1)^2)
    coeffs_color = np.array(coeffs_color_list) # Shape: (N, (L+1)^2, 3)
    
    # 3. Evaluate SH basis for the viewing direction d
    sh_basis = spherical_harmonic_basis(d, sh_degree) # Shape: ((L+1)^2,)
    
    # 4. Compute view-dependent sigma and color
    # sigma_i = sum_j (coeffs_sigma[i, j] * sh_basis[j])
    sigmas = np.dot(coeffs_sigma, sh_basis) # Shape: (N,)
    
    # color_i = sum_j (coeffs_color[i, j, :] * sh_basis[j])
    colors = np.einsum('ijk, k -> ij', coeffs_color, sh_basis) # Shape: (N, 3)
    
    # 5. Volume Rendering (Riemann Sum)
    # Calculate interval lengths
    dt = np.diff(np.concatenate(([0], t_vals))) # Shape: (N,)
    # Note: The source defines intervals I_i = [t_i, t_{i+1}] and I_0 = [0, t_1]
    # Assuming t_vals includes t_1...t_N. 
    # Let's align with source notation: t_1...t_N.
    # Intervals: [0, t_1], [t_1, t_2]...
    # The source uses N samples. Let's assume t_vals are the sample points.
    # dt[i] corresponds to interval ending at t_i? 
    # Source: I_i = [t_i, t_{i+1}]. I_0 = [0, t_1].
    # Let's assume t_vals are t_1...t_N.
    # dt_0 = t_1 - 0
    # dt_i = t_{i+1} - t_i for i < N-1
    # dt_{N-1} = t_f - t_{N-1} ? Or is t_N the last sample?
    # Source says: t_N = t_1 ... t_N. I_N = [t_N, inf].
    # Usually, we integrate up to t_f.
    
    # Re-aligning with standard NeRF discretization for clarity:
    # t_vals = [t_1, ..., t_N]
    # dt = [t_1-0, t_2-t_1, ..., t_f-t_N]
    dt_full = np.zeros(n_samples + 1)
    dt_full[0] = t_vals[0] - 0
    dt_full[1:-1] = np.diff(t_vals)
    dt_full[-1] = t_f - t_vals[-1]
    
    # Transmittance
    # T_i = exp(- sum_{j=0}^{i-1} sigma_j * dt_j)
    # Note: Source uses sigma_i for interval I_i.
    # Let's map sigma[i] to interval starting at t_i (or 0 for i=0).
    
    # Accumulate optical depth
    optical_depth = np.cumsum(sigmas * dt_full[:-1]) # Shape: (N,)
    # T_i for sample i (contributing to interval i)
    # T_0 = 1
    # T_1 = exp(-sigma_0 * dt_0)
    # T_i = exp(-sum_{j=0}^{i-1} sigma_j * dt_j)
    
    transmittance = np.exp(-optical_depth)
    transmittance = np.concatenate(([1.0], transmittance)) # Shape: (N+1,)
    
    # Alpha (opacity) for each interval
    # alpha_i = 1 - exp(-sigma_i * dt_i)
    alpha = 1.0 - np.exp(-sigmas * dt_full[:-1]) # Shape: (N,)
    
    # Contribution
    # C = sum c_i * T_i * alpha_i
    # T_i in source formula is T(t_i) ? 
    # Source: C = sum c_i T_i (1 - exp(...))
    # T_i is transmittance BEFORE interval i.
    
    contributions = colors * transmittance[:-1][:, np.newaxis] * alpha[:, np.newaxis]
    
    return np.sum(contributions, axis=0)

def train_step_loss(sigma_coeffs, color_coeffs, gt_color, sh_degree):
    """
    Computes loss including anisotropy regularization.
    """
    # 1. Color Loss
    # Render GT and compare
    pred_color = render_anisotropic_ray(...) # Simplified
    color_loss = np.mean((pred_color - gt_color)**2)
    
    # 2. Anisotropy Regularization
    # Decompose anisotropic elements and penalize energy
    # The source states: "decompose the anisotropic elements from the predicted density 
    # and latent features and employ point-wise operation to penalize the anisotropy"
    # This typically involves the higher-order SH coefficients (l > 0).
    
    # Extract higher-order coefficients (l >= 1)
    # Assuming coefficients are ordered by l, m
    # Total coeffs = (L+1)^2. l=0 has 1 coeff. l>=1 have (L+1)^2 - 1 coeffs.
    n_total = (sh_degree + 1)**2
    n_isotropic = 1
    aniso_coeffs_sigma = sigma_coeffs[:, n_isotropic:]
    aniso_coeffs_color = color_coeffs[:, n_isotropic:, :]
    
    # Penalize energy (L2 norm)
    aniso_reg = np.mean(aniso_coeffs_sigma**2) + np.mean(aniso_coeffs_color**2)
    
    total_loss = color_loss + lambda_aniso * aniso_reg
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** Similar to standard NeRF per sample, as SH evaluation is a dot product. However, it avoids the overhead of cone sampling or complex anti-aliasing approximations [S1].
*   **Memory Complexity:** Increases by the factor of the number of SH coefficients per feature dimension. For degree $L$, memory usage for features scales by $(L+1)^2$ compared to a single scalar/vector feature [S1].
*   **Strengths:** Eliminates directional ambiguity and shape-radiance ambiguity without relying on external geometric priors or multi-view stereo constraints [S1]. It is flexible and can be plugged into various NeRF-based frameworks [S1].
*   **Limitations:** Requires careful regularization to prevent over-fitting to anisotropic noise, which can cause shape-radiance ambiguity if not handled [S1].
*   **When to use:** When high-quality novel view synthesis is required for scenes with significant view-dependent effects (e.g., glossy materials, complex lighting) and standard NeRF suffers from blurring or ambiguity [S1].

## Background
The method was proposed in "Improving Neural Volume Rendering via Learning View-Dependent Integral Approximation" [S1]. It addresses the limitation of standard NeRF where direction information in point intervals is ignored, resulting in ambiguous features [S1]. It is related to anti-aliasing NeRFs but introduces a plug-and-play anisotropic neural representation [S1].

## Sources
[S1] Improving Neural Volume Rendering via Learning View-Dependent Integral Approximation, https://doi.org/10.1109/tvcg.2025.3554692, fetched 2026-10-26
