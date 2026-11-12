# NeRV (Neural Reflectance and Visibility Fields)

Also called: Neural Reflectance and Visibility Fields

## Overview

NeRV is a differentiable inverse rendering method that reconstructs a 3D scene representation from images under unconstrained, known lighting, enabling view synthesis and relighting with indirect illumination [S1]. It extends the standard emission-absorption NeRF model by representing the scene as a volumetric field of oriented particles that absorb and reflect light, parameterized by neural networks [S1]. Unlike its sibling [Neural Radiance Fields](44-radiance-field-optimization.md) which only models direct emission, or [Neural Reflectance Fields](6755-neural-radiosity.md) which is limited to single-point-light direct illumination, NeRV supports arbitrary environment lighting and one-bounce global illumination [S1]. It achieves this tractability by introducing a learned "neural visibility field" that approximates visibility and surface termination depths, reducing the complexity of light transport simulation from quadratic/cubic to linear/double-spherical integral [S1].

## How it works

**Inputs:** A set of images of a scene with known, unconstrained lighting (environment map) and known camera poses.
**Outputs:** A 3D volumetric representation capable of rendering novel views under arbitrary lighting, including indirect illumination.

**Data Structures:**
The method uses two Multi-Layer Perceptrons (MLPs) parameterized by weights $\theta$ and $\psi$:
1.  **Shape MLP ($MLP_\theta$):**
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$.
    *   Output: Volume density $\sigma \in \mathbb{R}$.
2.  **Reflectance MLP ($MLP_\psi$):**
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$.
    *   Output: Material parameters $\mathbf{a} \in \mathbb{R}^3$ (diffuse albedo) and $\gamma \in \mathbb{R}$ (specular roughness).
3.  **Visibility MLP ($MLP_\phi$):**
    *   Input: 3D position $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\boldsymbol{\omega} \in \mathbb{S}^2$.
    *   Output: A tuple $(\tilde{V}, \tilde{t}_0)$, where $\tilde{V} \in [0, 1]$ is the estimated visibility and $\tilde{t}_0 \in \mathbb{R}^+$ is the estimated distance to the first surface intersection (expected termination depth).

**Surface Normal:**
The surface normal $\mathbf{n}$ is not an output of the MLP but is computed as the negative normalized gradient of the density $\sigma$ with respect to position $\mathbf{x}$, using automatic differentiation [S1]:
$$
\mathbf{n}(\mathbf{x}) = -\frac{\nabla_\mathbf{x} \sigma(\mathbf{x})}{\|\nabla_\mathbf{x} \sigma(\mathbf{x})\|}
$$

**Reflectance Function:**
The reflectance $R$ is a standard microfacet BRDF [S1]. At any location $\mathbf{x}$, it is parameterized by the diffuse albedo $\mathbf{a}$, roughness $\gamma$, and the computed normal $\mathbf{n}$ [S1].

**Rendering Equation:**
The total radiance $L(\mathbf{c}, \boldsymbol{\omega}_o)$ observed from camera center $\mathbf{c}$ in direction $\boldsymbol{\omega}_o$ is the sum of direct and indirect lighting components [S1]:

$$
L(\mathbf{c}, \boldsymbol{\omega}_o) = L_{\text{direct}} + L_{\text{indirect}}
$$

