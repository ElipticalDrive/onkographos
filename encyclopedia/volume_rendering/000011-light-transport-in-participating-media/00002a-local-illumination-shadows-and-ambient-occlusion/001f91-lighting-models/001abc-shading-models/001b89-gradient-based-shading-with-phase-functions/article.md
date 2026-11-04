# Gradient-based Shading with Phase Functions

Also called: Lambert's Law Phase Function Shading, Shading and Illumination, Shading and Shadows

## Overview
Gradient-based shading with phase functions adds illumination terms to the volume rendering integral to simulate single-scattering effects or computes shading for implicit iso-surfaces, thereby providing a sense of depth that absorption and emission alone cannot achieve [S1, unofficial][S2, book]. It distinguishes itself from sibling methods by explicitly deriving surface normals from the scalar field gradient and applying phase functions (such as Lambert's law or Henyey-Greenstein) to model light interaction, rather than relying solely on frequency domain approximations or simplified Phong models [S1, unofficial][S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume field $f$, a ray origin $o$ and direction $r$, light source positions, and material parameters (e.g., scattering coefficient, phase function parameters).
*   **Output:** A shaded color value for the ray intersection point.

**Data Structures:**
*   **Scalar Field:** A 3D grid of floating-point values representing density or intensity.
*   **Iso-surface Definition:** A special transfer function that evaluates to 1 for a specific discrete input value and 0 for all others [S1, unofficial].

**Steps:**
1.  **Ray-Volume Intersection:** Trace the ray through the volume. If shading an iso-surface, find the intersection point where the scalar field value matches the iso-value [S1, unofficial].
2.  **Gradient Computation:** Compute the normal vector $\mathbf{n}$ at the intersection point. For a continuous field, $\mathbf{n} = \nabla f / \|\nabla f\|$. For discrete samples, use central differencing on the neighbors of the sample point [S2, book].
    *   *Pitfall:* If $\|\nabla f\| \approx 0$, the normal is undefined. [G] In this case, default to a zero vector or a predefined normal, though this results in no shading variation.
3.  **Phase Function Evaluation:** Calculate the phase function $P(\alpha)$ based on the angle $\alpha$ between the incoming ray direction and the normal vector $\mathbf{n}$.
    *   **Lambert's Law Derived Phase Function:** For diffuse particles much larger than the light wavelength, use [S2, book]:
        $$ P_L(\alpha) = \frac{8}{3\pi} (\sin\alpha + (\pi - \alpha)\cos\alpha) $$
        where $\alpha$ is the angle between the incoming ray $\mathbf{r}$ and the normal $\mathbf{n}$ [S2, book].
    *   **Henyey-Greenstein (Alternative):** For multiple scattering attenuation, the Henyey-Greenstein function can be used. Schlick's approximation is often preferred for speed [S2, book]:
        $$ P_S(\alpha) = \frac{1-g^2}{(1-g\cos\alpha)^2} $$
        where $g$ is the anisotropy factor [S2, book].
4.  **Shading Application:** Combine the phase function with lighting models. This can include Phong shading with multiple light sources or global illumination techniques [S1, unofficial]. For shadows, a two-pass approach may be used: first render the volume from the light source using the absorption model to store light contribution, then apply this to the view pass [S2, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_shading(volume, ray_origin, ray_dir, light_positions, iso_value):
    # 1. Find intersection with iso-surface
    # [G] Assumes a ray-marching or intersection routine exists
    t, point = find_iso_surface_intersection(volume, ray_origin, ray_dir, iso_value)
    if point is None:
        return volume.get_value(point) # No shading, return raw value or black

    # 2. Compute normal via central differencing
    # [G] Assumes volume has a method to get neighbor values
    dx = volume.get_value(point + (1, 0, 0)) - volume.get_value(point - (1, 0, 0))
    dy = volume.get_value(point + (0, 1, 0)) - volume.get_value(point - (0, 1, 0))
    dz = volume.get_value(point + (0, 0, 1)) - volume.get_value(point - (0, 0, 1))
    
    gradient = Vector3(dx, dy, dz)
    norm_magnitude = gradient.length()
    
    if norm_magnitude < 1e-6:
        normal = Vector3(0, 0, 1) # [G] Default normal if gradient is zero
    else:
        normal = gradient / norm_magnitude

    # 3. Compute shading contribution
    color = Vector3(0, 0, 0)
    for light_pos in light_positions:
        light_dir = (light_pos - point).normalize()
        
        # Angle between incoming ray (view direction) and normal
        # Note: In standard shading, we use light_dir. 
        # The source defines alpha as angle between incoming ray r and normal n.
        # In a ray tracer, 'r' is the view direction. For illumination, 
        # we typically look at the angle between the light direction and normal.
        # However, the source [S2, book] explicitly states: "theta ... is the angle 
        # between the incoming ray r and the normal vector n."
        # In the context of the volume rendering integral, 'r' is the ray direction.
        # If this is for single scattering from a light source, we usually 
        # calculate the phase function for the scattering angle.
        # Assuming 'r' in the formula refers to the direction of light propagation 
        # towards the point (i.e., -light_dir) or the view ray depending on 
        # the specific scattering model. 
        # Given the context of "shading of an implicit surface", standard 
        # Phong/Lambert uses the angle between the surface normal and the 
        # light direction. The source [S2, book] text is slightly ambiguous 
        # ("incoming ray r"). In volume rendering, the ray is the view ray.
        # If we are shading the volume based on the view ray, it's a 
        # view-dependent effect. If we are shading based on light, it's 
        # light-dependent. The formula P_L is for diffuse scattering.
        # Let's assume standard shading: angle between normal and light direction.
        
        # Re-reading [S2, book]: "theta ... is the angle between the incoming ray r 
        # and the normal vector n." In the context of the paragraph discussing 
        # "Shading and Shadows", it likely refers to the light ray.
        # However, in ray tracing, the "ray" is the view ray.
        # Let's stick to the literal definition in [S2, book] for the phase function 
        # argument, but apply it to the light direction for illumination.
        
        # Calculate angle between normal and light direction
        cos_alpha = dot(normal, light_dir)
        alpha = acos(clamp(cos_alpha, -1.0, 1.0))
        
        # Lambert's Law Phase Function
        phase = (8.0 / (3.0 * PI)) * (sin(alpha) + (PI - alpha) * cos(alpha))
        
        # [G] Standard diffuse shading: intensity * phase
        # Assuming diffuse material with albedo 1 for simplicity
        intensity = max(0.0, cos_alpha) # Lambertian reflectance
        color += light_dir * intensity * phase

    return color
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray for gradient computation and shading evaluation. Shadow rays add a multiplicative factor equal to the number of light sources.
*   **Memory Complexity:** Depends on the volume storage. Gradient computation requires accessing neighboring voxels, which is cache-friendly if the volume is contiguous.
*   **Strengths:** Provides intuitive depth cues; computationally efficient compared to full multiple scattering solutions; allows for standard surface shading techniques (Phong) to be applied to volumes [S1, unofficial][S2, book].
*   **Limitations:** Relies on the assumption that the volume can be approximated by an iso-surface or that local gradients are meaningful; may produce artifacts if the volume is noisy or has low resolution [S2, book].
*   **When to use:** When a sense of depth and surface-like appearance is desired in a volume without the cost of full light transport simulation.
*   **When not to use:** When accurate multiple scattering or complex internal lighting effects are required; when the volume lacks a clear iso-surface structure.

## Background
This technique originates from the need to enhance the visual quality of direct volume rendering by incorporating lighting models traditionally used for surfaces [S1, unofficial][S2, book]. The use of gradients for normal estimation in discrete volumes was established by Höhne and Bernstein [S2, book]. The application of Lambert's law derived phase functions for diffuse particles larger than the wavelength was described by Blinn [S2, book]. The integration of these concepts into volume rendering pipelines allows for the simulation of single-scattering effects and shadows, significantly improving the perceptibility of depth [S1, unofficial][S2, book].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
