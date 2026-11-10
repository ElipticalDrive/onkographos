# Volume Shader

Also called: Participating Media Material, Volume Material

## Overview
A volume shader is a material definition for participating media that occupies space, instructing the render engine how light interacts with the medium at every point within a volume [S1]. Unlike surface shaders that define boundary reflections, it governs the internal behavior of light as it traverses the medium. It is distinguished from its siblings by its role as the foundational material interface that maps artistic parameters (Scattering, Absorption, Emission) to physical interactions, serving as the basis for implementations like the [Standard Volume Shader Implementation](10712-standard-volume-shader-implementation.md) and the [Volumetric Scattering Model](11410-volumetric-scattering-model.md).

## How it works
**Inputs and Outputs**
*   **Input:** Ray position $\mathbf{p}$, ray direction $\boldsymbol{\omega}$, and spatial data (density, coefficients) at $\mathbf{p}$.
*   **Output:** Outgoing radiance $L_o(\mathbf{p}, \boldsymbol{\omega})$ contributed by the medium at that point.

**Data Structures**
The shader defines three primary coefficients per spatial point:
1.  **Scattering Coefficient ($\sigma_s$):** Often represented artistically as "Scatter Color," defining the color and intensity of light bounced off microscopic particles [S1].
2.  **Absorption Coefficient ($\sigma_a$):** The counterpart to transparency, defining how much light is absorbed (converted to heat or other energy) by the medium [S1].
3.  **Emission Coefficient ($\sigma_e$ or $L_e$):** The counterpart to incandescence, defining the light emitted by the medium itself [S1].

**Formulas**
The total extinction coefficient $\sigma_t$ combines absorption and scattering [S1, S5]:
$$ \sigma_t = \sigma_a + \sigma_s $$

The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss (extinction) and gain (emission and in-scattering) terms [G]. In the context of the volume shader's core definition, the interaction is characterized by:
*   **Absorption:** $dL_{abs} = -\sigma_a L_i dt$ [S1]
*   **Emission:** $dL_{emit} = \sigma_e dt$ [S1]
*   **Scattering:** Redirects incoming radiance $L_i$ into the view direction $\boldsymbol{\omega}$ based on the phase function and $\sigma_s$ [S1].

**Parameters**
*   **Scattering:** Defines the color and amount of light bounced off particles (e.g., white for clouds, grey for smoke) [S1].
*   **Absorption:** Defines the darkness or opacity of the volume (e.g., thick smoke appears dark) [S1].
*   **Emission:** Defines self-illumination (e.g., fire, explosions) [S1].

**Pseudocode**
The following pseudocode illustrates the logical structure of a volume shader's contribution calculation, assuming access to the medium's properties at the current ray position.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_volume_shader_contribution(ray_position, ray_direction, medium_properties):
    """
    Calculates the radiance contribution of a volume shader at a specific point.
    
    Args:
        ray_position (Vector3): Current position of the ray in the medium.
        ray_direction (Vector3): Direction of the ray.
        medium_properties (Dict): Contains scattering, absorption, and emission coefficients.
        
    Returns:
        Vector3: The radiance contribution from the volume at this point.
    """
    # Retrieve coefficients from the material definition
    sigma_s = medium_properties['scattering_coefficient']
    sigma_a = medium_properties['absorption_coefficient']
    sigma_e = medium_properties['emission_coefficient']
    
    # Calculate total extinction
    sigma_t = sigma_a + sigma_s
    
    # Calculate transmittance for a small step dt (conceptual, actual rendering integrates this)
    # T = exp(-sigma_t * dt)
    
    # Calculate emission contribution
    # Emission adds radiance directly to the view ray
    emission_contribution = sigma_e
    
    # Calculate scattering contribution
    # This requires integrating incoming radiance from all directions, weighted by the phase function
    # For a single point evaluation without full integration, this is often approximated or deferred
    # to the ray marching loop. Here we define the term.
    scattering_contribution = sigma_s * incoming_radiance_from_all_directions
    
    # Total contribution is the sum of emission and scattering
    # Note: Absorption reduces the incoming radiance, handled by the transmittance factor
    # in the integration step.
    total_contribution = emission_contribution + scattering_contribution
    
    return total_contribution
```

**Edge Cases and Pitfalls**
*   **Zero Density:** If the medium density is zero, all coefficients should effectively be zero, resulting in no interaction [S1].
*   **High Absorption:** High $\sigma_a$ can lead to numerical instability if not handled with proper clamping or log-space calculations during transmittance evaluation [G].
*   **Emission Dominance:** If emission is very high, it may saturate the output if not tone-mapped correctly [G].

## Subtopics

### [Standard Volume Shader Implementation](10712-standard-volume-shader-implementation.md)
A physically based implementation for clouds, smoke, fire, and explosions, mapping artist-friendly parameters to physical coefficients using a mean-free-path encoding.

### [Volumetric Scattering Model](11410-volumetric-scattering-model.md)
Describes the general phenomenon of light scattering in participating media like clouds, smoke, fire, and translucent materials, where the medium affects radiance along the ray.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Provides a unified framework for defining complex volumetric behaviors (clouds, smoke, fire) through three intuitive parameters [S1].
*   **Limitations:** Requires gathering data at many positions within the volume for realistic rendering, which can be computationally expensive [S1].

## Background
The concept of the volume shader as a material for participating media is rooted in the physics of light transport in heterogeneous media. The term "participating media" is used because the space itself participates in the lighting [S1]. Modern render engines, such as Redshift, implement this through shaders like the Standard Volume shader, which is built on the same conceptual framework as Random Walk Subsurface Scattering but adapted for spatially varying density [S1].

## Sources
[S1] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
