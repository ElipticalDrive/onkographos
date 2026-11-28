# Single Scattering Model

Also called: Kajiya and Von Herzen model, RTE with single scattering, Single Scattering Radiance Transport, Single-Scattering Participating Media Rendering, Two-Pass Single Scattering

## Overview
The Single Scattering Model is a light transport approximation for participating media that accounts for exactly one secondary scattering event between the observer and a light source, ignoring multiple scattering and emitted radiance [S1, S2][S3]. It solves the problem of rendering volumetric effects like smoke or clouds by assuming low albedo or low density, where single scattering events dominate light transport [S2][S4, unofficial]. It sets itself apart from its siblings by its rigorous inclusion of volume attenuation and phase function integration in a two-pass framework, whereas other approximations may rely on gradient guidance or simplified shading rules [S4, unofficial].

## How it works
The model computes the outgoing radiance $L_{out}$ at a viewpoint $x$ in direction $\boldsymbol{\omega}$ by decomposing it into reduced incident radiance $L_{ri}$ (light transmitted directly through the volume) and single-scattering radiance $L_{ss}$ (light scattered once into the view ray) [S1][S3].

### Inputs and Outputs
*   **Inputs:**
    *   Viewpoint position $x$ [S3].
    *   View direction $\boldsymbol{\omega}$ (unit vector) [S3].
    *   Density field $\rho(\mathbf{p})$ or optical property fields: absorption coefficient $\kappa_a(\mathbf{p})$, scattering coefficient $\kappa_s(\mathbf{p})$, and extinction coefficient $\kappa_t(\mathbf{p}) = \kappa_a(\mathbf{p}) + \kappa_s(\mathbf{p})$ [S3].
    *   Scattering albedo $\Omega(\mathbf{p}) = \kappa_s(\mathbf{p}) / \kappa_t(\mathbf{p})$ [S3].
    *   Phase function $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ describing the directional distribution of scattering [S3].
    *   Incoming ambient radiance $L_{in}(\boldsymbol{\omega}')$ from all directions $\boldsymbol{\omega}'$ [S3].
    *   Volume boundary points $x_a$ (near entry) and $x_b$ (far exit) for the ray passing through $x$ [S3].
*   **Outputs:**
    *   Outgoing radiance $L_{out}(x, \boldsymbol{\omega})$ [S3].

### Data Structures
*   **Ray:** Defined by origin $x$ and direction $\boldsymbol{\omega}$.
*   **Volume Representation:** A grid or voxel structure storing $\kappa_t$, $\Omega$, and $p$ parameters at each position $\mathbf{p}$ [S3].
*   **Transmittance Cache:** Precomputed or on-the-fly integration of $\kappa_t$ along the ray path to determine optical depth [S2].

### Formulas
The total outgoing radiance is the sum of direct transmission and single scattering [S1][S3]:
$$ L_{out}(x, \boldsymbol{\omega}) = L_{ri}(x, \boldsymbol{\omega}) + L_{ss}(x, \boldsymbol{\omega}) \quad (1) $$

**1. Reduced Incident Radiance ($L_{ri}$)**
This is the ambient light $L_{in}(\boldsymbol{\omega})$ attenuated by the transmittance from the volume boundary $x_a$ to the viewpoint $x$ [S1][S3]:
$$ L_{ri}(x, \boldsymbol{\omega}) = L_{in}(\boldsymbol{\omega}) \tau(x_a, x) \quad (2) $$

**2. Transmittance ($\tau$)**
The fraction of light surviving transmission between two points $x_0$ and $x_1$ is defined by the Beer-Lambert law [S1][S3]:
$$ \tau(x_0, x_1) = e^{-\int_{D_1}^{D_0} \kappa_t(x_u) du} \quad (3) $$
where $D_0$ and $D_1$ are the distances from the viewpoint $x$ to $x_0$ and $x_1$ respectively, and $x_u$ is a point along the ray path [S3]. For $\tau(x_a, x)$, the integral runs from $D_a$ (distance to $x_a$) to $0$ (distance to $x$), yielding $\tau(x_a, x) = e^{-\int_{0}^{D_a} \kappa_t(x_u) du}$ [S3].

**3. Single-Scattering Radiance ($L_{ss}$)**
This term integrates the light scattered from all incoming directions $\boldsymbol{\omega}'$ into the view direction $\boldsymbol{\omega}$ at every point $x_u$ along the ray [S1][S3]:
$$ L_{ss}(x, \boldsymbol{\omega}) = \int_{D_a}^{D_b} \tau(x_u, x) \kappa_s(x_u) J_{ss}(x_u, \boldsymbol{\omega}) du \quad (4) $$
Where:
*   $x_u$ is an arbitrary position inside the volume at distance $D_u$ from $x$ [S3].
*   $\tau(x_u, x)$ is the transmittance from the scattering point $x_u$ to the viewer $x$ [S3].
*   $\tau(x_b, x_u)$ is the transmittance from the volume entry point $x_b$ (where light enters from the environment) to the scattering point $x_u$ [S3]. The distance from viewer to $x_b$ is $D_b$, to $x_u$ is $D_u$ [S3].
*   $J_{ss}(v_p, \boldsymbol{\omega}) = \frac{\Omega}{4\pi} \int_{S^2} L_{in}(\boldsymbol{\omega}') p(\boldsymbol{\omega}, \boldsymbol{\omega}') d\boldsymbol{\omega}'$ represents the reduced incident radiance at voxel $v_p$, integrating incident radiance weighted by the phase function over the unit sphere [S1][S3].

**4. Two-Pass Implementation (Kajiya and Von Herzen)**
For efficient computation, especially with complex lighting, the model can be implemented in two passes [S6, unofficial]:
*   **Pass 1 (Illumination):** Compute the illumination $I(X)$ reaching each point $X$ in the volume from the light sources. This involves propagating flux from the light source through the volume, accounting for the transparency of each layer before propagating to the next [S6, unofficial].
*   **Pass 2 (Scattering to View):** Gather the scattered intensity along viewing rays. The shading rule $g(X)$ determines the intensity scattered from point $X$ towards the viewpoint [S6, unofficial]:
    $$ g(X) = r(X, \boldsymbol{\omega}, \boldsymbol{\omega}') I(X) $$
    Where the bidirectional reflection distribution function (BRDF) for scattering is:
    $$ r(X, \boldsymbol{\omega}, \boldsymbol{\omega}') = a \tau(X) p(\boldsymbol{\omega}, \boldsymbol{\omega}') \quad (5) $$
    Here $a$ is the albedo (fraction of extinction representing scattering, i.e., $\Omega$) and $\tau(X)$ denotes the extinction coefficient in this notation [S6, unofficial]. The final radiance is the integral of $g(X)$ along the viewing ray, weighted by transmittance to the viewer [S6, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def single_scattering_render(viewpoint, ray_direction, volume_grid, light_source):
    """
    Renders a single ray using the Single Scattering Model.
    
    Args:
        viewpoint: 3D position of the camera.
        ray_direction: Unit vector of the view ray.
        volume_grid: Data structure containing kappa_t, kappa_s, and phase function p.
        light_source: Object providing incoming radiance L_in(omega') for all directions.
        
    Returns:
        L_out: Float, the outgoing radiance at the viewpoint.
    """
    # 1. Determine ray bounds within the volume
    xa, xb = volume_grid.intersect_ray(viewpoint, ray_direction)
    if xa is None or xb is None:
        return 0.0
    
    Da = math.dist(viewpoint, xa)
    Db = math.dist(viewpoint, xb)
    
    # 2. Compute Reduced Incident Radiance (Direct Transmission)
    tau_view = compute_transmittance(viewpoint, xa, volume_grid)
    L_in_direct = light_source.get_radiance(ray_direction)
    L_ri = L_in_direct * tau_view
    
    # 3. Compute Single-Scattering Radiance
    L_ss = 0.0
    num_steps = 100
    step_size = (Db - Da) / num_steps
    
    for i in range(num_steps):
        Du = Da + i * step_size
        xu = [viewpoint[j] + ray_direction[j] * Du for j in range(3)]
        
        if Du > Da + step_size: break
        
        kappa_s = volume_grid.get_kappa_s(xu)
        if kappa_s == 0:
            continue
            
        tau_xu_view = compute_transmittance(viewpoint, xu, volume_grid)
        tau_env_xu = compute_transmittance(xb, xu, volume_grid)
        
        J_ss = compute_incident_scattering_term(xu, ray_direction, light_source, volume_grid.get_phase_function)
        
        contribution = tau_xu_view * kappa_s * (J_ss / (4 * math.pi)) * step_size
        L_ss += contribution

    return L_ri + L_ss

def compute_transmittance(p1, p2, volume_grid):
    dist = math.dist(p1, p2)
    if dist == 0: return 1.0
    
    samples = 10
    integral_kappa_t = 0.0
    step = dist / samples
    
    for i in range(samples):
        t = i * step
        pos = [p1[j] + (p2[j] - p1[j]) * (t / dist) for j in range(3)]
        kappa_t = volume_grid.get_kappa_t(pos)
        integral_kappa_t += kappa_t * step
        
    return math.exp(-integral_kappa_t)

def compute_incident_scattering_term(pos, view_dir, light_source, phase_func):
    # For point/parallel lights, calculate directly without Monte Carlo [S1]
    # For environment maps, use importance sampling [S1]
    # Here we implement a direct evaluation for a single dominant light direction
    light_dir = light_source.get_direction(pos)
    L_in = light_source.get_intensity(pos)
    p_val = phase_func(view_dir, light_dir)
    return L_in * p_val
```

### Parameters
*   $\kappa_t$: Extinction coefficient ($\kappa_a + \kappa_s$) [S3].
*   $\Omega$: Scattering albedo ($\kappa_s / \kappa_t$) [S3].
*   $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$: Phase function. For water droplets, peaks in the forward scattering direction [S6, unofficial].
*   $a$: Albedo, fraction of extinction representing scattering [S6, unofficial].

### Edge Cases and Pitfalls
*   **Validity:** The model is only valid if the albedo or density is low, such that multiple scattering is unlikely [S6, unofficial].
*   **Precision:** Numerical integration of transmittance and phase functions requires sufficient sampling to avoid artifacts, especially in heterogeneous media [S3].
*   **Boundaries:** Correctly identifying volume entry and exit points ($x_a, x_b$) is critical for accurate transmittance calculation [S3].

## Subtopics

### [Single Scattering Propagator](1299-single-scattering-propagator.md)
Models light transport with zero or exactly one scattering event, separating direct and singly-scattered light. Distinguished by rigorous attenuation and phase integration compared to the Utah Approximation.

### [Single Scattering (Blinn Model)](4581-single-scattering-blinn-model.md)
Applies Blinn's 1982 approximations for uniform, low-density media, building on the general single scattering model.

## Operating characteristics
*   **Angular Integration:** For simple point or parallel lights, $J_{ss}$ is calculated directly without Monte Carlo integration [S1]. For environment map lighting, importance sampling estimates $J_{ss}$ efficiently [S1].
*   **Strengths:** Computationally efficient compared to full multiple scattering models; physically plausible for low-density media like smoke or thin clouds [S2].
*   **Limitations:** Fails for high-density or high-albedo media where multiple scattering dominates [S6, unofficial]. Ignores emitted radiance [S1].
*   **When to use:** Smoke reconstruction, thin clouds, or differentiable rendering of fluids where speed is prioritized and multiple scattering is negligible [S1, S2].

## Background
The Single Scattering Model is derived from the Radiative Transfer Equation (RTE) [S1]. The two-pass method for computing single scattering was introduced by Kajiya and Von Herzen [S6, unofficial]. It has been applied in various contexts, including the "Heidelberg ray tracing model" for medical images [S6, unofficial] and recent differentiable rendering frameworks for fluid and smoke reconstruction [S1, S2][S3].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
[S2] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
[S3] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.9450546, fetched 2026-10-27
[S4] A Tile-Based Approach for Photo-Realistic Volume Rendering, http://hdl.handle.net/1794/23836, fetched 2026-10-26 (unofficial)
[S5] Physics-Based Differentiable Rendering for Efficient and ... - MDPI, https://www.mdpi.com/1099-4300/25/9/1348, fetched 2026-10-27 (unofficial)
[S6] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
