# Single Scattering Model

Also called: Heidelberg ray tracing model, Kajiya and Von Herzen model, Single Scattering Radiance Transport, Single-Scattering Participating Media Rendering, Two-Pass Single Scattering

## Overview
The Single Scattering Model approximates light transport in participating media by accounting for light that travels directly from a source to a point in the volume and scatters exactly once toward the observer, while ignoring multiple scattering events and internal emission [S1, S2][S3, unofficial]. It solves the problem of rendering volumetric effects like smoke or clouds efficiently by assuming that for low-density media, single scattering dominates the radiance contribution [S2]. It sets itself apart from the [Absorption-only model](839-absorption-only-model.md) by including in-scattering, and from the full [Radiative Transfer Equation Formulation](1194-radiative-transfer-equation-formulation.md) by truncating the scattering series after the first order, thereby enabling faster computation at the cost of accuracy in dense media [S1][S3, unofficial].

## How it works
The model computes the outgoing radiance $L_{out}$ at a viewpoint $x$ in direction $\boldsymbol{\omega}$ as the sum of reduced incident radiance $L_{ri}$ (light transmitted directly through the volume) and single-scattering radiance $L_{ss}$ (light scattered once into the view ray) [S1, S2]:

$$ L_{out}(x, \boldsymbol{\omega}) = L_{ri}(x, \boldsymbol{\omega}) + L_{ss}(x, \boldsymbol{\omega}) $$

