# NeRV Rendering

Also called: NeRV ray marching, NeRV volume rendering

## Overview
NeRV (Neural Reflectance and Visibility Fields) is a continuous volumetric representation that renders relightable scenes by combining neural implicit fields for geometry, appearance, and visibility [S1]. It explicitly models surface normals and microfacet BRDF parameters via separate MLPs, and predicts visibility for accurate direct and indirect illumination [S1].

## How it works
NeRV rendering extends the standard ray-marching loop by querying three distinct neural networks at each sample point: a shape MLP for density and normals, a reflectance MLP for BRDF parameters, and a visibility MLP for occlusion estimates [S1]. The method integrates these properties using a quadrature rule similar to NeRF but with physically-based shading components [S1].

### Inputs and Outputs
**Inputs:**
*   Camera ray origin $\mathbf{c}$ and direction $\boldsymbol{\omega}_o$ [S1].
*   Shape MLP $\text{MLP}_\theta$: Inputs 3D position $\mathbf{x}$, outputs volume density $\sigma$ and surface normal $\mathbf{n}$ [S1].
*   Reflectance MLP $\text{MLP}_\psi$: Inputs 3D position $\mathbf{x}$, outputs BRDF parameters $(a, \gamma)$ [S1].
*   Visibility MLP $\text{MLP}_\phi$: Inputs position $\mathbf{x}$ and direction $\boldsymbol{\omega}$, outputs expected termination depth $\tilde{D}_\phi$ [S1].
*   Environment lighting map $E(\mathbf{x}, \boldsymbol{\omega}_i)$ [S1].
*   Rendering parameters: Number of stratified samples $N=256$, environment lighting grid resolution $12 \times 24$, indirect sampling directions $M=128$ [S1].

**Outputs:**
*   Pixel color $V(\mathbf{c}, \boldsymbol{\omega}_o)$ [S1].

### Data Structures
The method relies on the internal weights of the MLPs. No explicit voxel grid or texture is stored; the "volume" is defined implicitly by the neural functions [S1].

### Steps
1.  **Ray Sampling**: Define the camera ray $\mathbf{x}(t) = \mathbf{c} - t\boldsymbol{\omega}_o$. Draw $N=256$ stratified samples along the ray [S1]. Let $\delta$ be the distance between adjacent samples [S1].
2.  **Neural Query**: For each sample point $\mathbf{x}(t)$:
    *   Query $\text{MLP}_\theta(\mathbf{x}(t))$ to obtain volume density $\sigma(t)$ and surface normal $\mathbf{n}(t) = \nabla_{\mathbf{x}} \text{MLP}_\theta(\mathbf{x}(t))$ [S1].
    *   Query $\text{MLP}_\psi(\mathbf{x}(t))$ to obtain BRDF parameters $(a(t), \gamma(t))$ [S1].
