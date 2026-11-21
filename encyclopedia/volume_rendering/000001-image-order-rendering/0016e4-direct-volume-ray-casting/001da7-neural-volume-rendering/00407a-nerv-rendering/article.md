# NeRV Rendering

Also called: NeRV ray marching, NeRV volume rendering

## Overview
NeRV (Neural Reflectance and Visibility Fields) is a neural volume rendering method that represents a 3D scene using separate neural networks for geometry, reflectance, and visibility to enable high-quality relighting and view synthesis [S1]. It differs from standard neural radiance fields (NeRF) by explicitly modeling surface normals and microfacet BRDF parameters, allowing for physically-based shading with direct and indirect illumination rather than relying solely on view-dependent color [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera origin $\mathbf{c}$ and direction $\boldsymbol{\omega}_o$.
*   Pre-trained neural network parameters:
    *   Shape MLP ($\text{MLP}_\theta$): Maps $\mathbf{x}$ to density $\sigma$ and normal $\mathbf{n}$.
    *   Reflectance MLP ($\text{MLP}_\psi$): Maps $\mathbf{x}$ to BRDF parameters $(a, \gamma)$.
    *   Visibility MLP ($\text{MLP}_\phi$): Maps $(\mathbf{x}, \boldsymbol{\omega}_i)$ to expected termination depth $\tilde{t}_0$.
*   Environment lighting map $E(\mathbf{x}, \boldsymbol{\omega})$.

**Outputs:**
*   Pixel color $V(\mathbf{c}, \boldsymbol{\omega}_o)$.

### Data Structures
The method relies on three Multi-Layer Perceptrons (MLPs). The specific architecture (layer sizes, activation functions) is not detailed in the provided source excerpts, but the input/output signatures are defined as follows [S1]:
1.  **Shape Network**: Input $\mathbf{x} \in \mathbb{R}^3$. Outputs scalar density $\sigma \in \mathbb{R}$ and vector normal $\mathbf{n} \in \mathbb{R}^3$.
2.  **Reflectance Network**: Input $\mathbf{x} \in \mathbb{R}^3$. Outputs scalar albedo $a \in \mathbb{R}$ and scalar roughness parameter $\gamma \in \mathbb{R}$.
3.  **Visibility Network**: Input $(\mathbf{x}, \boldsymbol{\omega}_i) \in \mathbb{R}^3 \times \mathbb{R}^3$. Outputs scalar expected depth $\tilde{t}_0 \in \mathbb{R}$.

### Algorithm Steps

1.  **Ray Sampling and Neural Query**:
    Draw $N=256$ stratified samples along the camera ray $\mathbf{x}(t) = \mathbf{c} - t\boldsymbol{\omega}_o$. For each sample point $\mathbf{x}(t)$, query the networks:
    *   Density: $\sigma = \text{MLP}_\theta(\mathbf{x}(t))$ [S1].
    *   Normal: $\mathbf{n} = \nabla_\mathbf{x} \text{MLP}_\theta(\mathbf{x}(t))$ [S1].
    *   BRDF Parameters: $(a, \gamma) = \text{MLP}_\psi(\mathbf{x}(t))$ [S1].

2.  **Direct Illumination Estimation**:
    For each point $\mathbf{x}(t)$ along the ray, estimate the direct illumination integral.
    *   Sample $K=12 \times 24 = 288$ directions $\boldsymbol{\omega}_i$ on the sphere using a fixed grid [S1].
    *   For each $\boldsymbol{\omega}_i$, retrieve environment lighting $E(\mathbf{x}(t), -\boldsymbol{\omega}_i)$ [S1].
    *   Query the visibility network for the predicted visibility/depth: $\tilde{V}_\phi = \tilde{t}_0 = \text{MLP}_\phi(\mathbf{x}(t), \boldsymbol{\omega}_i)$ [S1].
    *   Compute the microfacet BRDF value $R(\mathbf{x}(t), \boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ using the parameters $(a, \gamma)$ and normal $\mathbf{n}$ [S1].
    *   Integrate the product $E \cdot \tilde{V}_\phi \cdot R$ over the sphere to obtain the direct illumination contribution $L_{dir}$ [S1].

3.  **Indirect Illumination Estimation**:
    Estimate indirect illumination for points along the ray.
    *   Compute the expected camera ray termination depth $t_0 = D(\mathbf{c}, -\boldsymbol{\omega}_o)$ using the density samples from Step 1 [S1].
    *   At the termination point $\mathbf{x}(t_0)$, sample $M=128$ random directions $\boldsymbol{\omega}_i$ on the upper hemisphere [S1].
    *   For each $\boldsymbol{\omega}_i$, query the visibility network to get the expected secondary depth $\tilde{t}''_0 = \text{MLP}_\phi(\mathbf{x}(t_0), \boldsymbol{\omega}_i)$ [S1].
    *   Compute the secondary intersection point $\mathbf{x}_0(\tilde{t}''_0) = \mathbf{x}(t_0) + \tilde{t}''_0 \boldsymbol{\omega}_i$ [S1].
    *   Shade each secondary point $\mathbf{x}_0(\tilde{t}''_0)$ with direct illumination following the procedure in Step 2 [S1].
    *   Multiply the estimated indirect illumination incident at $\mathbf{x}(t_0)$ by the microfacet BRDF values $R(\mathbf{x}(t_0), \boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ and integrate over the hemisphere to produce the indirect illumination contribution $L_{ind}$ [S1].

4.  **Compositing**:
    Compute the total reflected radiance $L_r(\mathbf{x}(t), \boldsymbol{\omega}_o) = L_{dir} + L_{ind}$ at each sample [S1]. Composite along the ray using the quadrature rule:
    $$
    V(\mathbf{c}, \boldsymbol{\omega}_o) = \sum_t V(\mathbf{x}(t), \mathbf{c}) \alpha(\sigma(\mathbf{x}(t)) \delta) L_r(\mathbf{x}(t), \boldsymbol{\omega}_o)
    $$
    where:
    *   $\delta$ is the distance between samples along the ray [S1].
    *   $\alpha(z) = 1 - \exp(-z)$ [S1].
    *   $V(\mathbf{x}(t), \mathbf{c}) = \exp\left(-\sum_{s < t} \sigma(\mathbf{x}(s)) \delta\right)$ is the transmittance [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def render_nerv_ray(c, w_o, env_light, mlp_theta, mlp_psi, mlp_phi):
    """
    Renders a single ray using the NeRV method.
    
    Args:
        c: Camera origin (3,)
        w_o: Camera direction (3,)
        env_light: Function E(x, w) returning lighting value
        mlp_theta: Shape MLP returning (sigma, normal)
        mlp_psi: Reflectance MLP returning (albedo, roughness)
        mlp_phi: Visibility MLP returning expected depth
        
    Returns:
        Pixel color (scalar or RGB)
    """
    num_samples = 256
    # Generate stratified samples along the ray
    # Note: Exact stratification logic depends on implementation, 
    # assuming uniform spacing delta for simplicity in this high-level pseudocode
    # In practice, t values are sampled from a distribution.
    t_values = np.linspace(0, 100, num_samples) # Placeholder for ray bounds
    delta = t_values[1] - t_values[0]
    
    samples = []
    for t in t_values:
        x = c - t * w_o
        samples.append(x)
        
    # Step 1: Query networks
    densities = []
    normals = []
    brdf_params = []
    
    for x in samples:
        sigma, n = mlp_theta(x)
        a, gamma = mlp_psi(x)
        densities.append(sigma)
        normals.append(n)
        brdf_params.append((a, gamma))
        
    # Step 2 & 3: Compute Radiance at each point
    radiances = []
    
    for i, x in enumerate(samples):
        n = normals[i]
        a, gamma = brdf_params[i]
        
        # Direct Illumination
        # Sample 12x24 grid of directions
        dir_grid = generate_sphere_grid(12, 24)
        direct_light = 0.0
        
        for w_i in dir_grid:
            # Environment lighting
            E = env_light(x, -w_i)
            
            # Visibility prediction
            t_vis = mlp_phi(x, w_i)
            
            # BRDF evaluation
            R = microfacet_brdf(n, w_i, w_o, a, gamma)
            
            direct_light += E * t_vis * R
            
        # Integrate direct light over sphere (simplified sum)
        # Note: Proper integration requires weighting by solid angle
        L_dir = direct_light * (4 * math.pi / len(dir_grid))
        
        # Indirect Illumination
        # Compute expected termination depth for current ray
        # Using cumulative density to find t_0
        t_0 = compute_termination_depth(densities[:i], delta)
        x_term = c - t_0 * w_o
        
        # Sample 128 random directions on upper hemisphere
        hemi_dirs = sample_hemisphere(n, 128)
        indirect_light = 0.0
        
        for w_i in hemi_dirs:
            # Query visibility for secondary ray
            t_sec = mlp_phi(x_term, w_i)
            x_sec = x_term + t_sec * w_i
            
            # Shade secondary point with direct illumination
            # This requires querying networks at x_sec
            sigma_sec, n_sec = mlp_theta(x_sec)
            a_sec, gamma_sec = mlp_psi(x_sec)
            
            L_sec_dir = 0.0
            for w_i_sec in dir_grid:
                E_sec = env_light(x_sec, -w_i_sec)
                t_vis_sec = mlp_phi(x_sec, w_i_sec)
                R_sec = microfacet_brdf(n_sec, w_i_sec, w_o, a_sec, gamma_sec)
                L_sec_dir += E_sec * t_vis_sec * R_sec
            
            L_sec_dir *= (4 * math.pi / len(dir_grid))
            
            # Multiply by BRDF at primary point
            R_ind = microfacet_brdf(n, w_i, w_o, a, gamma)
            indirect_light += L_sec_dir * R_ind
            
        # Integrate indirect light
        L_ind = indirect_light * (2 * math.pi / len(hemi_dirs))
        
        L_r = L_dir + L_ind
        radiances.append(L_r)
        
    # Step 4: Compositing
    color = 0.0
    transmittance = 1.0
    
    for i, x in enumerate(samples):
        sigma = densities[i]
        alpha = 1.0 - math.exp(-sigma * delta)
        
        contribution = transmittance * alpha * radiances[i]
        color += contribution
        
        transmittance *= (1.0 - alpha)
        
    return color

def compute_termination_depth(densities_so_far, delta):
    """
    Computes the expected depth t_0 where the ray terminates.
    This is a simplified placeholder. The source states t_0 is computed 
    using density samples. In NeRV, this likely involves finding the 
    depth where transmittance drops below a threshold or integrating 
    the expected distance.
    """
    # Placeholder: Return a fixed depth or integrate based on source logic
    # Source [S1] says "using the density samples from Step 1"
    # A common approximation is the mean free path or first significant opacity
    return 10.0 # Not given in the sources.

def microfacet_brdf(n, w_i, w_o, a, gamma):
    """
    Placeholder for microfacet BRDF evaluation.
    Source [S1] mentions microfacet BRDF values R(x, w_i, w_o).
    Exact formula is not provided in the excerpts.
    """
    return 1.0 # Not given in the sources.

def generate_sphere_grid(res_x, res_y):
    """Generates a grid of directions on a sphere."""
    dirs = []
    for i in range(res_x):
        theta = 2 * math.pi * i / res_x
        for j in range(res_y):
            phi = math.pi * j / res_y
            x = math.sin(phi) * math.cos(theta)
            y = math.sin(phi) * math.sin(theta)
            z = math.cos(phi)
            dirs.append(np.array([x, y, z]))
    return dirs

def sample_hemisphere(n, count):
    """Samples random directions on the upper hemisphere defined by normal n."""
    # Placeholder implementation
    return [np.array([0,0,1])] * count # Not given in the sources.
```

### Parameters
*   **Stratified samples per ray**: 256 [S1].
*   **Environment lighting grid**: $12 \times 24$ directions [S1].
*   **Indirect hemisphere samples**: 128 random directions [S1].

### Edge Cases and Pitfalls
*   **Visibility MLP Accuracy**: The method relies on $\text{MLP}_\phi$ to predict termination depths for both visibility occlusion and indirect ray bounces. If the visibility network is inaccurate, the shading will be incorrect (e.g., self-occlusion errors or incorrect bounce points) [S1].
*   **Normal Estimation**: Normals are derived as the gradient of the shape network $\nabla_\mathbf{x} \text{MLP}_\theta$. This requires the shape network to be smooth and differentiable; noisy density fields may produce unstable normals [S1].
*   **Integration Error**: The integration of direct and indirect illumination uses discrete sampling (grid and random). High-frequency lighting or sharp BRDF lobes may require more samples to avoid noise, though the fixed counts (288 and 128) are specified as the standard [S1].

## Operating characteristics

*   **Time Complexity**: High. Each pixel requires 256 forward passes through the shape and reflectance networks. Additionally, for every sample, direct illumination requires 288 queries to the visibility network and environment lighting lookup, and indirect illumination requires 128 queries to the visibility network plus shading of secondary points. This makes it significantly slower than standard NeRF which typically does not perform explicit ray tracing for lighting.
*   **Memory Complexity**: Depends on the size of the three MLPs ($\theta, \psi, \phi$). No large voxel grids or hash tables are mentioned in the provided excerpts, suggesting a compact representation compared to grid-based methods [S1].
*   **Strengths**: Enables physically-based relighting and view synthesis with explicit surface normals and BRDFs, allowing for realistic indirect illumination [S1].
*   **Limitations**: Computationally expensive due to the nested sampling for lighting and the need for three separate networks [S1].

## Background

NeRV (Neural Reflectance and Visibility Fields) was introduced by [S1]. It extends neural volume rendering by decoupling geometry, reflectance, and visibility into separate neural fields to facilitate relighting.

## Sources

[S1] NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, https://doi.org/10.1109/cvpr46437.2021.00741, fetched 2026-10-26