### Inputs and Outputs
*   **Inputs:**
    *   Viewpoint position $x$ and viewing direction $\boldsymbol{\omega}$ [S2].
    *   Volume density field $\rho(\mathbf{p})$ or optical property fields defined over the volume [S2].
    *   Extinction coefficient $\kappa_t(\mathbf{p})$, absorption coefficient $\kappa_a(\mathbf{p})$, and scattering coefficient $\kappa_s(\mathbf{p})$ [S2]. Note: $\kappa_t = \kappa_a + \kappa_s$ [S2].
    *   Scattering albedo $\Omega(\mathbf{p}) = \kappa_s(\mathbf{p}) / \kappa_t(\mathbf{p})$ [S2].
    *   Phase function $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ describing the angular distribution of scattered light [S3, unofficial].
    *   Incident radiance field $L_{in}(\boldsymbol{\omega}')$ from light sources or environment maps [S2].
    *   Volume boundary entry point $x_a$ and exit point $x_b$ for the viewing ray [S2].
*   **Outputs:**
    *   Outgoing radiance $L_{out}(x, \boldsymbol{\omega})$ [S2].

### Data Structures
*   **Density/Optical Field:** A 3D grid or voxel structure storing $\kappa_t, \kappa_a, \kappa_s, \Omega$ at each position $\mathbf{p}$ [S2].
*   **Phase Function:** A function or lookup table evaluating $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ based on the angle between viewing direction $\boldsymbol{\omega}$ and light direction $\boldsymbol{\omega}'$ [S3, unofficial].

### Formulas
1.  **Transmittance:** The fraction of light surviving transmission between two points $x_u$ and $x_v$ along a ray is given by the Beer-Lambert law [S2][S3, unofficial]:
    $$ \tau(x_u, x_v) = e^{-\int_{D_u}^{D_v} \kappa_t(x_w) dw} $$
    where $D_u$ and $D_v$ are the distances from the viewpoint $x$ to positions $x_u$ and $x_v$ respectively, and $x_w$ is a point along the path [S2].

2.  **Reduced Incident Radiance ($L_{ri}$):** Ambient light $L_{in}(\boldsymbol{\omega})$ attenuated by the volume between the boundary $x_a$ and the viewpoint $x$ [S2]:
    $$ L_{ri}(x, \boldsymbol{\omega}) = L_{in}(\boldsymbol{\omega}) \tau(x_a, x) $$

3.  **Single-Scattering Radiance ($L_{ss}$):** The integral of light scattered from all directions $\boldsymbol{\omega}'$ into the viewing direction $\boldsymbol{\omega}$ at points $x_u$ along the ray segment $[x_a, x_b]$ [S2]:
    $$ L_{ss}(x, \boldsymbol{\omega}) = \int_{D_b}^{D_a} \tau(x_u, x) \kappa_s(x_u) \left( \int_{S^2} p(\boldsymbol{\omega}, \boldsymbol{\omega}') L_{in}(\boldsymbol{\omega}') \tau(x, x_u, \boldsymbol{\omega}') d\boldsymbol{\omega}' \right) du $$
    *Note: The inner integral represents the illumination $I(x_u)$ reaching point $x_u$ from all directions, attenuated by the path from the source to $x_u$. The term $\tau(x_u, x)$ attenuates this scattered light as it travels to the viewer [S3, unofficial].*

    Alternatively, using the shading rule $g(X)$ from the two-pass formulation [S3, unofficial]:
    $$ g(X) = r(X, \boldsymbol{\omega}, \boldsymbol{\omega}') I(X) $$
    where $r(X, \boldsymbol{\omega}, \boldsymbol{\omega}') = a \kappa_t(X) p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ is the bidirectional reflection distribution function (BRDF) for scattering, $a$ is the albedo (equivalent to $\Omega$), and $I(X)$ is the incoming illumination [S3, unofficial].

### Parameters
*   $\kappa_t$: Total extinction coefficient ($m^{-1}$) [S2].
*   $\kappa_s$: Scattering coefficient ($m^{-1}$) [S2].
*   $\kappa_a$: Absorption coefficient ($m^{-1}$) [S2].
*   $\Omega$: Scattering albedo (dimensionless, $0 \le \Omega \le 1$) [S2].
*   $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$: Phase function (dimensionless). For water droplets, it peaks in the forward scattering direction [S3, unofficial].
*   $L_{in}(\boldsymbol{\omega}')$: Incident radiance from direction $\boldsymbol{\omega}'$ [S2].

### Steps
The implementation typically follows a two-pass approach or a direct integration along the viewing ray [S3, unofficial].

1.  **Compute Transmittance:** For a given viewing ray from $x_a$ to $x_b$, calculate the transmittance $\tau(x_u, x)$ from any point $x_u$ on the ray to the viewpoint $x$ by integrating $\kappa_t$ along the path [S2].
2.  **Compute Incoming Illumination ($I(x_u)$):** For each point $x_u$ on the ray, calculate the total incident radiance from all light directions. This involves integrating $L_{in}(\boldsymbol{\omega}')$ over the unit sphere $S^2$, weighted by the phase function $p(\boldsymbol{\omega}', \boldsymbol{\omega}_{light})$ and the transmittance from the light source to $x_u$ [S2][S3, unofficial].
    *   For simple point or parallel lights, this can be calculated directly [S1].
    *   For environment map lighting, importance sampling may be used to estimate the integral [S1].
3.  **Compute Single-Scattering Contribution:** Integrate the product of the transmittance to the viewer $\tau(x_u, x)$, the scattering coefficient $\kappa_s(x_u)$, and the incoming illumination $I(x_u)$ along the ray segment $[x_a, x_b]$ [S2].
    $$ L_{ss} = \int_{x_a}^{x_b} \tau(x_u, x) \kappa_s(x_u) I(x_u) du $$
4.  **Compute Reduced Incident Radiance:** Calculate $L_{ri} = L_{in}(\boldsymbol{\omega}) \tau(x_a, x)$ [S2].
5.  **Combine:** Sum the components: $L_{out} = L_{ri} + L_{ss}$ [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering_render(viewpoint, ray_origin, ray_direction, volume_field, light_source):
    """
    Renders a single ray using the single scattering model.
    
    Args:
        viewpoint: Position of the camera.
        ray_origin: Entry point of the ray into the volume (x_a).
        ray_direction: Unit vector of the viewing ray.
        volume_field: Object providing kappa_t, kappa_s, and phase function at any point.
        light_source: Object providing incident radiance L_in(omega) and position.
        
    Returns:
        L_out: Total outgoing radiance.
    """
    
    # 1. Find exit point x_b (Not given in sources: requires ray-volume intersection)
    ray_exit = ray_volume_intersection(ray_origin, ray_direction, volume_field)
    
    # 2. Compute Reduced Incident Radiance L_ri
    # Transmittance from entry to viewpoint
    tau_entry_to_view = compute_transmittance(ray_origin, viewpoint, volume_field)
    L_ri = light_source.get_ambient_radiance(ray_direction) * tau_entry_to_view
    
    # 3. Compute Single-Scattering Radiance L_ss
    # Integrate along the ray from x_a to x_b
    L_ss = 0.0
    
    # Discretize the ray for numerical integration (e.g., ray marching)
    num_steps = 100
    step_size = distance(ray_origin, ray_exit) / num_steps
    
    for i in range(num_steps):
        # Current point x_u on the ray
        dist_from_origin = i * step_size
        x_u = ray_origin + ray_direction * dist_from_origin
        
        # Transmittance from x_u to viewpoint
        tau_u_to_view = compute_transmittance(x_u, viewpoint, volume_field)
        
        # Scattering coefficient at x_u
        kappa_s = volume_field.get_scattering_coefficient(x_u)
        
        # Incoming illumination I(x_u)
        # Integrate over all light directions omega'
        # For a single point light at position L_pos with intensity I_light:
        # Direction to light: omega_l = normalize(L_pos - x_u)
        # Transmittance from light to x_u: tau_light_to_u
        # Note: Sources mention environment maps require importance sampling [S1]
        # Here we assume a simple point light for clarity
        light_dir = normalize(light_source.position - x_u)
        tau_light_to_u = compute_transmittance(light_source.position, x_u, volume_field)
        
        # Incident radiance from light direction
        L_in = light_source.intensity * tau_light_to_u
        
        # Phase function p(omega_view, omega_light)
        phase = volume_field.phase_function(ray_direction, light_dir)
        
        # Contribution from this point
        # dL_ss = tau_u_to_view * kappa_s * L_in * phase * ds
        L_ss += tau_u_to_view * kappa_s * L_in * phase * step_size
        
    # 4. Combine
    L_out = L_ri + L_ss
    return L_out

def compute_transmittance(p1, p2, volume_field):
    """
    Computes transmittance between p1 and p2 by integrating kappa_t.
    """
    # Not given in sources: specific integration method (e.g., trapezoidal rule)
    # Approximate by sampling along the segment
    num_samples = 10
    total_extinction = 0.0
    dist = distance(p1, p2)
    step = dist / num_samples
    
    for i in range(num_samples):
        p_mid = p1 + (i + 0.5) * step * normalize(p2 - p1)
        kappa_t = volume_field.get_extinction_coefficient(p_mid)
        total_extinction += kappa_t * step
        
    return exp(-total_extinction)
```

### Edge Cases and Pitfalls
*   **High Density/Albedo:** The model is invalid if the albedo or density is high, as multiple scattering becomes significant and is ignored [S3, unofficial].
*   **Emission:** The model ignores emitted radiance, so it cannot render self-luminous volumes like flames without modification [S1, S2].
*   **Surface Interactions:** The model assumes pure volume scenes and does not handle surface-media interactions [S2].
*   **Precision:** Numerical integration of the transmittance and scattering terms requires sufficient sampling to avoid artifacts, especially in heterogeneous media [S2].

## Subtopics

### [Single Scattering Propagator](1299-single-scattering-propagator.md)
Models light transport with zero or exactly one scattering event, separating direct and singly-scattered light. It differs from the general model by using precomputed propagation fields and rigorous attenuation integration, unlike the Utah Approximation which ignores occlusion.

### [Single Scattering (Blinn Model)](4581-single-scattering-blinn-model.md)
A specific approximation for uniform, low-density media using Blinn's 1982 formulas. It differs from the general model by using a direct ray-tracing integration approach rather than precomputed fields and relies on specific assumptions about low albedo.

## Operating characteristics
*   **Time Complexity:** Depends on the number of ray marching steps and the complexity of the illumination integration (e.g., number of light samples). For simple lights, it is $O(N)$ where $N$ is the number of samples per ray [S1].
*   **Memory Complexity:** $O(V)$ where $V$ is the volume of the density field, to store optical properties [S2].
*   **Performance:** Significantly faster than full multiple scattering models (e.g., path tracing) because it avoids recursive light transport calculations [S2].
*   **Strengths:** Physically plausible for low-density media; efficient; differentiable (useful for reconstruction) [S1, S2].
*   **Limitations:** Inaccurate for dense or highly scattering media; ignores emission; assumes pure volume scenes [S1, S2][S3, unofficial].
*   **When to use:** Smoke, fog, or clouds with low optical depth; differentiable rendering for volume reconstruction [S1, S2].
*   **When not to use:** Dense fog, fire, or scenes with significant multiple scattering or surface interactions [S2][S3, unofficial].

## Background
The Single Scattering Model was introduced by Kajiya and Von Herzen in 1984 as a two-pass ray tracing method for volume rendering [S3, unofficial]. It was later applied to medical images by Meinzer et al. as the "Heidelberg ray tracing model" [S3, unofficial]. The model is based on the Radiative Transfer Equation (RTE) but simplifies it by considering only the first-order scattering term [S1, S2]. It has been widely used in differentiable rendering for fluid and smoke reconstruction due to its computational efficiency and differentiability [S1, S2].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
[S2] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
[S3] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
