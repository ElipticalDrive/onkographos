# NeRV (Neural Reflectance and Visibility Fields)

Also called: Neural Reflectance and Visibility Fields

## Overview
NeRV is a differentiable inverse rendering method that reconstructs a 3D scene representation from images under unconstrained, known lighting, enabling view synthesis and relighting with indirect illumination [S1]. It extends the standard emission-absorption volume rendering model by representing the scene as a field of oriented particles that absorb and reflect light, parameterized by neural networks [S1]. Unlike its sibling [Neural Radiance Fields](44-radiance-field-optimization.md) which only models emission, or [Neural Reflectance Fields](6755-neural-radiosity.md) which is limited to single point lights, NeRV supports arbitrary environment lighting and one-bounce global illumination by introducing a **Neural Visibility Field** [S1]. This field approximates visibility and surface intersection depths via an MLP, reducing the computational complexity of light transport simulation from quadratic/cubic to linear/double-spherical, making optimization tractable for complex scenes [S1].

## How it works
NeRV optimizes two Multi-Layer Perceptrons (MLPs) alongside the scene geometry to simulate light transport. The method decomposes incident illumination into direct lighting (from an environment map) and indirect lighting (reflected from other surfaces) [S1].

### Inputs and Outputs
*   **Inputs:** A set of images of a scene with known, unconstrained lighting (environment map $E$). Camera poses and intrinsics are assumed known or optimized jointly (standard NeRF setup).
*   **Outputs:** A 3D representation capable of rendering novel views under arbitrary lighting, including indirect illumination. The representation consists of:
    *   Volume density $\sigma(\mathbf{x})$
    *   Surface normal $\mathbf{n}(\mathbf{x})$
    *   Material parameters: diffuse albedo $\mathbf{a}(\mathbf{x})$ and specular roughness $\gamma(\mathbf{x})$
    *   Neural visibility approximation $\tilde{V}_\phi(\mathbf{x}, \boldsymbol{\omega})$
    *   Expected termination depth $D(\mathbf{x}, \boldsymbol{\omega})$

### Data Structures
1.  **Shape MLP ($\text{MLP}_\theta$):**
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$.
    *   Output: Volume density $\sigma \in \mathbb{R}$.
    *   Implementation: Standard MLP. Surface normal $\mathbf{n}$ is computed as the negative normalized gradient of $\sigma$ with respect to $\mathbf{x}$:
        $$ \mathbf{n}(\mathbf{x}) = -\frac{\nabla_\mathbf{x} \sigma(\mathbf{x})}{\|\nabla_\mathbf{x} \sigma(\mathbf{x})\|} $$
        [S1]
2.  **Reflectance MLP ($\text{MLP}_\psi$):**
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$.
    *   Output: Diffuse albedo $\mathbf{a} \in \mathbb{R}^3$ and roughness $\gamma \in \mathbb{R}$.
    *   Reflectance Function $R$: A standard microfacet BRDF parameterized by $\mathbf{a}$, $\gamma$, and $\mathbf{n}$ [S1].
3.  **Visibility MLP ($\text{MLP}_\phi$):**
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$ and direction $\boldsymbol{\omega} \in \mathbb{S}^2$.
    *   Output: Visibility $\tilde{V} \in [0, 1]$ and expected termination depth $D \in \mathbb{R}^+$.
    *   Purpose: Approximates the true visibility $V$ and the distance to the first surface intersection, avoiding expensive line integrals [S1].

### Rendering Equation
The total radiance $L(\mathbf{c}, \boldsymbol{\omega}_o)$ observed from camera center $\mathbf{c}$ in direction $\boldsymbol{\omega}_o$ is the sum of direct and indirect components [S1]:

$$ L(\mathbf{c}, \boldsymbol{\omega}_o) = L_{\text{direct}} + L_{\text{indirect}} $$

**1. Direct Lighting Component:**
The direct lighting integral accounts for attenuation of the environment map $E$ by the volume density along incident directions $\boldsymbol{\omega}_i$. The true visibility $V$ is replaced by the neural approximation $\tilde{V}_\phi$ [S1]:

$$ L_{\text{direct}} = \int_0^\infty V(\mathbf{x}(t), \mathbf{c}) \sigma(\mathbf{x}(t)) \left[ \int_S \tilde{V}_\phi(\mathbf{x}(t), \boldsymbol{\omega}_i) E(\mathbf{x}(t), -\boldsymbol{\omega}_i) R(\mathbf{x}(t), \boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \, d\boldsymbol{\omega}_i \right] dt $$

Where:
*   $V(\mathbf{x}(t), \mathbf{c}) = \exp\left(-\int_0^t \sigma(\mathbf{x}(s)) \, ds\right)$ is the transmittance along the camera ray [S1].
*   $E(\mathbf{x}, -\boldsymbol{\omega}_i)$ is the radiance from the environment map in direction $-\boldsymbol{\omega}_i$.
*   $S$ is the unit sphere of incident directions.
*   Complexity: Linear in samples per ray $n$ (reduced from quadratic) [S1].

**2. Indirect Lighting Component:**
To handle one-bounce indirect illumination, NeRV approximates the outermost integral (accumulated radiance) by treating the volume as a hard surface at the expected termination depth $t_0 = D(\mathbf{c}, -\boldsymbol{\omega}_o)$ [S1]. Note: $t_0$ is derived from the visibility MLP's depth output, but the density sampling for the camera ray is independent [S1]. The indirect contribution is a spherical integral at point $\mathbf{x}(t_0)$:

$$ L_{\text{indirect}} = \int_S L(\mathbf{x}(t_0), -\boldsymbol{\omega}_i) R(\mathbf{x}(t_0), \boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \, d\boldsymbol{\omega}_i $$

Where $L(\mathbf{x}, -\boldsymbol{\omega}_i)$ is the incident radiance at $\mathbf{x}$ from direction $-\boldsymbol{\omega}_i$, which itself may contain direct and indirect components (recursive definition, but truncated to one-bounce in the paper's primary formulation) [S1].
*   Complexity: Reduced from cubic to a double spherical integral [S1].

### Training Procedure
1.  **Initialization:** Initialize $\text{MLP}_\theta$, $\text{MLP}_\psi$, and $\text{MLP}_\phi$.
2.  **Sampling:** For each training iteration, sample a batch of pixels, their camera rays, and stratified random points along each ray to approximate the volume rendering integral [S1].
3.  **Visibility Supervision:** The Visibility MLP is supervised to be consistent with the volume density samples observed during optimization. Specifically, it learns to predict visibility and termination depth that match the $\sigma$ values sampled by $\text{MLP}_\theta$ [S1].
4.  **Optimization:** Optimize all MLPs jointly using gradient descent. The loss is the difference between estimated and observed pixel colors [S1].
    *   Optimizer: Adam [S1].
    *   Learning Rate: Starts at $10^{-5}$, decays exponentially to $10^{-6}$ [S1].
    *   Hyperparameters: $\beta_1 = 0.9, \beta_2 = 0.999, \epsilon = 10^{-8}$ [S1].
    *   Iterations: 1 million [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
from jax import grad, value_and_grad

def shape_mlp(x, params_theta):
    """Returns volume density sigma."""
    # Standard MLP implementation
    return output_sigma

def reflectance_mlp(x, params_psi):
    """Returns diffuse albedo a and roughness gamma."""
    # Standard MLP implementation
    return output_a, output_gamma

def visibility_mlp(x, direction, params_phi):
    """Returns visibility V_tilde and expected termination depth D."""
    # Standard MLP implementation
    return output_V_tilde, output_D

def compute_normal(x, params_theta):
    """Computes surface normal as negative normalized gradient of sigma."""
    sigma_fn = lambda x: shape_mlp(x, params_theta)
    grad_sigma = grad(sigma_fn)(x)
    normal = -grad_sigma / jnp.linalg.norm(grad_sigma)
    return normal

def microfacet_brdf(a, gamma, n, omega_i, omega_o):
    """Computes reflectance R using standard microfacet model."""
    # Implementation of Walter et al. BRDF
    return R

def render_direct_lighting(ray_origin, ray_direction, params_theta, params_psi, params_phi, env_map):
    """
    Computes direct lighting component for a camera ray.
    Complexity: Linear in samples per ray.
    """
    # Sample points along the camera ray
    t_vals = sample_stratified_points(ray_origin, ray_direction)
    x_vals = ray_origin + t_vals[:, None] * ray_direction
    
    # Compute transmittance along camera ray
    sigma_vals = shape_mlp(x_vals, params_theta)
    delta_t = t_vals[1:] - t_vals[:-1]
    # Transmittance T_i = exp(-sum(sigma_j * delta_j))
    # Note: Standard NeRF transmittance calculation
    T = jnp.exp(-jnp.cumsum(sigma_vals[:-1] * delta_t))
    
    direct_radiance_sum = 0.0
    for x, sigma, T_i in zip(x_vals[:-1], sigma_vals[:-1], T):
        # Sample incident directions omega_i from sphere
        omega_is = sample_spherical_directions()
        
        # Compute incident radiance from environment map, attenuated by neural visibility
        # E_e(x, omega_i) = V_tilde(x, omega_i) * E(x, -omega_i)
        V_tilde = visibility_mlp(x, omega_is, params_phi)[:, 0] # Extract visibility
        E_vals = sample_env_map(x, -omega_is)
        E_attenuated = V_tilde * E_vals
        
        # Compute normal and BRDF
        n = compute_normal(x, params_theta)
        a, gamma = reflectance_mlp(x, params_psi)
        
        # Reflectance R(x, omega_i, omega_o)
        R_vals = microfacet_brdf(a, gamma, n, omega_is, ray_direction)
        
        # Integrate over sphere: sum(E_attenuated * R) / N_samples
        incident_radiance = jnp.mean(E_attenuated * R_vals)
        
        # Volume rendering term: T_i * (1 - exp(-sigma * delta)) * incident_radiance
        # Approximation: T_i * sigma * delta * incident_radiance for small delta
        # Or standard discrete volume rendering:
        alpha = 1.0 - jnp.exp(-sigma * delta_t) # Using delta from previous step or current
        # Note: The paper simplifies the direct integral in Eq 9 by replacing V with V_tilde
        # and integrating over the sphere at each point.
        direct_radiance_sum += T_i * alpha * incident_radiance
        
    return direct_radiance_sum

def render_indirect_lighting(ray_origin, ray_direction, params_theta, params_psi, params_phi):
    """
    Computes indirect lighting component.
    Approximates volume as hard surface at expected termination depth.
    """
    # Get expected termination depth from visibility MLP
    # D(c, -omega_o) -> depth along the viewing direction
    _, D_val = visibility_mlp(ray_origin, ray_direction, params_phi)
    
    # Point on surface
    x_surface = ray_origin + D_val * ray_direction
    
    # Compute normal and material at surface
    n = compute_normal(x_surface, params_theta)
    a, gamma = reflectance_mlp(x_surface, params_psi)
    
    # Sample incident directions for indirect lighting
    omega_is = sample_spherical_directions()
    
    # Incident radiance L(x, -omega_i)
    # In one-bounce approximation, this is often just the direct lighting at that point
    # or a cached value. The paper defines L(x, -omega_i) as incoming radiance.
    # For one-bounce, we can approximate L(x, -omega_i) using direct lighting calculation
    # at x_surface for direction -omega_i.
    
    # Simplified: Assume L(x, -omega_i) is computed via direct lighting at x_surface
    # towards direction -omega_i (which points into the scene).
    # However, the paper states: "approximate the outermost integral ... with a single 
    # point evaluation ... treating the volume as a hard surface".
    # So we need L(x_surface, -omega_i).
    
    # Let's assume we have a function to get incident radiance at a point
    L_incident = get_incident_radiance(x_surface, -omega_is, params_theta, params_psi, params_phi, env_map)
    
    # BRDF
    R_vals = microfacet_brdf(a, gamma, n, omega_is, ray_direction)
    
    # Integrate over sphere
    indirect_radiance = jnp.mean(L_incident * R_vals)
    
    return indirect_radiance

def get_incident_radiance(x, direction, params_theta, params_psi, params_phi, env_map):
    """Helper to get incident radiance at x from direction. 
    For one-bounce, this is direct lighting from env map at x."""
    # V_tilde for direction
    V_tilde = visibility_mlp(x, direction, params_phi)[:, 0]
    E = sample_env_map(x, -direction)
    return V_tilde * E

def loss_fn(params_theta, params_psi, params_phi, images, cameras, env_map):
    """Total loss including visibility consistency."""
    total_loss = 0.0
    for img, cam in zip(images, cameras):
        # Render image
        rendered_img = render_image(cam, params_theta, params_psi, params_phi, env_map)
        
        # Color loss
        color_loss = jnp.mean((rendered_img - img) ** 2)
        total_loss += color_loss
        
        # Visibility consistency loss
        # Supervise visibility MLP to be consistent with volume density samples
        # This implies comparing V_tilde predictions with actual visibility computed from sigma
        # along sampled rays.
        vis_loss = compute_visibility_consistency_loss(params_theta, params_phi)
        total_loss += vis_loss
        
    return total_loss

def compute_visibility_consistency_loss(params_theta, params_phi):
    """
    Ensures the visibility MLP predicts visibility consistent with the shape MLP's density.
    """
    # Sample random points and directions
    x_samples = random_3d_points()
    dir_samples = random_directions()
    
    # Get true visibility from density (expensive, but used for supervision signal)
    # In practice, this might be approximated or sampled sparsely.
    # The paper says: "supervised to be consistent with the volume density samples observed during optimization"
    # This likely means: if the ray is occluded by high density, V_tilde should be low.
    
    # Simplified supervision:
    # For each (x, dir), sample points along dir.
    # Compute transmittance from sigma.
    # Compare with V_tilde(x, dir).
    
    sigma_samples = shape_mlp(x_samples, params_theta)
    # ... compute true transmittance ...
    # ... compare with visibility_mlp(x_samples, dir_samples, params_phi) ...
    
    return jnp.mean((V_tilde_pred - V_true) ** 2)
```

### Parameters and Typical Values
*   **MLP Architecture:** Not specified in detail, but standard NeRF MLPs are used [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $10^{-5}$ decaying to $10^{-6}$ [S1].
*   **Iterations:** 1,000,000 [S1].
*   **Hardware:** 128 TPU cores, 1 day training time [S1].

### Edge Cases and Pitfalls
*   **Brute-force intractability:** Naive raymarching for light transport is intractable for environment lights or global illumination [S1].
*   **Geometry changes:** Existing visibility data structures (octrees, BVHs) are too expensive to rebuild during optimization as geometry changes [S1].
*   **Visibility MLP Consistency:** The visibility MLP must be supervised to match the density field; otherwise, it may predict incorrect occlusions [S1].

## Subtopics

### NeRV Evaluation Variants
*   **NeRV with Neural Visibility Fields (NeRV, NVF):** Uses the trained neural visibility MLP to estimate visibility and depth at test time. This is the standard, efficient rendering mode [S1].
*   **NeRV with Test-time Tracing (NeRV, Trace):** Uses brute-force ray tracing to estimate visibility to point light sources at test time. This produces sharper shadows but is slower. Both variants use the same training procedure [S1].

## Operating characteristics
*   **Time Complexity:**
    *   Direct Lighting: Linear in the number of samples per ray $n$ (reduced from quadratic) [S1].
    *   Indirect Lighting: Reduced from cubic to a double spherical integral (one spherical integral per point, evaluated at a single depth) [S1].
*   **Memory Complexity:** Standard NeRF memory usage plus the parameters for the Visibility MLP. The visibility MLP avoids storing large visibility maps or BVHs [S1].
*   **Performance:** Trained on 128 TPU cores for 1 million iterations (1 day) [S1].
*   **Strengths:** Supports arbitrary environment lighting and one-bounce indirect illumination; efficient visibility queries via MLP; differentiable end-to-end training [S1].
*   **Limitations:** Approximates indirect lighting with one-bounce; visibility MLP accuracy depends on training consistency; test-time tracing (for sharp shadows) is slow [S1].
*   **When to use:** When relighting with complex environment maps and indirect illumination is required, and brute-force light transport is too slow [S1].
*   **When not to use:** When real-time rendering with sharp shadows is required without precomputation (use NVF for speed, Trace for quality); when only direct lighting is needed (simpler methods suffice) [S1].

## Background
NeRV was introduced by Pratul P. Srinivasan, Boyang Deng, Xiuming Zhang, Matthew Tancik, Ben Mildenhall, and Jonathan T. Barron in 2021 [S1]. It extends the concept of Neural Reflectance Fields (which used voxel grids or single point lights) by replacing discrete geometry with continuous MLPs and introducing a neural visibility field to handle arbitrary lighting [S1]. The method builds on the differentiable volume rendering framework of NeRF [44-radiance-field-optimization.md] and addresses the intractability of light transport simulation in neural radiance fields [S1].

## Sources
[S1] NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, https://doi.org/10.1109/cvpr46437.2021.00741, fetched 2026-10-26
