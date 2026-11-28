# Boundary Relaxation for Differentiable SDF Rendering

Also called: Relaxed Visibility Boundary, Thin Band Relaxation

## Overview
Boundary Relaxation is a method for differentiable rendering of Signed Distance Fields (SDFs) that addresses non-differentiable visibility discontinuities by relaxing the sharp visibility boundary into a thin band. This allows accurate gradient computation with respect to scene parameters while retaining surface representations for forward rendering, avoiding the high variance of area sampling or the complexity of boundary sampling data structures. It differs from boundary sampling methods (e.g., [Li et al. 2018, Zhang et al. 2020]) by using a simple thin band relaxation instead of complex data structures like 6D Hough trees, and from area sampling methods (e.g., [Loubet et al. 2019, Vicini et al. 2022]) by introducing a small, controlled bias in exchange for low variance and architectural simplicity. Unlike volume-based methods (e.g., IDR, NeRF) that relax the entire surface into a volume, this method retains a surface representation for forward rendering and only relaxes the boundary to a thin volume/band for discontinuity sampling.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene parameters $\theta$ (defining the SDF and materials), camera parameters, and the SDF function $SDF(\mathbf{x})$.
*   **Outputs:** Rendered image radiance and gradients $\frac{\partial L}{\partial \theta}$.

**Data Structures:**
*   **SDF:** A function mapping $\mathbf{x} \in \mathbb{R}^3 \to \mathbb{R}$. Convention: negative inside, positive outside.
*   **Normal Field:** $\mathbf{n}(\mathbf{x}) = \frac{\nabla SDF(\mathbf{x})}{\|\nabla SDF(\mathbf{x})\|}$.
*   **Normal Velocity:** $\mathbf{v}(\mathbf{x}) = -\frac{\partial \mathbf{x}}{\partial \theta} \cdot \mathbf{n}(\mathbf{x})$ (or similar definition based on parameter perturbation). *Note: The exact formula for $\mathbf{v}(\mathbf{x})$ is cut off in the source [S1], but it is defined as the change of the surface at $\mathbf{x}$ along its normal with respect to a perturbation of $\theta$.*

**Rendering Equation:**
The outgoing radiance $L_0$ at point $\mathbf{x}$ in direction $\omega_0$ is:
$$L_0(\mathbf{x}, \omega_0) = L_e(\mathbf{x}, \omega_0) + \int_{S^2} f_S(\mathbf{x}, \omega, \omega_0) L_i^\perp(\mathbf{x}, \omega) \, d\sigma(\omega)$$
where $L_e$ is emission, $f_S$ is BSDF, and $L_i^\perp(\mathbf{x}, \omega) = L_i(\mathbf{r}(\mathbf{x}, \omega), -\omega) |\omega \cdot \mathbf{n}(\mathbf{x})|$ is the visible incident radiance.