1.  **Direct Lighting ($L_{\text{direct}}$):**
    The integral along the camera ray $\mathbf{x}(t)$, attenuated by the true visibility $V(\mathbf{x}(t), \mathbf{c})$ and the neural visibility approximation $\tilde{V}_\phi$ for the light direction [S1]:
    $$
    L_{\text{direct}} = \int_0^\infty V(\mathbf{x}(t), \mathbf{c}) \sigma(\mathbf{x}(t)) \left[ \int_S \tilde{V}_\phi(\mathbf{x}(t), \boldsymbol{\omega}_i) E(\mathbf{x}(t), -\boldsymbol{\omega}_i) R(\mathbf{x}(t), \boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \, d\boldsymbol{\omega}_i \right] dt
    $$
    Where $E$ is the environment map radiance, $S$ is the sphere of directions, and $\boldsymbol{\omega}_i$ are light directions [S1]. The term $V(\mathbf{x}(t), \mathbf{c})$ is the standard transmittance along the camera ray:
    $$
    V(\mathbf{x}(t), \mathbf{c}) = \exp\left(-\int_0^t \sigma(\mathbf{x}(s)) \, ds\right)
    $$

2.  **Indirect Lighting ($L_{\text{indirect}}$):**
    Approximated by treating the volume as a hard surface at the expected termination depth $t_0$ of the camera ray [S1]. The outermost integral is replaced by a single point evaluation at $\mathbf{x}(t_0)$:
    $$
    L_{\text{indirect}} = \int_S L(\mathbf{x}(t_0), -\boldsymbol{\omega}_i) R(\mathbf{x}(t_0), \boldsymbol{\omega}_i, \boldsymbol{\omega}_o) \, d\boldsymbol{\omega}_i
    $$
    Here, $L(\mathbf{x}(t_0), -\boldsymbol{\omega}_i)$ is the incident radiance at the surface point, which itself may include further bounces or direct lighting depending on the implementation depth, but NeRV specifically targets "one-bounce" indirect illumination in this formulation [S1]. The termination depth $t_0$ is derived from the camera ray sampling of $\sigma$, not the visibility MLP's prediction for the camera direction, to ensure consistency with the sampled density [S1].

**Training Procedure:**
1.  **Initialization:** Initialize $MLP_\theta$, $MLP_\psi$, and $MLP_\phi$.
2.  **Sampling:** For each training step, sample a batch of pixels, their camera rays, and stratified random points along those rays to approximate the integrals [S1].
3.  **Visibility Supervision:** The visibility MLP $MLP_\phi$ is supervised to be consistent with the volume density samples observed during optimization [S1]. Specifically, it learns to predict the visibility and termination depth that would result from raymarching through the current $\sigma$ field.
4.  **Optimization:** Optimize all MLP weights ($\theta, \psi, \phi$) via gradient descent to minimize the error between the rendered color and the observed input image colors [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_nev(camera_ray_origin, camera_ray_direction, env_map, 
               shape_mlp, reflectance_mlp, visibility_mlp, 
               num_samples_ray, num_samples_light):
    """
    Renders a single pixel using NeRV.
    
    Args:
        camera_ray_origin: 3D vector c
        camera_ray_direction: 3D vector omega_o
        env_map: Function E(x, omega) returning radiance from direction omega at x
        shape_mlp: MLP_theta returning sigma(x)
        reflectance_mlp: MLP_psi returning (albedo a, roughness gamma)
        visibility_mlp: MLP_phi returning (V_tilde, t_tilde)
        num_samples_ray: N, number of samples along camera ray
        num_samples_light: L, number of samples for light integration
        
    Returns:
        color: 3D vector (R, G, B)
    """
    
    # 1. Sample points along the camera ray
    t_values = stratified_sample(0, infinity, num_samples_ray)
    x_points = camera_ray_origin + t_values * camera_ray_direction
    
    # 2. Evaluate density and transmittance along the camera ray
    sigmas = shape_mlp(x_points)
    
    # Compute transmittance T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # delta_j is the distance between sample points
    deltas = t_values[1:] - t_values[:-1]
    deltas = torch.cat([torch.tensor([1e-10]), deltas]) # Guard for first element
    
    # Cumulative sum of sigma * delta
    sigma_deltas = sigmas * deltas
    alpha = 1.0 - torch.exp(-sigma_deltas)
    transmittance = torch.cumprod(torch.exp(-sigma_deltas), dim=0)
    transmittance = torch.cat([torch.tensor([1.0]), transmittance[:-1]])
    
    # 3. Compute Direct Lighting Component
    direct_radiance_sum = 0.0
    
    for i in range(num_samples_ray):
        x_i = x_points[i]
        sigma_i = sigmas[i]
        T_i = transmittance[i]
        alpha_i = alpha[i]
        
        # If density is negligible, skip
        if sigma_i < 1e-5:
            continue
            
        # Evaluate material properties at x_i
        albedo, roughness = reflectance_mlp(x_i)
        normal = compute_normal(shape_mlp, x_i) # Gradient of sigma
        
        # Evaluate incident direct lighting
        # Sample L directions from environment map
        light_directions = sample_sphere(num_samples_light)
        direct_light_contribution = 0.0
        
        for omega_i in light_directions:
            # Query neural visibility for light direction
            V_tilde, _ = visibility_mlp(x_i, omega_i)
            
            # Get environment radiance
            E_val = env_map(x_i, -omega_i)
            
            # Evaluate BRDF
            R_val = microfacet_brdf(normal, omega_i, camera_ray_direction, albedo, roughness)
            
            # Accumulate: V_tilde * E * R
            direct_light_contribution += V_tilde * E_val * R_val
            
        # Average over light samples
        direct_light_contribution /= num_samples_light
        
        # Add to ray integral: T_i * alpha_i * direct_light_contribution
        # Note: The integral is approximated by sum. 
        # The term sigma(x(t)) dt is handled by the alpha term in standard DVR,
        # but NeRV formulation in Eq 9 keeps sigma explicitly. 
        # Standard DVR: T_i * (1 - exp(-sigma * delta)) * color.
        # Here, the "color" is the direct lighting integral.
        
        # Re-calculate alpha for the specific segment if using discrete sum
        # alpha_i = 1 - exp(-sigma_i * delta_i)
        # contribution = T_i * alpha_i * direct_light_contribution
        
        # Using the provided Eq 9 structure which integrates sigma * V * ... dt
        # This is equivalent to the standard DVR weight if we consider the integrand
        # as sigma * (Direct Lighting Integral).
        
        direct_radiance_sum += T_i * (1.0 - torch.exp(-sigma_i * deltas[i])) * direct_light_contribution

    # 4. Compute Indirect Lighting Component
    # Approximate the volume as a hard surface at the expected termination depth t0
    # t0 is derived from the camera ray sampling of sigma, not the visibility MLP
    # Find the depth where the ray "terminates" (e.g., where transmittance drops low or max depth)
    # A common heuristic for t0 in this context is the depth of the last significant sample
    # or the mean depth weighted by alpha.
    
    # Calculate expected termination depth t0 from the sampled points
    # t0 = sum(t_i * alpha_i) / sum(alpha_i)
    weights = alpha * (1.0 - torch.exp(-sigmas * deltas)) # Actually alpha is already 1-exp...
    # Let's use the standard alpha computed above: alpha_i = 1 - exp(-sigma_i * delta_i)
    # t0 = sum(t_i * alpha_i) / sum(alpha_i)
    
    valid_mask = alpha > 1e-5
    if torch.sum(valid_mask) > 0:
        t0 = torch.sum(t_values * alpha) / torch.sum(alpha)
        x_surface = camera_ray_origin + t0 * camera_ray_direction
        
        # Evaluate material at surface
        albedo_s, roughness_s = reflectance_mlp(x_surface)
        normal_s = compute_normal(shape_mlp, x_surface)
        
        # Evaluate indirect lighting: Integral of L(x, -omega_i) * R(...) d omega_i
        # L(x, -omega_i) is the incident radiance. In one-bounce approximation,
        # this is often just the direct lighting at that point, or it can be recursive.
        # NeRV Eq 10 implies L is the accumulated radiance. 
        # For one-bounce, L(x, -omega_i) is the direct lighting arriving at x from -omega_i.
        
        indirect_light_contribution = 0.0
        for omega_i in light_directions:
            # Incident radiance from direction -omega_i (which is omega_i in the integral notation of Eq 10? 
            # Eq 10: L(x(t'), -omega_i). If omega_i is the direction FROM the light TO the point,
            # then the direction OF the light arriving is -omega_i.
            # The BRDF takes incoming omega_i and outgoing omega_o.
            
            # Get direct lighting at this surface point for direction omega_i
            # We can reuse the visibility MLP
            V_tilde, _ = visibility_mlp(x_surface, omega_i)
            E_val = env_map(x_surface, -omega_i)
            R_val = microfacet_brdf(normal_s, omega_i, camera_ray_direction, albedo_s, roughness_s)
            
            # The incident radiance L is V_tilde * E (assuming no other bounces for one-bounce approx)
            L_incident = V_tilde * E_val
            
            indirect_light_contribution += L_incident * R_val
            
        indirect_light_contribution /= num_samples_light
    else:
        indirect_light_contribution = 0.0

    return direct_radiance_sum + indirect_light_contribution

def compute_normal(shape_mlp, x):
    """
    Computes the negative normalized gradient of density.
    """
    x.requires_grad_(True)
    sigma = shape_mlp(x)
    grad = torch.autograd.grad(sigma, x, grad_outputs=torch.ones_like(sigma), create_graph=True)[0]
    normal = -grad / torch.norm(grad)
    return normal
```

## Subtopics

### [NeRV Evaluation Variants](10273-nerv-evaluation-variants.md)
Inference-time strategies that trade rendering speed for shadow fidelity or material generality, addressing the base method's soft shadows and diffuse-only assumption.

## Operating characteristics

**Time Complexity:**
-   **Direct Lighting:** Reduced from quadratic $O(n \cdot l)$ (where $n$ is samples per ray, $l$ is light sources/directions) to linear $O(n \cdot l)$? No, the source states it reduces complexity from quadratic to linear in samples per ray by using neural visibility approximations instead of line integrals [S1]. Specifically, replacing the visibility line integral (which would be $O(n)$ per light direction) with a single MLP query ($O(1)$) reduces the cost per ray-light pair from $O(n)$ to $O(1)$, making the total direct lighting cost $O(n \cdot l)$ where $n$ is camera ray samples and $l$ is light samples.
-   **Indirect Lighting:** Reduced from cubic $O(n \cdot l \cdot d)$ (where $d$ is indirect directions) to a double spherical integral $O(l \cdot d)$ at a single point [S1].

**Memory Complexity:**
-   Requires storing the weights of three MLPs ($\theta, \psi, \phi$).
-   Does not require storing intermediate visibility data structures (like BVHs or distance transforms) which are prohibitively expensive to rebuild during optimization [S1].

**Strengths:**
-   Supports arbitrary environment lighting and one-bounce indirect illumination [S1].
-   Tractable optimization with dynamic geometry changes because visibility is learned via MLPs rather than rebuilt discrete structures [S1].
-   Enables relighting and view synthesis simultaneously [S1].

**Limitations:**
-   Produces soft shadows due to the learned visibility approximation [S1].
-   Assumes diffuse surfaces in the base formulation (though variants exist) [S1].
-   Indirect lighting is limited to one-bounce approximation [S1].
-   Brute-force light transport simulation through NeRF's volumetric representation is intractable for environment lights or global illumination without this approximation [S1].

## Background

NeRV was introduced by Pratul P. Srinivasan, Boyang Deng, Xiuming Zhang, Matthew Tancik, Ben Mildenhall, and Jonathan T. Barron in 2021 [S1]. It extends the work on Neural Reflectance Fields, which was limited to single-point-light direct illumination, by introducing a neural visibility field to handle complex lighting and global illumination [S1]. The method addresses the intractability of brute-force light transport simulation in volumetric neural representations by replacing expensive visibility queries with a learned MLP [S1].

## Sources

[S1] NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, https://doi.org/10.1109/cvpr46437.2021.00741, fetched 2026-10-26
