# Dynamic Appearance Particle Neural Radiance Field

Also called: Dynamic Appearance Particle NeRF, Lagrangian dynamic model in NeRF

## Overview
Dynamic Appearance Particle Neural Radiance Field (DAP-NeRF) is a method for modeling dynamic scenes from monocular video data by superimposing a static grid-based radiance field with a dynamic component represented by moving appearance particles [S1]. It solves the problem of temporal variation in density and color without requiring multi-view stereo data or prior geometric knowledge of dynamic objects [S1]. Unlike ParticleNeRF, which encodes the entire scene with particles and requires multiple views per frame, DAP-NeRF applies particles only to dynamic elements and uses a grid-based field for efficient linear interpolation querying [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular video sequence (one camera view per time frame $t$), camera poses, and near/far bounds $s_n, s_f$.
*   **Outputs:** Rendered pixel color $C(\mathbf{r}, t)$ for a ray $\mathbf{r}$ at time $t$.

**Data Structures:**
1.  **Static Field $V_s(\mathbf{x})$:** A time-independent grid-based model (e.g., hash grid or triplane) storing features for density $\sigma$ and color $\mathbf{c}$ [S1].
2.  **Dynamic Field $V_{td}(\mathbf{x}, t)$:** An ensemble of particles. Each particle $i$ at time $t$ has:
    *   Position $\mathbf{p}_i(t) \in \mathbb{R}^3$.
    *   Appearance features $\mathbf{f}_i(t) \in \mathbb{R}^k$ (encoding density and color contributions).
    *   The particle positions are updated by a **Particle Motion Network** (NN$_{motion}$) [S1].
3.  **Mask $w_t(\mathbf{x})$:** A binary weight indicating if the dynamic field is effective at location $\mathbf{x}$ and time $t$ [S1].

**Formulas:**
The total feature field $V(\mathbf{x}, t)$ is a superposition of static and dynamic components [S1]:
$$
V(\mathbf{x}, t) = (1 - w_t(\mathbf{x}))V_s(\mathbf{x}) + w_t(\mathbf{x})V_{td}(\mathbf{x})
$$
where $V_s(\mathbf{x})$ is the static field and $V_{td}(\mathbf{x})$ is the dynamic field [S1].

The dynamic field $V_{td}$ is formulated as a convolution with a Dirac function to integrate over finite volumes [S1]:
$$
V_{td}(\mathbf{x}) = (V_{td} * \delta)(\mathbf{x}) = \int V_{td}(\mathbf{x}')\delta(\mathbf{x} - \mathbf{x}')d\nu(\mathbf{x}')
$$
where $\delta(\cdot)$ is the Dirac function [S1].

To enable quantization and efficient querying, the Dirac function $\delta$ is replaced with a kernel $W$ with finite support [S1]:
$$
W(\mathbf{x}) = 0 \quad \text{when } \|\mathbf{x}\| \text{ exceeds a small radius, and } \int W(\mathbf{x})d\nu(\mathbf{x}) = 1
$$
This leads to a finite-width kernel approximation [S1].

**Numbered Steps:**
1.  **Ray Casting:** For a pixel at time $t$, cast a ray $\mathbf{r}(s) = \mathbf{o} + s\mathbf{d}$ from camera center $\mathbf{o}$ with direction $\mathbf{d}$, where $s \in [s_n, s_f]$ [S1].
2.  **Discretization:** Sample points $\mathbf{x}_j$ along the ray [S1].
3.  **Dynamic Field Query:** For each sample point $\mathbf{x}_j$:
    *   Query the Particle Motion Network to get current particle positions $\mathbf{p}_i(t)$ and features $\mathbf{f}_i(t)$ [S1].
    *   Compute the contribution of nearby particles to $V_{td}(\mathbf{x}_j)$ using the kernel $W$ [S1].
    *   Determine $w_t(\mathbf{x}_j)$ based on particle presence/support [S1].
4.  **Static Field Query:** Query the static grid $V_s(\mathbf{x}_j)$ using linear interpolation [S1].
5.  **Superposition:** Combine fields: $V(\mathbf{x}_j, t) = (1 - w_t(\mathbf{x}_j))V_s(\mathbf{x}_j) + w_t(\mathbf{x}_j)V_{td}(\mathbf{x}_j)$ [S1].
6.  **Volume Rendering:** Compute density $\sigma$ and color $\mathbf{c}$ from $V(\mathbf{x}_j, t)$ and integrate along the ray using the standard volume rendering integral [S1]:
    $$
    C(\mathbf{r}, t) = \int_{s_n}^{s_f} T(s, t) \sigma(\mathbf{r}(s), t) c(\mathbf{r}(s), \mathbf{d}, t) ds
    $$
    where $T(s, t) = \exp \left( -\int_{s_n}^{s} \sigma(\mathbf{r}(s'), t) ds' \right)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_o, ray_d, time_t, static_grid, particles, kernel_W, mask_threshold):
    """
    Renders a single ray for DAP-NeRF.
    """
    # 1. Sample points along the ray
    samples = sample_along_ray(ray_o, ray_d, num_samples=128) # [N, 3]
    
    # 2. Query Static Field
    static_features = static_grid.query(samples) # [N, D]
    
    # 3. Query Dynamic Field
    # Get particle positions and features at time_t
    particle_positions, particle_features = particles.get_state(time_t) # [P, 3], [P, D]
    
    # Compute dynamic field contribution using kernel W
    # For each sample, sum contributions from nearby particles
    dynamic_features = zeros_like(static_features)
    for i in range(len(samples)):
        x = samples[i]
        # Find particles within kernel support radius
        nearby_particles = find_nearby(particle_positions, x, radius=kernel_W.radius)
        
        contribution = zeros_like(static_features[i])
        for p_idx in nearby_particles:
            p_pos = particle_positions[p_idx]
            p_feat = particle_features[p_idx]
            
            # Compute kernel weight
            dist = norm(x - p_pos)
            if dist <= kernel_W.radius:
                w = kernel_W.evaluate(dist)
                contribution += w * p_feat
        
        dynamic_features[i] = contribution
        
        # Determine mask value (1 if dynamic field is effective, 0 otherwise)
        # Effective if any particle contributes significantly
        if norm(contribution) > mask_threshold:
            mask_val = 1.0
        else:
            mask_val = 0.0
        dynamic_features[i] *= mask_val # Apply mask to features
        
    # 4. Superposition
    # Note: mask_val is per-sample, so we need to handle it carefully
    # The formula uses w_t(x). We assume mask_val is w_t(x)
    # Re-compute mask for each sample to be precise
    masks = zeros(len(samples))
    for i in range(len(samples)):
        x = samples[i]
        nearby_particles = find_nearby(particle_positions, x, radius=kernel_W.radius)
        has_dynamic = False
        for p_idx in nearby_particles:
            p_pos = particle_positions[p_idx]
            dist = norm(x - p_pos)
            if dist <= kernel_W.radius:
                w = kernel_W.evaluate(dist)
                if w > 0:
                    has_dynamic = True
                    break
        masks[i] = 1.0 if has_dynamic else 0.0
        
    # Superposition: V(x,t) = (1 - w_t(x))V_s(x) + w_t(x)V_td(x)
    combined_features = (1 - masks[:, None]) * static_features + masks[:, None] * dynamic_features
    
    # 5. Extract Density and Color
    sigma, color = extract_density_color(combined_features)
    
    # 6. Volume Rendering
    T = compute_transmittance(sigma)
    color_rendered = integrate_color(T, sigma, color)
    
    return color_rendered
```

**Parameters:**
*   **Kernel Radius:** Small radius for $W(\mathbf{x})$ [S1]. Not given in the sources.
*   **Mask Threshold:** Threshold for $w_t(\mathbf{x})$ to be 1 [S1]. Not given in the sources.
*   **Particle Motion Network:** NN$_{motion}$ updates particle positions [S1]. Architecture not specified in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The kernel approximation $\delta \to W$ must ensure $\int W(\mathbf{x})d\nu(\mathbf{x}) = 1$ to preserve field magnitude [S1].
*   **Boundaries:** The kernel $W(\mathbf{x}) = 0$ when $\|\mathbf{x}\|$ exceeds a small radius, so particles far from samples do not contribute [S1].
*   **Degenerate Input:** If no particles are near a sample, $w_t(\mathbf{x}) = 0$, and the static field is used [S1].

## Subtopics

### ParticleNeRF
Encodes the entire scene with particles and uses radius nearest neighbor searching; requires 20 camera views per time frame [S1].

### InstantNGP
An alternative framework that can integrate the Lagrangian representation [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of particles and samples. Querying dynamic field requires finding nearby particles for each sample [S1].
*   **Memory Complexity:** Stores static grid and particle states [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Handles monocular video data; no prior geometric knowledge required; efficient querying via linear interpolation for static part [S1].
*   **Limitations:** Requires tuning of kernel radius and mask threshold [S1].
*   **When to use:** Monocular dynamic scene reconstruction [S1].
*   **When not:** Multi-view static scenes (use standard NeRF) or scenes with known geometry (use deformation fields) [S1].

## Background
*   **Origin:** Proposed in [S1] (2025).
*   **History:** Builds on ParticleNeRF [S1] and static NeRF frameworks [S1].
*   **Usage:** Monocular video dynamic scene reconstruction [S1].

## Sources
[S1] Dynamic Appearance Particle Neural Radiance Field, https://doi.org/10.1109/tcsvt.2025.3540792, fetched 2026-11-05