**Algorithm Steps:**
1.  **Identify Discontinuities:** For a given point $\mathbf{x}$ and direction $\omega$, identify the set of visibility-induced discontinuities $B \subset S^2$. Each direction $\omega \in B$ is associated with a distant silhouette point $\mathbf{x}^*$.
2.  **Relax Boundary:** Instead of treating the boundary $B$ as a sharp discontinuity, relax it into a thin band $A$. This corresponds to relaxing the surfaces to thin volumes for sampling discontinuities.
3.  **Sample Band:** Sample directions within the relaxed band $A$. The SDF allows easy sampling of this band because the normal field $\mathbf{n}(\mathbf{x})$ is defined everywhere in the neighborhood of the surface.
4.  **Compute Gradient:** Compute the gradient of the rendered image with respect to scene parameters $\theta$ by integrating over the relaxed band. The method embraces nonzero bias in exchange for low variance and architectural simplicity.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sdf_with_boundary_relaxation(scene_params, camera, sdf):
    """
    Renders an image using SDF and relaxes visibility boundaries for differentiability.
    """
    # 1. Cast rays and find intersections
    rays = generate_rays(camera)
    image = zeros_like(rays)
    gradients = zeros_like(scene_params)
    
    for ray in rays:
        # Find intersection with SDF surface
        hit_point, hit_normal = sdf_intersect(sdf, ray)
        
        if hit_point is None:
            continue
            
        # 2. Identify visibility discontinuities
        # B is the set of directions on the unit sphere where visibility changes
        # In practice, this involves finding silhouettes and self-occlusions
        discontinuity_directions = find_visibility_boundaries(hit_point, hit_normal, sdf, scene_params)
        
        # 3. Relax boundary to thin band A
        # The band A is a neighborhood around the discontinuity boundary B
        # The width of the band is a parameter, typically small
        band_width = 0.01 # Example value, not given in sources
        relaxed_band = relax_boundary(discontinuity_directions, band_width)
        
        # 4. Sample from the relaxed band
        # Sample directions within the band A
        samples = sample_band(relaxed_band, num_samples=16) # Example sample count
        
        # 5. Compute radiance and gradient
        radiance = 0.0
        grad_contrib = 0.0
        
        for sample_dir in samples:
            # Compute incident radiance for the sample direction
            # L_i_perp is the visible incident radiance
            L_i_perp = compute_incident_radiance(hit_point, sample_dir, sdf, scene_params)
            
            # BSDF evaluation
            bsdf = evaluate_bsdf(hit_point, sample_dir, ray.direction, scene_params)
            
            # Contribution to radiance
            radiance += bsdf * L_i_perp
            
            # Contribution to gradient
            # The gradient is computed by differentiating the integral over the relaxed band
            # This involves the normal velocity v(x) and the change in radiance across the boundary
            # Delta_L is the step change in incident radiance across the oriented boundary
            delta_L = compute_radiance_jump(hit_point, sample_dir, sdf, scene_params)
            
            # Gradient contribution depends on the relaxation width and the jump
            # Exact formula for gradient contribution is not fully specified in sources
            # [G] Assuming a simple approximation: grad ~ delta_L * v(x) / band_width
            grad_contrib += delta_L * compute_normal_velocity(hit_point, scene_params) / band_width
            
        image[ray.id] = radiance / len(samples)
        gradients += grad_contrib / len(samples)
        
    return image, gradients

def relax_boundary(discontinuity_directions, band_width):
    """
    Expands the sharp boundary B into a thin band A.
    """
    # [G] Implementation detail: Create a spherical cap or strip around each discontinuity direction
    # The exact geometric construction of the band is not specified in the sources.
    return discontinuity_directions # Placeholder for actual band construction

def compute_normal_velocity(hit_point, scene_params):
    """
    Computes the normal velocity v(x) = -dx/dtheta . n(x)
    """
    # [G] This requires the derivative of the hit point with respect to scene parameters
    # which is non-trivial for SDFs. The source mentions v(x) is defined as the change
    # of the surface at x along its normal with respect to a perturbation of theta.
    # Exact implementation details are not provided.
    return 0.0 # Placeholder
```

**Parameters:**
*   **Band Width:** The width of the relaxed boundary band $A$. Not given in the sources.
*   **Number of Samples:** Number of samples taken from the relaxed band. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the SDF being smooth and differentiable. Numerical errors in SDF evaluation can affect the normal field and thus the gradient computation.
*   **Boundaries:** The relaxation introduces a bias. The width of the band must be chosen carefully to balance bias and variance. Too wide a band increases bias; too narrow a band increases variance.
*   **Degenerate Input:** If the SDF is not well-defined or has singularities, the normal field may be undefined, leading to errors.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method is demonstrated to be robust and competitive with or superior to existing work in end-to-end inverse rendering tasks [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Low variance, architectural simplicity, retains surface representation for forward rendering.
*   **Limitations:** Introduces nonzero bias.
*   **When to use:** When low variance and simplicity are prioritized over zero bias, and when the underlying surface is represented by an SDF.
*   **When not to use:** When zero bias is strictly required, or when the surface is not represented by an SDF.

## Background
*   **Origin:** The method was introduced by Ziyi Zhang, Wenzel Jakob, and Steve Marschner [S1].
*   **History:** It addresses the problem of non-differentiable visibility discontinuities in physically based rendering, which breaks the assumption that the derivative of an integral equals the integral of the derivative.
*   **Usage:** It is used in inverse rendering tasks, generative AI, sensor design, and visual arts [S1].

## Sources
[S1] A Simple Approach to Differentiable Rendering of SDFs, https://doi.org/10.1145/3680528.3687573, fetched 2026-10-26
