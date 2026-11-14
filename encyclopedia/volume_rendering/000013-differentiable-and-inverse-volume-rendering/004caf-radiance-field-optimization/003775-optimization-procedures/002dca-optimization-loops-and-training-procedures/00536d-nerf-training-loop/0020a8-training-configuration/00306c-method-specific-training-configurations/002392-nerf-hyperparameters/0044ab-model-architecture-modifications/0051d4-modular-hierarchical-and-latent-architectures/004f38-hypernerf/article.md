# HyperNeRF

Also called: Hyper-Space Template, Hyper-space Neural Radiance Fields, HyperNeRF Template

## Overview
HyperNeRF is a method for modeling non-rigidly deforming scenes with topological variations (e.g., openings, closings, or transient effects) by extending the canonical 3D input coordinate space of a Neural Radiance Field (NeRF) into a higher-dimensional ambient space [S2]. Unlike standard deformation-based NeRFs that rely on continuous deformation fields—which cannot easily model the discontinuities required for topological changes due to the band-limited nature of coordinate-based MLPs—HyperNeRF embeds the scene in $\mathbb{R}^{3+W}$ and represents each observation as a non-planar slice through this hyperdimensional volume [S2]. It sets itself apart from siblings like Strata-NeRF by explicitly addressing topological discontinuities via ambient dimensions rather than stratified sampling or hierarchical structures [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ images with known camera poses, viewing directions $\mathbf{d}$, and per-frame latent codes for deformation $\boldsymbol{\omega}_i$ and appearance $\boldsymbol{\psi}_i$.
*   **Outputs:** A volume rendering function that produces color $\mathbf{c}$ and density $\sigma$ for any ray, conditioned on the specific temporal/topological state of the scene.

**Data Structures:**
1.  **Deformation Field MLP ($W$):** Maps observation-space coordinates $\mathbf{x}$ and latent deformation code $\boldsymbol{\omega}_i$ to a rigid transformation $(\mathbf{r}, \mathbf{v}) \in \mathfrak{se}(3)$ [S2].
2.  **Slicing Surface MLP ($H$):** Maps spatial position $\mathbf{x}$ and latent code $\boldsymbol{\omega}_i$ to ambient coordinates $\mathbf{w}$ [S2].
3.  **Template NeRF MLP ($F$):** Maps canonical hyper-space coordinates $(\mathbf{x}', \mathbf{w})$, view direction $\mathbf{d}$, and appearance code $\boldsymbol{\psi}_i$ to color $\mathbf{c}$ and density $\sigma$ [S2].

**Numbered Steps:**

1.  **Positional Encoding:** Apply sinusoidal positional encoding $\gamma$ to spatial coordinates $\mathbf{x}$ and view directions $\mathbf{d}$. For the ambient dimension $\mathbf{w}$, use a lower bandwidth encoding (e.g., $m=1$) [S2].
    $$
    \gamma(\mathbf{x}) = [\mathbf{x}, \sin(2^0\pi\mathbf{x}), \cos(2^0\pi\mathbf{x}), \dots, \sin(2^{m-1}\pi\mathbf{x}), \cos(2^{m-1}\pi\mathbf{x})]
    $$
    Typical value for spatial $m$: 6. Typical value for ambient $m$: 1 [S2].

2.  **Deformation to Canonical Space:** For an observation $i$ with latent code $\boldsymbol{\omega}_i$, compute the deformation field $W(\mathbf{x}, \boldsymbol{\omega}_i)$ to obtain rotation $\mathbf{r}$ and translation $\mathbf{v}$. Transform the observation-space coordinate $\mathbf{x}$ to the canonical space coordinate $\mathbf{x}'$ using the rigid transformation defined by $(\mathbf{r}, \mathbf{v})$ [S2].

3.  **Ambient Coordinate Generation:** Compute the ambient coordinate $\mathbf{w}$ for the spatial position $\mathbf{x}$ using the slicing surface MLP:
    $$
    \mathbf{w} = H(\mathbf{x}, \boldsymbol{\omega}_i)
    $$
    Here, $\mathbf{w} \in \mathbb{R}^W$. The number of ambient dimensions $W$ is a hyperparameter. Typical value: $W=2$ [S2].

4.  **Hyper-Space Query:** Concatenate the canonical spatial coordinate $\mathbf{x}'$ and the ambient coordinate $\mathbf{w}$ to form the hyper-space input $\mathbf{z} = [\mathbf{x}', \mathbf{w}]$. Query the template NeRF $F$ with $\mathbf{z}$, the view direction $\mathbf{d}$, and the appearance code $\boldsymbol{\psi}_i$:
    $$
    (\mathbf{c}, \sigma) = F(\mathbf{x}', \mathbf{w}, \mathbf{d}, \boldsymbol{\psi}_i)
    $$
    [S2]

5.  **Volume Rendering:** Perform standard volume rendering along the ray using the sampled points, their transformed canonical coordinates, generated ambient coordinates, and the resulting $\sigma$ and $\mathbf{c}$ values [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
from jax import jit

def positional_encoding(x, m):
    """
    Standard sinusoidal positional encoding.
    x: (..., D)
    m: int, number of frequency bands
    Returns: (..., D * 2 * m)
    """
    freq_bands = 2.0 ** jnp.arange(m)
    # x shape: (..., D)
    # freq_bands shape: (m,)
    # output shape: (..., D, 2, m) -> (..., D * 2 * m)
    scaled_x = x[..., None, :] * freq_bands[None, None, :] * jnp.pi
    encoded = jnp.concatenate([jnp.sin(scaled_x), jnp.cos(scaled_x)], axis=-2)
    return encoded.reshape(x.shape[:-1] + (-1,))

def deform_field_mlp(x_obs, omega_def, params_W):
    """
    Deformation field W: (x, omega_i) -> (r, v) in se(3)
    x_obs: (..., 3) observation coordinates
    omega_def: (..., L) latent deformation code
    Returns: (..., 6) transformation parameters (rotation vector, translation vector)
    """
    # Implementation of MLP W using params_W
    # Concatenate x_obs and omega_def
    input_feat = jnp.concatenate([x_obs, omega_def], axis=-1)
    # Apply MLP layers defined in params_W
    # ... (MLP forward pass) ...
    # Output is (..., 6)
    return transformation_output

def slicing_surface_mlp(x_obs, omega_def, params_H):
    """
    Slicing surface H: (x, omega_i) -> w (ambient coords)
    x_obs: (..., 3) observation coordinates
    omega_def: (..., L) latent deformation code
    Returns: (..., W) ambient coordinates
    """
    input_feat = jnp.concatenate([x_obs, omega_def], axis=-1)
    # Apply MLP layers defined in params_H
    # ... (MLP forward pass) ...
    # Output is (..., W)
    return ambient_coords

def template_nerf_mlp(x_canon, w_ambient, d_view, psi_appearance, params_F):
    """
    Template NeRF F: (x, w, d, psi_i) -> (c, sigma)
    x_canon: (..., 3) canonical spatial coordinates
    w_ambient: (..., W) ambient coordinates
    d_view: (..., 3) view direction
    psi_appearance: (..., K) latent appearance code
    Returns: (..., 3) color, (..., 1) density
    """
    # Encode inputs
    # Note: w_ambient uses m=1, x_canon and d_view use m=6
    enc_x = positional_encoding(x_canon, m=6)
    enc_w = positional_encoding(w_ambient, m=1)
    enc_d = positional_encoding(d_view, m=6)
    
    input_feat = jnp.concatenate([enc_x, enc_w, enc_d, psi_appearance], axis=-1)
    
    # Apply MLP layers defined in params_F
    # ... (MLP forward pass) ...
    # Output is (..., 4) -> split into color (3) and density (1)
    return color, density

def hypernerf_forward(x_obs, d_view, omega_def, psi_appearance, params):
    """
    Full HyperNeRF forward pass for a batch of points.
    """
    # 1. Deform to canonical space
    transform_params = deform_field_mlp(x_obs, omega_def, params['W'])
    x_canon = apply_se3_transformation(x_obs, transform_params)
    
    # 2. Get ambient coordinates
    w_ambient = slicing_surface_mlp(x_obs, omega_def, params['H'])
    
    # 3. Query template NeRF
    color, density = template_nerf_mlp(x_canon, w_ambient, d_view, psi_appearance, params['F'])
    
    return color, density
```

**Parameters:**
*   $W$: Number of ambient dimensions. Typical value: 2 [S2].
*   $m$: Number of frequency bands in positional encoding. Spatial $m=6$, Ambient $m=1$ [S2].
*   $\boldsymbol{\omega}_i$: Per-observation latent deformation code [S2].
*   $\boldsymbol{\psi}_i$: Per-observation latent appearance code [S2].

**Edge Cases and Pitfalls:**
*   **Topological Discontinuities:** Standard deformation fields fail here because MLPs with positional encoding perform band-limited kernel interpolation, making it difficult to model the discontinuities required for topological changes (e.g., a mouth opening) [S2]. HyperNeRF avoids this by using ambient dimensions.
*   **Regularization:** The elastic regularization loss used in Nerfies (penalizing non-unit singular values of the deformation Jacobian) is ill-suited for scenes with topological variations as it violates the as-rigid-as-possible assumption [S2]. HyperNeRF instead uses an optimization strategy that encourages smooth behavior in the higher dimensions [S2].
*   **Inefficiency of Axis-Aligned Slicing:** Using axis-aligned planes to slice the hyper-space requires storing redundant copies of shapes. The deformable slicing surface $H$ addresses this by allowing non-planar slices, resulting in a more compact representation [S2].

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive. Full-HD (1920x1080) training with 256 samples per ray takes roughly 64 hours on 4 TPU v4s for 1M iterations [S2]. Half-resolution (960x540) takes roughly 8 hours for 250k iterations [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** HyperNeRF reduces average error rates by 4.1% for interpolation and 8.6% for novel-view synthesis compared to Nerfies, as measured by LPIPS [S2].
*   **Strengths:** Capable of synthesizing views in scenes with significant topological changes (e.g., human mouth opening, banana peeling) [S1, S2]. Handles appearance variations via latent codes [S2].
*   **Limitations:** Requires careful tuning of ambient dimensions; increasing $W$ beyond 2 did not improve performance in tested sequences [S2]. Relies on a coarse-to-fine optimization strategy to ensure smooth behavior in higher dimensions [S2].

## Background
HyperNeRF was introduced by Kevin Park et al. in June 2021 [S1]. It is inspired by level set methods pioneered in the late 1980s by Osher and Sethian, which model moving scenes as static objects in a higher-dimensional ambient space [S2]. The method builds on Nerfies [S1, S2] by extending the canonical space to a higher dimension and using a slicing MLP to describe how to return to the 3D representation using ambient space coordinates. It addresses the limitations of continuous deformation fields in modeling topological variations [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] HyperNeRF, https://doi.org/10.1145/3478513.3480487, fetched 2026-10-28