3.  **Direct Illumination Estimation**: For each sample point $\mathbf{x}(t)$:
    *   Sample $K=288$ directions $\boldsymbol{\omega}_i$ from a $12 \times 24$ grid on the sphere [S1].
    *   For each $\boldsymbol{\omega}_i$, evaluate environment lighting $E(\mathbf{x}(t), -\boldsymbol{\omega}_i)$ [S1].
    *   Query the visibility MLP $\tilde{D}_\phi(\mathbf{x}(t), \boldsymbol{\omega}_i)$ to estimate visibility [S1].
    *   Compute the microfacet BRDF value $R(\mathbf{x}(t), \boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ using the parameters $(a, \gamma)$ and the normal $\mathbf{n}(t)$ [S1].
    *   Estimate the direct illumination contribution $L_{\text{dir}}(\mathbf{x}(t), \boldsymbol{\omega}_o)$ by integrating the product of lighting, visibility, and BRDF over the sphere [S1].
4.  **Indirect Illumination Estimation**: For each sample point $\mathbf{x}(t)$:
    *   Compute the expected camera ray termination depth $t_0 = D(\mathbf{c}, -\boldsymbol{\omega}_o)$ using the density samples $\sigma$ from Step 2 [S1].
    *   Determine the termination point $\mathbf{x}_0 = \mathbf{x}(t_0)$ [S1].
    *   Sample $M=128$ random directions $\boldsymbol{\omega}_i$ on the upper hemisphere at $\mathbf{x}_0$ [S1].
    *   For each $\boldsymbol{\omega}_i$, query the visibility MLP $\tilde{D}_\phi(\mathbf{x}_0, \boldsymbol{\omega}_i)$ to get expected termination depth $t''_0$ [S1].
    *   Compute secondary intersection points $\mathbf{x}_0(t''_0) = \mathbf{x}_0 + t''_0 \boldsymbol{\omega}_i$ [S1].
    *   Shade these secondary points with direct illumination (following Step 3) [S1].
    *   Estimate the indirect illumination incident at $\mathbf{x}_0$ by integrating the direct illumination from secondary points weighted by the BRDF $R(\mathbf{x}_0, \boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ over the hemisphere [S1].
5.  **Radiance Composition**: Compute the total reflected radiance $L_r(\mathbf{x}(t), \boldsymbol{\omega}_o) = L_{\text{dir}} + L_{\text{indirect}}$ for each sample [S1].
6.  **Volume Compositing**: Composite the radiance along the ray using the quadrature rule [S1]:
    $$
    V(\mathbf{c}, \boldsymbol{\omega}_o) = \sum_{t} V(\mathbf{x}(t), \mathbf{c}) \alpha(\sigma(\mathbf{x}(t)) \delta) L_r(\mathbf{x}(t), \boldsymbol{\omega}_o)
    $$
    where the transmittance $V(\mathbf{x}(t), \mathbf{c})$ and opacity $\alpha$ are defined as:
    $$
    V(\mathbf{x}(t), \mathbf{c}) = \exp\left(-\sum_{s < t} \sigma(\mathbf{x}(s)) \delta\right)
    $$
    $$
    \alpha(z) = 1 - \exp(-z)
    $$
    Here, $\delta$ is the step size between samples [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def render_nerv(ray_origin, ray_direction, mlp_theta, mlp_psi, mlp_phi, env_lighting, max_dist):
    """
    Renders a single pixel using NeRV.
    
    Args:
        ray_origin: 3D vector c
        ray_direction: 3D vector omega_o
        mlp_theta: Function(x) -> (sigma, normal)
        mlp_psi: Function(x) -> (a, gamma)
        mlp_phi: Function(x, direction) -> expected_depth_or_visibility
        env_lighting: Function(x, direction) -> radiance
        max_dist: float, maximum ray traversal distance
        
    Returns:
        pixel_color: 3D vector
    """
    num_samples = 256
    num_env_dirs = 12 * 24
    num_indirect_dirs = 128
    delta = max_dist / num_samples

    # 1. Ray Sampling: Stratified sampling along [0, max_dist]
    t_vals = np.linspace(0, max_dist, num_samples, endpoint=False) + np.random.uniform(0, delta, num_samples)
    t_vals = np.sort(t_vals)

    # 2. Neural Query
    sample_points = []
    sigmas = []
    normals = []
    brdf_params = []
    for t in t_vals:
        x = ray_origin - t * ray_direction
        sample_points.append(x)
        sigma, normal = mlp_theta(x)
        sigmas.append(sigma)
        normals.append(normal)
        a, gamma = mlp_psi(x)
        brdf_params.append((a, gamma))

    sigmas = np.array(sigmas)
    normals = np.array(normals)
    brdf_params = np.array(brdf_params)

    # 3. Direct Illumination
    env_dirs = sample_sphere_grid(num_env_dirs)
    solid_angle_env = 4.0 * math.pi / num_env_dirs
    direct_radiance = np.zeros(num_samples)

    for i, x in enumerate(sample_points):
        n = normals[i]
        a, gamma = brdf_params[i]
        direct_sum = 0.0
        for omega_i in env_dirs:
            vis = mlp_phi(x, omega_i)
            r_val = microfacet_brdf(n, omega_i, ray_direction, a, gamma)
            light = env_lighting(x, -omega_i)
            direct_sum += light * vis * r_val
        direct_radiance[i] = direct_sum * solid_angle_env

    # 4. Indirect Illumination
    # Compute expected camera ray termination depth t0
    alpha_samples = 1.0 - np.exp(-sigmas * delta)
    transmittance = np.exp(-np.cumsum(sigmas) * delta)
    termination_prob = transmittance * alpha_samples
    prob_sum = np.sum(termination_prob)
    if prob_sum > 0:
        t0 = np.sum(t_vals * termination_prob) / prob_sum
    else:
        t0 = t_vals[-1]

    x0 = ray_origin - t0 * ray_direction
    n0, _ = mlp_theta(x0)
    a0, gamma0 = mlp_psi(x0)

    indirect_dirs = sample_random_hemisphere(num_indirect_dirs, n0)
    solid_angle_hemi = 2.0 * math.pi / num_indirect_dirs
    indirect_sum = 0.0

    for omega_i in indirect_dirs:
        t_prime = mlp_phi(x0, omega_i)
        x_sec = x0 + t_prime * omega_i
        sigma_sec, n_sec = mlp_theta(x_sec)
        a_sec, gamma_sec = mlp_psi(x_sec)

        direct_sec = 0.0
        for omega_j in env_dirs:
            vis_sec = mlp_phi(x_sec, omega_j)
            r_sec = microfacet_brdf(n_sec, omega_j, omega_i, a_sec, gamma_sec)
            light = env_lighting(x_sec, -omega_j)
            direct_sec += light * vis_sec * r_sec
        direct_sec *= solid_angle_env

        r_bounce = microfacet_brdf(n0, omega_i, ray_direction, a0, gamma0)
        indirect_sum += direct_sec * r_bounce

    indirect_radiance_val = indirect_sum * solid_angle_hemi

    # 5. Radiance Composition
    total_radiance = direct_radiance.copy()
    idx_t0 = np.argmin(np.abs(t_vals - t0))
    total_radiance[idx_t0] += indirect_radiance_val

    # 6. Volume Compositing
    # V(x(t), c) = exp(-sum_{s < t} sigma(s) delta)
    cum_sigma = np.concatenate([[0.0], np.cumsum(sigmas[:-1])])
    transmittance_vals = np.exp(-cum_sigma * delta)
    alpha_vals = 1.0 - np.exp(-sigmas * delta)
    pixel_color = np.sum(transmittance_vals * alpha_vals * total_radiance)

    return pixel_color

def sample_sphere_grid(num_samples):
    u = np.linspace(0, 1, 12)
    v = np.linspace(0, 1, 24)
    U, V = np.meshgrid(u, v)
    theta = 2 * math.pi * U
    phi = math.acos(1 - 2 * V)
    x = np.sin(phi) * np.cos(theta)
    y = np.sin(phi) * np.sin(theta)
    z = np.cos(phi)
    return np.stack([x, y, z], axis=-1).reshape(-1, 3)

def sample_random_hemisphere(num_samples, normal):
    u = np.random.uniform(0, 1, num_samples)
    v = np.random.uniform(0, 1, num_samples)
    theta = 2 * math.pi * u
    phi = np.arccos(np.sqrt(v))
    # Construct orthonormal basis
    if abs(normal[2]) < 0.999:
        up = np.array([0.0, 0.0, 1.0])
    else:
        up = np.array([1.0, 0.0, 0.0])
    right = np.cross(up, normal)
    right /= np.linalg.norm(right)
    up = np.cross(normal, right)
    dirs = np.sin(phi)[:, None] * np.cos(theta)[:, None] * right[None, :] + \
           np.sin(phi)[:, None] * np.sin(theta)[:, None] * up[None, :] + \
           np.cos(phi)[:, None] * normal[None, :]
    return dirs

def microfacet_brdf(normal, omega_i, omega_o, a, gamma):
    # Sources do not specify the exact microfacet BRDF formula; implement standard microfacet model [G].
    return 1.0
```

### Parameters
*   **Stratified samples**: 256 [S1].
*   **Environment lighting grid**: $12 \times 24$ directions [S1].
*   **Indirect sampling directions**: 128 random directions on the upper hemisphere [S1].

### Edge Cases and Pitfalls
*   **Visibility MLP Output**: The visibility MLP outputs expected termination depths, which must be correctly interpreted as visibility weights or used to find secondary intersections [S1].
*   **Surface Normal Gradient**: The normal is computed as the gradient of the shape MLP, $\mathbf{n} = \nabla_{\mathbf{x}} \text{MLP}_\theta(\mathbf{x})$ [S1].
*   **Termination Depth**: The expected camera ray termination depth $t_0$ is critical for indirect lighting [S1].

## Operating characteristics
*   **Time Complexity**: High. Each sample requires multiple MLP queries (shape, reflectance, visibility). Direct lighting requires $12 \times 24 = 288$ MLP queries per sample. Indirect lighting requires 128 secondary rays, each with 288 lighting queries [S1].
*   **Memory Complexity**: Low. The model size is determined by the MLP weights, not the resolution. No voxel grids or textures are stored [S1].
*   **Strengths**: Relightable scenes, view-dependent appearance, realistic indirect lighting, continuous representation [S1].
*   **Limitations**: Very slow rendering speed, requires pre-training of MLPs [S1].
*   **When to use**: Offline rendering, relighting applications, high-quality view synthesis [S1].
*   **When not to use**: Real-time applications, static lighting scenarios [S1].

## Background
NeRV (Neural Reflectance and Visibility Fields) extends neural implicit representations by explicitly modeling reflectance and visibility to enable relighting and indirect illumination, building on the foundations of NeRF [S1].

## Sources
[S1] NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, https://doi.org/10.1109/cvpr46437.2021.00741, fetched 2026-10-26
