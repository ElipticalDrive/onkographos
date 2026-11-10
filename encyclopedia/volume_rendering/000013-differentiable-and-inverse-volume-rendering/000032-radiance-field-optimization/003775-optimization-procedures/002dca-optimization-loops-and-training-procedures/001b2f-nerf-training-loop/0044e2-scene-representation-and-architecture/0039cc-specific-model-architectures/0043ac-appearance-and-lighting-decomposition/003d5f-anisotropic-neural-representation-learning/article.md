# Anisotropic Neural Representation Learning

Also called: Anisotropic NeRF, SH-guided anisotropic features

## Overview
Anisotropic Neural Representation Learning is a method that models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity in neural rendering, replacing the isotropic point-sampled features of standard NeRF [S1]. Unlike Mip-NeRF which samples cones to approximate pre-filtered representations, this method modifies the MLP to predict view-dependent SH coefficients directly, avoiding the need for shape-based sampling parameterization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$.
*   **Output:** Density $\sigma$ and color $\mathbf{c}$.

**Data Structures:**
The core data structure is the set of Spherical Harmonic coefficients predicted by the density network.
*   **SH Coefficients:** A vector of coefficients $c_{l,m}$ for degree $L$ and order $m$. The specific degree is not given in the sources [S1].
*   **MLP Architecture:** The first Multi-Layer Perceptron (MLP), typically responsible for density and latent features in standard NeRF, is modified. Instead of outputting scalar density and feature vectors directly, it outputs the SH coefficients [S1].

**Numbered Steps:**
1.  **Positional Encoding:** Encode the 3D position $\mathbf{x}$ using positional encoding $\gamma(\mathbf{x})$ [S1, S3]. The encoding includes the original value and sinusoidal functions at increasing frequencies:
    $$
    \gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
    $$
    where $L$ is the number of frequency bands [S1, S3].
2.  **SH Coefficient Prediction:** The density network $F_\sigma$ takes $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1, S3]. The output is the vector of SH coefficients for density and latent features [S1].
3.  **View-Dependent Evaluation:** The predicted SH coefficients are evaluated at the query viewing direction $\mathbf{d}$ to capture anisotropy [S1]. This produces view-dependent density $\sigma$ and latent features [S1].
4.  **Color Prediction:** The color network $F_c$ takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the view-dependent latent features (derived from the SH evaluation) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1, S3].
5.  **Rendering Integration:** During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering [S1, S3]:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\delta_i$ is the distance between samples [S1, S3].

**Formulas:**
The volume rendering integral is approximated as a Riemann sum [S1]:
$$
\hat {C}(\mathbf {r})=\int _{0}^{\infty } T(t)\sigma (\mathbf {r}(t))\mathbf {c}(t)\, dt
$$
where $T (t) = \exp \left(- \int _{0}^{t} \sigma (\mathbf {r}(s)) \, ds\right)$ represents occlusions [S1].

Within the near and far bounds, $t_n$ and $t_f$ of the cast ray, a subset of points on the ray is sampled in a near-to-far order. Let $t_N = t_1, \dots, t_N$ be $N$ samples on the ray that define the intervals, e.g., $I_i = [t_i, t_{i+1}]$ is the $i$-th interval, and $I_0 = [0, t_1], I_N = [t_N, \infty]$ [S1]. The volume density for particles along the interval $I_i$ is predicted under the assumption that opacity is constant along each interval, which indicates $\sigma(\mathbf{r}(t)) = \sigma(\mathbf{r}(t_i)), \forall t \in [t_i, t_{i+1}]$ for particles of constant radius and material. We denote $\sigma(\mathbf{r}(t_i)) = \sigma_i$ for notation convenience [S1]. Under this assumption, the rendered color can be written as an approximation of the $N$ sampled points [S1]:
$$
\hat {C}(\mathbf {r}) =\sum _{i=0}^{N}\mathbf {c}_{i}T_i(1-\exp (-\sigma _{i}(t_{i+1}-t_{i})))
$$

**Parameters:**
*   $N$: The number of samples on the ray within near and far bounds ($t_n$ and $t_f$), defining intervals $I_i = [t_i, t_{i+1}]$ [S1].

