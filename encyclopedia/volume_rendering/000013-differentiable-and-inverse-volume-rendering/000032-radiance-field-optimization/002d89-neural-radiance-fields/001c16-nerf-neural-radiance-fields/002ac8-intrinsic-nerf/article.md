# Intrinsic NeRF

Also called: Intrinsic Image Decomposition in NeRF, Intrinsic NeRF Decomposition, Intrinsic NeRF Model

## Overview
Intrinsic NeRF is a variant of [NeRF (Neural Radiance Fields)](1185-nerf-neural-radiance-fields.md) that solves the ill-posed inverse rendering problem by decoupling geometry, illumination, and material properties without requiring supervised labels for lighting or reflectance [S1]. It differs from standard NeRF by outputting separate shading and reflectance components instead of a single color value, enabling downstream tasks such as relighting and material editing [S1].

## How it works
**Inputs and Outputs:**
The method takes a set of calibrated camera poses and corresponding RGB images as input [G]. Supervision uses only the RGB images without additional lighting or material labels [S1]. It outputs a continuous volumetric representation defined by a function $F_I$ that maps a 3D location $\mathbf{X}$ and a view direction $\mathbf{D}$ to a tuple $(\sigma, \mathbf{R}, S)$ [S1].
- $\sigma$: Volume density (scalar).
- $\mathbf{R}$: Reflectance (vector in $\mathbb{R}^3$), representing material properties.
- $S$: Shading (scalar in $\mathbb{R}$), representing illumination effects.

**Data Structures:**
Two Multi-Layer Perceptrons (MLPs) are used to approximate the intrinsic components:
1.  **Reflectance MLP:** Takes only the 3D location $\mathbf{X}$ as input. It does not depend on the view direction or lighting conditions [S1].
2.  **Shading MLP:** Takes both the 3D location $\mathbf{X}$ and the view direction $\mathbf{D}$ as inputs. It captures view-dependent shading effects [S1].

**Algorithm Steps:**
1.  **Ray Sampling:** For each camera ray $\mathbf{r}$, sample $N$ points along the ray from near plane $t_n$ to far plane $t_f$ [G].
2.  **Intrinsic Field Evaluation:** For each sample point $t_i$ with position $\mathbf{X}_i$ and view direction $\mathbf{D}$:
    - Compute density $\sigma_i$ using the standard NeRF density MLP (or the shared geometry component) [G].
    - Compute reflectance $\mathbf{R}_i = \text{MLP}_R(\mathbf{X}_i)$ [S1].
    - Compute shading $S_i = \text{MLP}_S(\mathbf{X}_i, \mathbf{D})$ [S1].
3.  **Volume Rendering:** Compute the rendered color $\hat{\mathbf{C}}(\mathbf{r})$ by integrating the product of transmittance, density, and the dot product of shading and reflectance along the ray [S1]:
    $$
    \hat{\mathbf{C}}(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(t) (\mathbf{S}(t) \cdot \mathbf{R}(t)) dt
    $$
    where $T(t)$ is the accumulated transmittance along the ray [S1].
4.  **Loss Computation:** Minimize the reconstruction loss between the rendered color and the ground truth RGB image $\mathbf{C}(\mathbf{r})$ using only RGB supervision [S1]:
    $$
    \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \|\hat{\mathbf{C}}(\mathbf{r}) - \mathbf{C}(\mathbf{r})\|_2^2
    $$
    where $\mathcal{R}$ is the set of camera rays in a batch [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intrinsic_nerf_forward(ray_origins, ray_directions, mlp_reflectance, mlp_shading, mlp_density, t_near, t_far, num_samples):
    # 1. Stratified sampling along the ray [G]
    t_vals = linspace(t_near, t_far, num_samples)
    t_vals = t_vals + random.uniform(0, 1, size=t_vals.shape) * (t_far - t_near) / num_samples

    # 2. Compute deltas and midpoints
    deltas = t_vals[1:] - t_vals[:-1]
    delta_inf = t_far - t_vals[-1]
    deltas = concatenate([deltas, delta_inf])
    t_mid = t_vals + deltas / 2.0
    X = ray_origins + ray_directions * t_mid[:, newaxis, newaxis]

    # 3. Evaluate MLPs
    sigma = mlp_density(X)  # Shape: (B, num_samples, 1)
    R = mlp_reflectance(X)  # Shape: (B, num_samples, 3)
    D_expanded = ray_directions[:, newaxis, :].expand(-1, num_samples, -1)
    S = mlp_shading(X, D_expanded)  # Shape: (B, num_samples, 1)

    # 4. Compute alpha and transmittance
    alpha = 1.0 - exp(-relu(sigma) * deltas[:, :, newaxis])
    T = ones_like(alpha[:, :1])
    for i in range(num_samples):
        T = concatenate([T, T[:, -1:] * (1.0 - alpha[:, i:i+1, :])], dim=1)
    T = T[:, 1:, :]

    # 5. Volume rendering
    color_contrib = T * alpha * (S * R)
    rendered_colors = sum(color_contrib, dim=1)

    return rendered_colors
```

**Edge Cases and Pitfalls:**
-   **Ill-posedness:** Without additional constraints, the decomposition of shading and reflectance is ambiguous [S1].
-   **Noise:** Without smoothness regularization, normal maps exhibit significantly more noise, particularly in complex scenes [S1].

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
-   **Complexity:** Time complexity is dominated by the evaluation of three MLPs (density, reflectance, shading) per sample point, similar to standard NeRF but with slightly higher memory usage due to the additional outputs [G].
-   **Memory:** Requires storing parameters for two additional MLPs (reflectance and shading) compared to standard NeRF [G].
-   **Strengths:** Enables inverse rendering tasks such as relighting and material editing without supervised labels for lighting or materials [S1].
-   **Limitations:** The decomposition is inherently ill-posed [S1].
-   **When to use:** When the goal is to extract physically meaningful material and lighting properties for relighting, rather than just novel view synthesis [G].

## Background
Intrinsic NeRF was introduced to address the challenge of inverse rendering in 3D scenes using neural radiance fields [S1]. It adapts the concept of intrinsic image decomposition, traditionally used in 2D computer vision, to the 3D volumetric setting [S1]. The method allows for the decoupling of geometry, illumination, and material properties, facilitating applications like scene relighting and object insertion [S1].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
