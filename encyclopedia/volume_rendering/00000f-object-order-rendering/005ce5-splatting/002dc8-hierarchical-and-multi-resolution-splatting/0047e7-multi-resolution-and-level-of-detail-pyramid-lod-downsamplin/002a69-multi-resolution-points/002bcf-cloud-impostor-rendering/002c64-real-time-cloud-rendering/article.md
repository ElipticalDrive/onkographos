# Real-Time Cloud Rendering

Also called: Harris and Lastra, Harris and Lastra [30, 31], Harris and Lastra cloud rendering, Precomputed Scattering Cloud Impostors, Two-pass Cloud Rendering

## Overview
Real-Time Cloud Rendering is a two-pass algorithm for the realistic, real-time rendering of static, constant-shape clouds suitable for flight simulation and games [S2, unofficial]. It solves the performance bottleneck of particle-based cloud rendering by approximating multiple forward scattering in a preprocess and first-order anisotropic scattering at runtime, using dynamically generated 2D impostors to accelerate rendering [S2, unofficial]. It sets itself apart from its sibling [Cloud Impostor Rendering](5125-cloud-impostor-rendering.md) by explicitly separating the illumination calculation into a precomputed multiple forward scattering term and a runtime anisotropic single scattering term, rather than relying solely on isotropic approximations or simple texture mapping [S1].

## How it works
**Inputs:** A set of static cloud particles, where each particle is defined by a center position, radius, density, and color [S2, unofficial]. The particles are assumed to be static relative to each other [S2, unofficial]. A light direction vector.

**Outputs:** A rendered image of the clouds using screen-oriented splats [S2, unofficial].

**Data Structures:**
*   **Particle:** A structure containing `center` (3D vector), `radius` (scalar), `density` (scalar), and `color` (RGB vector) [S2, unofficial].
*   **Impostor:** A 2D texture map generated per cloud (or per particle group) that stores the precomputed shading information. The exact layout is not specified, but it is texture-mapped onto screen-oriented polygons [S1][S2, unofficial].

**Steps:**

1.  **Preprocess (Pass 1):** Compute the multiple forward scattering illumination for each particle in the light direction [S2, unofficial].
    *   This step assumes the clouds are static and of constant shape, allowing shading to be computed once per scene [S2, unofficial].
    *   Multiple forward scattering is approximated by considering only the most significant direction from the whole sphere of directions: the light direction [S1].
    *   This approximation is a harder restriction of the space of directions than methods like Nishita et al. [S1].
    *   The result is stored in the impostor texture [S2, unofficial].

2.  **Runtime (Pass 2):** Render the clouds using the precomputed shading and single scattering toward the viewer [S2, unofficial].
    *   Calculate anisotropic first-order scattering at runtime [S1].
    *   The scattering phase function used is the Rayleigh phase function [S1]. Note: This is noted as far from well-suited for clouds; more accurate phase functions should be used for higher accuracy [S1].
    *   Render particles using splatting [S1][S2, unofficial].
    *   Use dynamically generated impostors to achieve real-time rendering in scenes with many complex clouds [S1][S2, unofficial].
    *   The impostors are screen-oriented polygons texture-mapped with the Gaussian density function and the precomputed shading [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocess Step
def preprocess_clouds(clouds, light_direction):
    """
    Compute multiple forward scattering for each cloud.
    clouds: List of Cloud objects.
    light_direction: Normalized 3D vector.
    """
    for cloud in clouds:
        # Approximate multiple forward scattering by considering only the light direction
        # The exact formula for the precomputed shading is not given in the sources [G].
        # It likely involves integrating the particle density along the light direction
        # and applying an albedo/scattering model.
        precomputed_shading = compute_multiple_forward_scattering(cloud, light_direction)
        
        # Generate impostor texture from precomputed shading
        # The exact generation process (e.g., projection, resolution) is not specified [G].
        cloud.impostor_texture = generate_impostor_texture(precomputed_shading)

# Runtime Step
def render_clouds(clouds, view_matrix, projection_matrix, light_direction):
    """
    Render clouds using impostors and runtime single scattering.
    """
    for cloud in clouds:
        # Generate screen-oriented polygon (billboard) for the impostor
        # The exact method for generating the impostor geometry is not specified [G].
        # It is assumed to be a screen-aligned quad or similar primitive.
        impostor_geometry = generate_screen_oriented_impostor(cloud, view_matrix)
        
        # Bind the precomputed impostor texture
        bind_texture(cloud.impostor_texture)
        
        # Draw the impostor with runtime anisotropic single scattering
        # The exact shader implementation is not given [G].
        # It combines the precomputed multiple forward scattering with
        # anisotropic single scattering using the Rayleigh phase function.
        draw_impostor(impostor_geometry, light_direction)
```

**Parameters:**
*   **Phase Function:** Rayleigh phase function [S1].
*   **Particle Model:** Gaussian density falloff from the center [S2, unofficial].
*   **Scattering Approximation:** Multiple forward scattering (light direction) + Anisotropic first-order scattering (eye direction) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Phase Function Accuracy:** The Rayleigh phase function is noted as far from well-suited for clouds [S1]. Using it may result in less accurate visual results compared to more complex phase functions [S1].
*   **Static Assumption:** The method assumes particles are static relative to each other [S2, unofficial]. Dynamic clouds are not supported without re-preprocessing [S2, unofficial].
*   **Directional Approximation:** Approximating multiple scattering only in the light direction is a significant simplification [S1]. This may lead to inaccuracies in complex lighting scenarios or when the light direction is not the dominant source of illumination [S1].

## Operating characteristics
*   **Time Complexity:** Preprocessing is done once per cloud [S1]. Runtime rendering is accelerated by using impostors, allowing hundreds of clouds and hundreds of thousands of particles to be rendered at high frame rates [S2, unofficial].
*   **Memory Complexity:** Requires storing impostor textures for each cloud [S1][S2, unofficial].
*   **Strengths:** Real-time performance for complex scenes; allows viewer interaction (flying in and around clouds); reduces artifacts from direct particle rendering [S2, unofficial].
*   **Limitations:** Static clouds only; Rayleigh phase function is inaccurate for clouds; multiple scattering approximation is limited to the light direction [S1][S2, unofficial].
*   **When to use:** Flight simulation and games requiring real-time rendering of static clouds [S1][S2, unofficial].
*   **When not to use:** Dynamic clouds; scenes requiring high-fidelity scattering physics [S1].

## Background
The method was presented by Harris and Lastra [S2, unofficial]. It is based on a two-pass method by Dobashi et al. [S1][S2, unofficial]. The work is concerned with the real-time rendering of high-quality static constant-shape clouds suitable for flight simulation and games [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Real-Time Cloud Rendering, https://www.cs.unc.edu/techreports/01-005.pdf, fetched 2026-10-28 (unofficial)