**Edge Cases and Pitfalls:**
*   Training with only color reconstruction loss causes strong anisotropy and shape-radiance ambiguity [S1].
*   This is handled by decomposing anisotropic elements and applying point-wise regularization to prevent over-fitting [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherical_harmonics_eval(direction, coefficients, degree):
    """
    Evaluates spherical harmonics at a given direction using coefficients.
    direction: numpy array of shape (3,) representing viewing direction d.
    coefficients: numpy array of SH coefficients.
    degree: integer, the degree of the spherical harmonics.
    Returns: numpy array of values.
    """
    # Implementation of SH basis evaluation at direction d
    # Not given in the sources. Standard SH basis functions Y_l^m are used.
    # The coefficients vector contains c_l_m for l=0..degree, m=-l..l.
    # This function computes sum(c_l_m * Y_l^m(d)) for each output channel.
    pass

def density_network_forward(x_encoded, sh_coefficients_mlp):
    """
    Predicts SH coefficients for density and latent features.
    x_encoded: numpy array, positional encoding of position x.
    sh_coefficients_mlp: MLP object that takes x_encoded and outputs SH coefficients.
    Returns: numpy array of SH coefficients.
    """
    # The first MLP in NeRF is modified to predict SH coefficients instead of density/features.
    # Not given in the sources. Standard MLP forward pass.
    pass

def anisotropic_nerf_forward(x, d, sh_coefficients_mlp, color_mlp, degree):
    """
    Forward pass for Anisotropic Neural Representation Learning.
    x: numpy array of shape (3,), 3D position.
    d: numpy array of shape (3,), viewing direction.
    sh_coefficients_mlp: MLP for predicting SH coefficients.
    color_mlp: MLP for predicting color.
    degree: integer, degree of spherical harmonics.
    Returns: density sigma, color c.
    """
    # Step 1: Positional Encoding
    x_encoded = positional_encoding(x)
    
    # Step 2: SH Coefficient Prediction
    sh_coeffs = sh_coefficients_mlp(x_encoded)
    
    # Step 3: View-Dependent Evaluation
    # Evaluate SH coefficients at direction d to get view-dependent density and latent features
    view_dependent_features = spherical_harmonics_eval(d, sh_coeffs, degree)
    
    # Extract density and latent features from view_dependent_features
    # Not given in the sources. Typically, the first component or specific channels map to density.
    sigma = view_dependent_features[0] # Placeholder for exact mapping
    latent_features = view_dependent_features[1:]
    
    # Step 4: Color Prediction
    # Color network takes encoded position, raw direction, and latent features
    c = color_mlp(x_encoded, d, latent_features)
    
    return sigma, c

def render_ray(ray_origin, ray_direction, near, far, num_samples, sh_coefficients_mlp, color_mlp, degree):
    """
    Renders a single ray using volumetric rendering.
    ray_origin: numpy array of shape (3,).
    ray_direction: numpy array of shape (3,).
    near: float, near bound t_n.
    far: float, far bound t_f.
    num_samples: integer, N.
    sh_coefficients_mlp: MLP for SH coefficients.
    color_mlp: MLP for color.
    degree: integer, degree of spherical harmonics.
    Returns: rendered color C(r).
    """
    # Sample points along the ray
    t_vals = np.linspace(near, far, num_samples)
    points = ray_origin + t_vals[:, np.newaxis] * ray_direction
    
    sigma_sum = 0.0
    color_accum = np.zeros(3)
    T = 1.0
    
    for i in range(num_samples):
        # Get density and color at point i
        sigma, c = anisotropic_nerf_forward(points[i], ray_direction, sh_coefficients_mlp, color_mlp, degree)
        
        # Compute opacity alpha
        delta = t_vals[i+1] - t_vals[i] if i < num_samples - 1 else far - t_vals[i] # Last interval
        alpha = 1.0 - np.exp(-sigma * delta)
        
        # Accumulate color
        color_accum += T * alpha * c
        T *= (1.0 - alpha)
        
        # Early termination if T is very small
        if T < 1e-4:
            break
            
    return color_accum
```

## Operating characteristics
*   **Time and Memory Complexity:** The method is described as "compact and generalizable" and "flexible and can be plugged into NeRF-based frameworks" [S1]. It improves "memory and computational efficiency" compared to methods relying on geometric priors and structural output like Patch-based Consistency or warp-based losses [S1].
*   **Measured Performance:** Extensive experiments on synthetic and real-world benchmark datasets show that the proposed representation can boost the rendering quality of various NeRFs [S1].
*   **Strengths and Limitations:** It eliminates directional ambiguity and prevents over-fitting of anisotropy through point-wise regularization [S1]. It avoids the need for shape-based sampling parameterization required by methods like Mip-NeRF [S1].
*   **When to use it:** When high-quality novel view synthesis is required and shape-radiance ambiguity is a concern, particularly in scenes with strong anisotropic effects.
*   **When not:** When computational resources are extremely limited and the overhead of SH evaluation is prohibitive, though the source claims efficiency gains over other regularization methods.

## Background
*   **Origin:** The method was proposed in "Improving Neural Volume Rendering via Learning View-Dependent Integral Approximation" [S1].
*   **History and Development:** It addresses the limitation of standard NeRF which ignores direction information in point intervals, resulting in ambiguous features and limited reconstruction quality [S1]. It builds on the idea of using spherical harmonics to model view-dependent functions, modifying the first MLP in NeRF to predict geometry explicitly in terms of spherical harmonics [S1].
*   **Where it is used:** It can be used as a sub-model to replace the first MLP in various NeRFs, resulting in anisotropic geometric features and better rendering quality [S1].

## Sources
[S1] Improving Neural Volume Rendering via Learning View-Dependent Integral Approximation, https://doi.org/10.1109/tvcg.2025.3554692, fetched 2026-10-26
