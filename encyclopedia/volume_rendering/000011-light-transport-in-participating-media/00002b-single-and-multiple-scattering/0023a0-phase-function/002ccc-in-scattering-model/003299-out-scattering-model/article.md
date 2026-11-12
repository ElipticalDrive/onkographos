# Out-Scattering Model

Also called: Out Scattering, Scattering Loss, real scattering

## Overview
Out-scattering is the phenomenon where photons are scattered away from their incident direction, causing a reduction in radiance along a ray as light travels through a participating medium [S1, unofficial][S3, book]. It represents the loss of light from the current viewing direction due to collisions with particles that deflect energy into other directions [S3, book]. Unlike in-scattering, which adds radiance, out-scattering acts as an attenuation term in the radiative transfer equation, often grouped with true absorption under the total extinction coefficient [S2, unofficial][S3, book].

## How it works
**Inputs and Outputs**
*   **Input**: Position $p$, direction $\omega$, incident radiance $L_i(p, -\omega)$, scattering coefficient $\sigma_s(p, \omega)$.
*   **Output**: Differential change in radiance $dL_o(p, \omega)$ due to out-scattering.

**Data Structures**
No specific data structures are required for the core definition. The method relies on scalar or vector fields for the scattering coefficient $\sigma_s$.

**Parameters**
*   $\sigma_s(p, \omega)$: Scattering coefficient (or scattering cross-section). Defined as the probability of an out-scattering event occurring per unit distance [S2, unofficial][S3, book]. It may vary with position $p$ and direction $\omega$ [S3, book].
*   $dt$: Differential length along the ray [S3, book].
*   $\chi$: Total extinction coefficient, defined as the sum of true absorption $\kappa$ and out-scattering $\sigma_s$ ($\chi = \kappa + \sigma_s$) [S2, unofficial].
*   $W$: Albedo, a measure of reflectivity or the ability to scatter, defined as $W = \frac{\sigma_s}{\chi} = \frac{\sigma_s}{\kappa + \sigma_s}$ [S2, unofficial].

**Formulas**
The differential reduction in radiance due to out-scattering is given by [S3, book]:

$$
dL_o(p, \omega) = -\sigma_s(p, \omega) L_i(p, -\omega) dt
$$

Where:
*   $L_i(p, -\omega)$ is the incident radiance arriving at point $p$ from direction $-\omega$ (the direction opposite to the ray's propagation $\omega$).
*   The negative sign indicates a loss of radiance [S3, book].

The total reduction in radiance along a path segment is determined by integrating this differential loss, often combined with absorption in the transmittance calculation [S1, unofficial][S3, book]. The transmittance $T$ over a distance $d$ considering both absorption and out-scattering (extinction) is typically calculated using the Beer-Lambert law with the total extinction coefficient $\chi$ [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_out_scattering_loss(position, direction, incident_radiance, scattering_coefficient, delta_length):
    """
    Computes the differential radiance loss due to out-scattering.
    
    Args:
        position: Current position p in the medium.
        direction: Ray propagation direction omega.
        incident_radiance: Radiance L_i(p, -omega) arriving from the opposite direction.
        scattering_coefficient: Scattering coefficient sigma_s(p, omega).
        delta_length: Differential step size dt.
        
    Returns:
        dL_o: The differential reduction in radiance.
    """
    # Formula: dL_o = -sigma_s * L_i * dt
    # Note: incident_radiance represents the light traveling in direction -omega
    # that is being scattered out of the direction omega.
    
    loss = -scattering_coefficient * incident_radiance * delta_length
    return loss
```

**Edge Cases and Pitfalls**
*   **Precision**: If $\sigma_s$ is zero, there is no out-scattering loss [S3, book].
*   **Boundary**: The coefficient $\sigma_s$ must be defined within the medium boundaries; outside the medium, $\sigma_s$ is typically zero [S3, book].
*   **Distinction from Null Scattering**: Out-scattering is part of "real scattering," distinct from "null scattering" techniques used for acceleration where scattering events are virtually ignored or handled differently [S3, book].

## Subtopics

### In-scattering Model
*   **What it is**: The complementary process where light is scattered *into* the current ray's direction from other directions.
*   **Difference**: While out-scattering removes radiance from the current path, in-scattering adds radiance to it.
*   **When to choose**: Must be computed alongside out-scattering for accurate volume rendering, as they are coupled in the radiative transfer equation.
*   **Link**: [In-scattering Model](5367-in-scattering-model.md)

## Operating characteristics
*   **Time Complexity**: The computation of the differential loss is $O(1)$ per step. However, it is part of the integral solution to the radiative transfer equation, which generally requires integration over the path length.
*   **Memory Complexity**: $O(1)$ for the coefficient lookup if precomputed, or dependent on the volume data structure size.
*   **Strengths**: Physically accurate representation of light loss in participating media [S3, book].
*   **Limitations**: Requires accurate knowledge of the scattering coefficient $\sigma_s$, which can be difficult to measure or estimate [S3, book].
*   **When to use**: When rendering participating media where light attenuation due to scattering is significant [S1, unofficial][S3, book].
*   **When not to use**: For vacuum or non-participating media where $\sigma_s = 0$ [S3, book].

## Background
Out-scattering is a fundamental concept in the physics of light transport in participating media, described in standard computer graphics literature such as the Physically Based Rendering (PBRT) book [S3, book]. It is often introduced alongside absorption and in-scattering to explain the full radiative transfer equation [S1, unofficial][S3, book]. The term "real scattering" is used to distinguish it from null scattering approximations [S3, book].

## Sources
[S1] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S3] 11.1 Volume Scattering Processes, https://pbr-book.org/4ed/Volume_Scattering/Volume_Scattering_Processes, fetched 2026-10-29 (book)
