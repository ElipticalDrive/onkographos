# Volumetric Lighting with Phase Functions

Also called: Secondary Raymarching for Volumometric Lighting, Volumetric Lighting and Phase Functions

## Overview
This method computes the illumination of participating media (such as clouds or fog) by estimating how much light from a distant source (e.g., the sun) reaches a specific point in the volume and how that light is scattered toward the camera. It solves the problem of realistic light shafts, edge highlights, and "silver linings" in single-scattering approximations. It differs from [Basic Volume Rendering and Ray Marching Loop with Absorption & Scattering](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md) by explicitly integrating the light transport path from the volume sample to the light source, rather than relying on pre-computed ambient or direct lighting terms.

## How it works
**Inputs:**
- 3D density field (e.g., Perlin-Worley noise texture) [S1, unofficial].
- Camera position and view direction.
- Light source direction (assumed distant, e.g., the sun).
- Asymmetry parameter $g$ for the phase function.

**Outputs:**
- Illuminated color contribution for the current volume sample.

**Data Structures:**
- Density sample: A scalar value representing optical depth or particle density at a 3D coordinate. If the density is zero, no lighting calculation is performed [S1, unofficial].

**Steps:**
1. **Density Check:** Sample the primary 3D density field at the current view ray intersection point. If the density is zero, the contribution is zero [S1, unofficial].
2. **Secondary Raymarching:** Cast a short ray from the current volume sample toward the light source. This ray typically consists of 4 to 6 steps [S1, unofficial].
3. **Optical Thickness Integration:** Along the secondary ray, sample the density field and integrate the optical thickness $d$. The optical thickness is the integral of density along the light path.
4. **Solar Attenuation:** Calculate the transmittance $T$ of light reaching the current point from the source using Beer's Law:
   $$ T = \exp(-d) $$
   where $d$ is the integrated light-path density [S1, unofficial].
5. **Phase Function Evaluation:** Calculate the scattering probability based on the angle $\theta$ between the view ray (from camera to sample) and the sun ray (from sample to light). Use the Henyey-Greenstein phase function:
   $$ P(\theta) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g\cos\theta)^{1.5}} $$
   where $g$ is the asymmetry parameter and $\theta$ is the scattering angle [S1, unofficial].
6. **Combine:** Multiply the solar attenuation $T$ by the phase function $P(\theta)$ and the incident light intensity to get the final illumination contribution.

**Parameters:**
- $g$ (asymmetry parameter): Controls the directionality of scattering.
  - Forward-scattering lobe: $g \approx 0.8$ [S1, unofficial].
  - Mild backward-scattering lobe: $g \approx -0.2$ [S1, unofficial].
  - Combining these lobes creates edge highlights and silver linings [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volumetric_lighting(sample_pos, view_dir, sun_dir, density_field, g_forward=0.8, g_backward=-0.2):
    # Step 1: Check density
    density = density_field.sample(sample_pos)
    if density <= 0:
        return 0.0

    # Step 2 & 3: Secondary Raymarching for optical thickness
    # March 4-6 steps toward the sun
    num_steps = 5
    step_size = 1.0 # Normalized or world units depending on implementation
    optical_thickness = 0.0
    
    for i in range(num_steps):
        current_pos = sample_pos + sun_dir * (i * step_size)
        step_density = density_field.sample(current_pos)
        optical_thickness += step_density * step_size
        
    # Step 4: Solar Attenuation (Beer's Law)
    transmittance = exp(-optical_thickness)
    
    # Step 5: Phase Function (Henyey-Greenstein)
    # Calculate angle between view ray and sun ray
    # Note: view_dir is typically from camera to sample, sun_dir is from sample to sun
    cos_theta = dot(view_dir, sun_dir)
    
    # Forward scattering component
    p_forward = (1 - g_forward**2) / (4 * pi * (1 + g_forward**2 - 2 * g_forward * cos_theta)**1.5)
    
    # Backward scattering component
    p_backward = (1 - g_backward**2) / (4 * pi * (1 + g_backward**2 - 2 * g_backward * cos_theta)**1.5)
    
    # Combine lobes (weights not specified in source, assumed equal or blended)
    phase = (p_forward + p_backward) / 2.0
    
    # Step 6: Final Contribution
    # Assuming incident light intensity is 1.0 for normalized calculation
    return transmittance * phase
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of volume samples requiring lighting and $M$ is the number of secondary ray steps (typically 4-6). This is significantly more expensive than basic volume rendering as it requires a secondary raymarch per sample.
- **Memory Complexity:** Low, as it only requires temporary storage for the optical thickness integral.
- **Performance:** Computationally intensive due to the secondary raymarching. Optimizations such as quarter-resolution raymarching and bilateral upsampling are recommended to maintain frame rates [S1, unofficial].
- **Strengths:** Produces realistic light shafts, god rays, and silver linings.
- **Limitations:** Only accounts for single scattering. Multi-scattering effects are not captured. Performance degrades with higher resolution or more secondary steps.

## Background
This technique is part of the broader field of real-time volumetric rendering, specifically for simulating clouds and atmospheric effects. The use of Perlin-Worley noise for density generation and secondary raymarching for lighting is described in the context of OpenGL implementations [S1, unofficial]. The Henyey-Greenstein phase function is a standard model in radiative transfer for anisotropic scattering [S1, unofficial].

## Sources
[S1] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
