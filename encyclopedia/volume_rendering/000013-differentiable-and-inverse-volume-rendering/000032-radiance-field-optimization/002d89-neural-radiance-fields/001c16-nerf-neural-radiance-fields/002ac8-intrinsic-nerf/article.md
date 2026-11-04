# Intrinsic NeRF

Also called: Intrinsic Image Decomposition in NeRF, Intrinsic NeRF Decomposition, Intrinsic NeRF Model

## Overview
Intrinsic NeRF is a variant of Neural Radiance Fields that solves the ill-posed inverse rendering problem by decoupling geometry, illumination, and material properties without requiring supervised labels for lighting or reflectance [S1]. It differs from standard NeRF by outputting separate shading and reflectance components instead of a single color value, enabling downstream tasks such as relighting and material editing [S1].

## How it works
**Inputs and Outputs:**
The method takes a set of calibrated camera poses and corresponding RGB images as input. It outputs a continuous volumetric representation defined by a function $F_I$ that maps a 3D location $\mathbf{X}$ and a view direction $\mathbf{D}$ to a tuple $(\sigma, \mathbf{R}, S)$ [S1].
- $\sigma$: Volume density (scalar).
- $\mathbf{R}$: Reflectance (vector in $\mathbb{R}^3$), representing material properties.
- $S$: Shading (scalar in $\mathbb{R}$), representing illumination effects.

**Data Structures:**
Two Multi-Layer Perceptrons (MLPs) are used to approximate the intrinsic components:
1.  **Reflectance MLP:** Takes only the 3D location $\mathbf{X}$ as input. It does not depend on the view direction or lighting conditions [S1].
2.  **Shading MLP:** Takes both the 3D location $\mathbf{X}$ and the view direction $\mathbf{D}$ as inputs. It captures view-dependent shading effects [S1].

**Algorithm Steps:**
1.  **Ray Sampling:** For each camera ray $\mathbf{r}$, sample $N$ points along the ray from near plane $t_n$ to far plane $t_f$.
2.  **Intrinsic Field Evaluation:** For each sample point $t_i$ with position $\mathbf{X}_i$ and view direction $\mathbf{D}$:
    - Compute density $\sigma_i$ using the standard NeRF density MLP (or the shared geometry component).
    - Compute reflectance $\mathbf{R}_i = \text{MLP}_R(\mathbf{X}_i)$.
    - Compute shading $S_i = \text{MLP}_S(\mathbf{X}_i, \mathbf{D})$.
3.  **Volume Rendering:** Compute the rendered color $\hat{\mathbf{C}}(\mathbf{r})$ by integrating the product of transmittance, density, and the dot product of shading and reflectance along the ray [S1]:
    $$
    \hat{\mathbf{C}}(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(t) (\mathbf{S}(t) \cdot \mathbf{R}(t)) dt
    $$
    where $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(s) ds\right)$ is the accumulated transmittance [S1].
