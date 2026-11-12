# Basic Gas Rendering Algorithm
Also called: Low-albedo gas illumination model, CFD gas visualization algorithm

## Overview
The Basic Gas Rendering Algorithm is a volume rendering method designed for visualizing Computational Fluid Dynamics (CFD) data, specifically gaseous flows and atmospheric phenomena [S1, unofficial]. It solves the problem of revealing internal flow details, shock waves, and pressure changes that are often obscured by traditional surface-based rendering techniques [S1, unofficial]. Unlike methods that rely on gradient-based surface implication and Phong illumination, this approach uses self-shadowing and volume transparency to provide a deeper understanding of three-dimensional flow structures [S1, unofficial].

## How it works
**Inputs:**
- Ray origin and direction.
- Volume density function (interpolated from a CFD grid) [S1, unofficial].
- Solid shadow table for self-shadowing queries [S1, unofficial].
- Parameters: `density_scalar` (multiplier), `power_function` exponent (contrast enhancer), `transparency_threshold` (default 0.01) [S1, unofficial].

**Outputs:**
- Final accumulated color (`final_color`) for the pixel.
- Accumulated density (`sum_density`) [S1, unofficial].

**Data Structures:**
- **CFD Grid:** Stores raw density values. Interpolation (e.g., trilinear) is used to evaluate density at arbitrary points [S1, unofficial].
- **Solid Shadow Table:** A precomputed or runtime-accessible structure storing shadowing information for volume elements [S1, unofficial].
- **A-buffer:** Used to store fragment information for the final image [S1, unofficial].

**Formulas:**
1.  **Opacity (Optical Depth):** The opacity $\tau$ is calculated as the integral of density $\rho(s)$ along the ray path from start $s_i$ to end $s_e$ [S1, unofficial]:
    $$ \tau = \int_{s_i}^{s_e} \rho(s) ds $$
    In practice, this is approximated by summing density values multiplied by the step size [S1, unofficial].

2.  **Density Modification:** Raw density $\rho_{raw}$ is modified using a scalar and a power function to enhance visualization [S1, unofficial]:
    $$ \rho_{modified} = (\rho_{raw} \times \text{density\_scalar})^{\text{power\_function}} $$
    *(Note: The exact order of operations for scalar and power is not explicitly defined in the source, but both are applied to enhance volumetric information and contrast [S1, unofficial].)*

3.  **Illumination Model:** A low-albedo illumination model is used. The phase function $p(\theta)$ is based on the summation of Henyey-Greenstein functions [S1, unofficial]. The amount of light $I$ reflected from an element depends on the angle between the light and the eye [S1, unofficial].