4.  **Loss Computation:** Minimize the reconstruction loss between the rendered color and the ground truth RGB image $\mathbf{C}(\mathbf{r})$ using only RGB supervision [S1]:
    $$
    \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \|\hat{\mathbf{C}}(\mathbf{r}) - \mathbf{C}(\mathbf{r})\|_2^2
    $$
    where $\mathcal{R}$ is the set of camera rays in a batch [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intrinsic_nerf_forward(ray_origins, ray_directions, mlp_reflectance, mlp_shading, mlp_density, t_near, t_far, num_samples):
    """
    Computes the rendered color for a batch of rays using Intrinsic NeRF.
    
    Args:
        ray_origins: Tensor of shape (B, 3), ray origins.
        ray_directions: Tensor of shape (B, 3), normalized ray directions.
        mlp_reflectance: MLP taking (B, 3) -> (B, 3) for reflectance R.
        mlp_shading: MLP taking (B, 3) and (B, 3) -> (B, 1) for shading S.
        mlp_density: MLP taking (B, 3) -> (B, 1) for density sigma.
        t_near: Float, near plane distance.
        t_far: Float, far plane distance.
        num_samples: Int, number of samples per ray.
        
    Returns:
        rendered_colors: Tensor of shape (B, 3).
    """
    # 1. Sample points along the ray
    t_vals = torch.linspace(t_near, t_far, num_samples)
    # Add jitter for anti-aliasing (standard NeRF practice, implied by volume rendering context)
    t_vals = t_vals + torch.rand_like(t_vals) * (t_far - t_near) / num_samples
    
    # 2. Compute 3D positions
    # X = origin + direction * t
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    delta_inf = t_far - t_vals[:, -1:]
    deltas = torch.cat([deltas, delta_inf], -1)
    
    # Midpoints for sampling
    t_mid = t_vals + deltas / 2.0
    X = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_mid.unsqueeze(2)
    
    # 3. Evaluate MLPs
    # Density
    sigma = mlp_density(X) # Shape: (B, num_samples, 1)
    
    # Reflectance (depends only on X)
    R = mlp_reflectance(X) # Shape: (B, num_samples, 3)
    
    # Shading (depends on X and view direction D)
    # D is constant for all samples on the same ray
    D_expanded = ray_directions.unsqueeze(1).expand(-1, num_samples, -1)
    S = mlp_shading(X, D_expanded) # Shape: (B, num_samples, 1)
    
    # 4. Volume Rendering
    # Transmittance
    alpha = 1.0 - torch.exp(-torch.relu(sigma) * deltas) # Standard alpha computation
    T = torch.cumprod(torch.cat([torch.ones_like(alpha[:, :1]), 1.0 - alpha + 1e-10], dim=1), dim=1)[:, :-1]
    
    # Weighted sum of color contributions
    # Color contribution = transmittance * density * (Shading dot Reflectance)
    # Note: The formula uses S(t) * R(t). S is scalar, R is vector.
    # Dot product is element-wise multiplication here since S is scalar.
    color_contrib = T * alpha * (S * R) # Shape: (B, num_samples, 3)
    
    rendered_colors = torch.sum(color_contrib, dim=1)
    
    return rendered_colors
```

**Edge Cases and Pitfalls:**
-   **Ill-posedness:** Without additional constraints, the decomposition of shading and reflectance is ambiguous. The model relies on the structural priors of the MLPs and the geometry regularization (see Subtopics) to converge to a meaningful solution [S1].
-   **Normalization:** View directions $\mathbf{D}$ should be normalized before input to the shading MLP to ensure consistent lighting estimation [S1].

## Subtopics

### Geometry Decomposition and Normal Regularization
This variant extends the basic Intrinsic NeRF by explicitly computing and regularizing geometry to improve the smoothness of the extracted normal maps [S1]. It computes depth and normal maps via volume rendering integrals:
-   **Depth Map:** $\mathbf{D}(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(t) t dt$ [S1].
-   **Normal Map:** $\mathbf{N}(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(t) (-\nabla(\sigma)) dt$ [S1].

To prevent noisy normal maps, especially in complex scenes, it introduces a depth-dependent normal regularization loss $\mathcal{L}_d$:
$$
\mathcal{L}_d = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left(1 - \frac{\mathbf{N}(\mathbf{r}) \cdot \mathbf{N}(\mathbf{r}_{adj})}{|\mathbf{N}(\mathbf{r})| |\mathbf{N}(\mathbf{r}_{adj})|}\right) \frac{1}{\mathbf{D}(\mathbf{r})^2}
$$
where $\mathbf{N}(\mathbf{r}_{adj})$ is the normal of the neighborhood around pixel $\mathbf{r}$ [S1]. The term $\frac{1}{\mathbf{D}(\mathbf{r})^2}$ acts as a depth-dependent weight, guiding regularization because a pixel in the background corresponds to a larger physical area than one in the foreground [S1]. Without this smoothness regularization, normal maps exhibit significantly more noise [S1].

## Operating characteristics
-   **Complexity:** Time complexity is dominated by the evaluation of three MLPs (density, reflectance, shading) per sample point, similar to standard NeRF but with slightly higher memory usage due to the additional outputs.
-   **Memory:** Requires storing parameters for two additional MLPs (reflectance and shading) compared to standard NeRF.
-   **Strengths:** Enables inverse rendering tasks such as relighting and material editing without supervised labels for lighting or materials [S1].
-   **Limitations:** The decomposition is inherently ill-posed; results depend heavily on the inductive biases of the network architecture and regularization terms [S1].
-   **When to use:** When the goal is to extract physically meaningful material and lighting properties for relighting, rather than just novel view synthesis.

## Background
Intrinsic NeRF was introduced to address the challenge of inverse rendering in 3D scenes using neural radiance fields. It adapts the concept of intrinsic image decomposition, traditionally used in 2D computer vision, to the 3D volumetric setting. The method allows for the decoupling of geometry, illumination, and material properties, facilitating applications like scene relighting and object insertion [S1].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