**Numbered Steps:**
1.  Initialize `final_color` to 0 and `sum_density` to 0.
2.  Iterate through each section of the gas volume along the ray.
3.  For each increment (sample point) along the ray:
    a.  Evaluate the modified density $\rho_{modified}$ at the current sample point using the CFD grid and parameters [S1, unofficial].
    b.  Calculate the opacity contribution for this step. If this is the final increment, scale the opacity proportionally to account for the reduced step size [S1, unofficial].
    c.  If self-shadowing is enabled, retrieve the shadowing factor for the current element from the solid shadow table [S1, unofficial].
    d.  Calculate the gas illumination color using the opacity, modified density, shadowing factor, and the low-albedo illumination model (Henyey-Greenstein phase function) [S1, unofficial].
    e.  Add the calculated illumination color to `final_color` [S1, unofficial].
    f.  Add the modified density to `sum_density` [S1, unofficial].
    g.  Calculate the transparency of the accumulated path. If transparency is less than 0.01, stop tracing the ray [S1, unofficial].
    h.  Increment the sample point. Monte Carlo sampling may be used to choose sample points to reduce aliasing artifacts [S1, unofficial].
    i.  Create the a-buffer fragment for the current sample [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def basic_gas_rendering(ray_origin, ray_direction, cfd_grid, shadow_table, params):
    final_color = [0.0, 0.0, 0.0]
    sum_density = 0.0
    sample_point = ray_origin
    step_size = params.get('step_size', 1.0) # Not given in sources, assumed constant or adaptive
    transparency_threshold = 0.01
    density_scalar = params.get('density_scalar', 1.0)
    power_function = params.get('power_function', 1.0)
    self_shadowing_enabled = params.get('self_shadowing', True)
    
    # Monte Carlo sampling offset to reduce aliasing [S1, unofficial]
    # Implementation detail: jitter sample_point within step_size
    
    while True:
        # Get raw density from CFD grid
        raw_density = cfd_grid.interpolate(sample_point)
        
        # Apply density modifications [S1, unofficial]
        modified_density = (raw_density * density_scalar) ** power_function
        
        # Calculate opacity for this step
        # Opacity is density * step_size [S1, unofficial]
        # Scale for final increment if needed [S1, unofficial]
        current_opacity = modified_density * step_size
        
        # Retrieve shadowing if enabled [S1, unofficial]
        shadow_factor = 1.0
        if self_shadowing_enabled:
            shadow_factor = shadow_table.retrieve_shadow(sample_point)
            
        # Calculate illumination using low-albedo model [S1, unofficial]
        # Phase function based on Henyey-Greenstein summation [S1, unofficial]
        illumination_color = calculate_gas_illumination(
            opacity=current_opacity,
            density=modified_density,
            shadow_factor=shadow_factor,
            light_direction=..., # Not given in sources
            view_direction=ray_direction
        )
        
        # Accumulate color [S1, unofficial]
        final_color += illumination_color
        
        # Accumulate density [S1, unofficial]
        sum_density += modified_density
        
        # Check transparency threshold [S1, unofficial]
        # Transparency is related to accumulated opacity
        # If transparency < 0.01, stop
        accumulated_opacity = ... # Not given in sources, typically 1 - exp(-sum_density)
        transparency = 1.0 - accumulated_opacity
        if transparency < transparency_threshold:
            break
            
        # Increment sample point [S1, unofficial]
        sample_point += ray_direction * step_size
        
        # Create a-buffer fragment [S1, unofficial]
        create_a_buffer_fragment(sample_point, final_color, sum_density)
        
        # Safety break to prevent infinite loops
        if sample_point.distance(ray_origin) > max_ray_length:
            break
            
    return final_color, sum_density
```

**Edge Cases and Pitfalls:**
- The final increment along the ray may be smaller than standard steps; its opacity must be scaled proportionally to account for the reduced step size [S1, unofficial].
- Precision issues may arise when transparency is very close to the threshold (0.01) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples along the ray. The use of a shadow table allows for fast shadow queries, but the table itself may require significant precomputation or memory [S1, unofficial].
- **Memory Complexity:** Depends on the size of the CFD grid and the solid shadow table [S1, unofficial].
- **Strengths:** Reveals internal flow details, shock waves, and pressure changes that are lost in traditional surface-based rendering. Self-shadowing increases the understandability of the 3D flow [S1, unofficial].
- **Limitations:** Traditional techniques using gradient-based surface implication and Phong illumination are simpler but lack the depth of information provided by this method [S1, unofficial]. The method may be computationally expensive due to the need for shadow table lookups and multiple density evaluations [S1, unofficial].
- **When to use:** When visualizing CFD data where internal structures, flow dynamics, and pressure variations are critical, and when traditional surface rendering fails to convey this information [S1, unofficial].

## Background
The Basic Gas Rendering Algorithm was developed for the visualization of Computational Fluid Dynamics (CFD) data [S1, unofficial]. It was introduced as an alternative to traditional volumetric rendering techniques that rely on gradient-based surface implication and Phong illumination [S1, unofficial]. The method emphasizes the use of self-shadowing and volume transparency to enhance the visualization of gaseous flows and atmospheric phenomena [S1, unofficial].

## Sources
[S1] PDF Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
