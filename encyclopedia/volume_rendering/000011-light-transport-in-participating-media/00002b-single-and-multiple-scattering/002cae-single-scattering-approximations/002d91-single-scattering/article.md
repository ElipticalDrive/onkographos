# Single Scattering

Also called: Direct Lighting in Participating Media

## Overview
Single scattering approximates the in-scattered radiance at a point $\mathbf{x}$ by assuming it originates directly from light sources, attenuated by the transmittance from the source to $\mathbf{x}$, and travels directly to the camera, attenuated by the transmittance from $\mathbf{x}$ to the camera [S2][S3, book]. This method solves the light transport problem in participating media by ignoring multiple bounces, reducing the integral equation to a form similar to direct lighting [S1, unofficial]. It is distinguished from [Single Scattering Approximations](5340-single-scattering-approximations.md) by its specific implementation via Next Event Estimation (NEE) over the spherical domain, and from [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md) by its general applicability to arbitrary volumes rather than just atmospheric models.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera origin $\mathbf{o}$, direction $\boldsymbol{\omega}$, scene geometry, light sources (position, intensity, geometry), volume properties (scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, extinction coefficient $\sigma_t = \sigma_s + \sigma_a$), and phase function $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$.
*   **Outputs:** Radiance $L_o(\mathbf{o}, \boldsymbol{\omega})$ for the pixel.

**Data Structures:**
*   **Light List:** A list of light sources. Each light has position $\mathbf{l}$, intensity $L_e$, and potentially geometric extent (e.g., area light).
*   **Volume Grid/Texture:** Stores $\sigma_s(\mathbf{x})$, $\sigma_a(\mathbf{x})$, and $\sigma_t(\mathbf{x})$ at query points $\mathbf{x}$.
*   **Phase Function:** A function evaluating $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$, often Henyey-Greenstein or Mie.

**Numbered Steps:**
1.  **Trace Ray:** Cast a ray $\mathbf{r}(t) = \mathbf{o} + t\boldsymbol{\omega}$ from the camera.
2.  **Compute Eye Transmittance:** Calculate the transmittance from the camera origin $\mathbf{o}$ to a sample point $\mathbf{x}$ on the ray:
    $$ T(\mathbf{o}, \mathbf{x}) = \exp\left(-\int_{\mathbf{o}}^{\mathbf{x}} \sigma_t(\mathbf{u}) d\mathbf{u}\right) $$
    This is typically computed via discrete integration or lookup in a precomputed texture [S2][S3, book].
3.  **Enumerate Lights (NEE):** For each light source $i$ in the scene:
    a.  Compute the vector $\mathbf{l}_i$ from $\mathbf{x}$ to the light.
    b.  Compute the distance $d_i = \|\mathbf{l}_i\|$.
    c.  Compute the transmittance from $\mathbf{x}$ to the light $T(\mathbf{x}, \mathbf{l}_i)$, accounting for occlusion by geometry or other volumes.
    d.  Compute the incident radiance $L_i(\mathbf{x}, \boldsymbol{\omega}_i)$ from the light, including geometric attenuation (e.g., $1/d_i^2$ for point lights) and the phase function $p(\boldsymbol{\omega}_i, \boldsymbol{\omega})$.
4.  **Integrate In-Scattered Radiance:** Sum the contributions from all lights. The in-scattered radiance $L_s$ at $\mathbf{x}$ is:
    $$ L_s(\mathbf{x}, \boldsymbol{\omega}) = \sum_{i} T(\mathbf{x}, \mathbf{l}_i) \cdot \sigma_s(\mathbf{x}) \cdot L_i(\mathbf{x}, \boldsymbol{\omega}_i) \cdot p(\boldsymbol{\omega}_i, \boldsymbol{\omega}) \cdot \Delta V_i $$
    where $\Delta V_i$ is the effective volume element or sampling weight associated with light $i$ [S1, unofficial].
5.  **Composite:** Combine the in-scattered radiance with the background radiance $L_{bg}$ attenuated by the eye transmittance:
    $$ L_o(\mathbf{o}, \boldsymbol{\omega}) = T(\mathbf{o}, \mathbf{p}_{far}) L_{bg} + \int_{\mathbf{o}}^{\mathbf{p}_{far}} T(\mathbf{o}, \mathbf{x}) L_s(\mathbf{x}, \boldsymbol{\omega}) d\mathbf{x} $$
    The integral is approximated by discrete summation along the ray segments [S2][S3, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering_ray_trace(camera_origin, camera_direction, scene, lights):
    # Initialize accumulated radiance
    L_accum = vec3(0.0)
    
    # Current position along the ray
    t = 0.0
    # Eye transmittance accumulator
    T_eye = 1.0
    
    # Sample points along the ray
    while t < scene.max_distance:
        x = camera_origin + t * camera_direction
        
        # 1. Compute eye transmittance to current point (updated from previous step)
        # In practice, T_eye is updated incrementally or via texture lookup
        sigma_t = scene.volume.get_extinction(x)
        dt = scene.volume.get_step_size(x)
        T_eye *= exp(-sigma_t * dt)
        
        # 2. Compute in-scattered radiance at x
        L_in_scattered = vec3(0.0)
        
        for light in lights:
            # Vector to light
            l_vec = light.position - x
            d = length(l_vec)
            if d < 1e-6: continue
            
            omega_i = normalize(l_vec)
            
            # Transmittance from x to light (shadow ray)
            T_light = scene.compute_transmittance(x, light.position)
            
            # Incident radiance from light
            L_light = light.intensity / (d * d) # Inverse square law
            
            # Phase function
            phase = scene.phase_function(omega_i, camera_direction)
            
            # Scattering coefficient
            sigma_s = scene.volume.get_scattering(x)
            
            # Contribution
            # Note: The integral term is approximated by summing contributions
            # The volume element dt is handled by the loop structure
            L_in_scattered += T_light * sigma_s * L_light * phase * dt
            
        # 3. Add to accumulated radiance
        # The term T_eye * L_in_scattered represents the light scattered at x
        # and reaching the camera
        L_accum += T_eye * L_in_scattered
        
        # 4. Update t
        t += dt
        
    # 5. Add background
    L_accum += T_eye * scene.background_radiance(camera_direction)
    
    return L_accum
```

**Parameters:**
*   **Step Size ($dt$):** Depends on volume resolution and desired accuracy. Smaller steps reduce noise but increase cost [S1, unofficial].
*   **Phase Function:** Henyey-Greenstein is common for its efficiency [S1, unofficial]. Mie scattering is used for more physical accuracy [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** Transmittance can underflow to zero, causing artifacts. Use epsilon checks [G].
*   **Boundaries:** Ensure ray tracing handles volume entry/exit correctly [G].
*   **Degenerate Input:** Lights inside the volume or at the camera origin require special handling [G].

## Subtopics

### Monochromatic Mie Scattering Shading
This variant treats Mie scattering intensity $I_{SM}$ as monochromatic, occupying only one channel in the scattering texture [S2]. During rendering, this monochromatic intensity is multiplied by the attenuated direct illumination $I_0^I$ [S2]. An alternative is to use the proportionality rule from [Bruneton and Neyret 2008] instead of direct multiplication [S2].

### Simple Compositing Illumination Approximation
This approximation ignores scattering completely, reducing the illumination equation to standard volume compositing [S3, book]. The intensity is computed as:
$$ I(t_0, \vec{\omega}) = \int_{t_0}^{\infty} E(\tau) e^{(-\int_{t_0}^{t} \sigma_a(\tau) \, d\tau)} \, d\tau $$
Discretized, this becomes:
$$ I(t_{near}, \vec{\omega}) = \sum_{t = t_{near}}^{t \leq t_{far}} \alpha(t) \prod_{t' = t_{near}}^{t' < t_{far}}(1 - a(t')) $$
Or recursively:
$$ I(t_n, \vec{\omega}) = \alpha(t_n) + (1 - \alpha(t_n)) I(t_{n+1}, \vec{\omega}) $$
This produces unrealistic images as it lacks scattering effects [S3, book].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of ray samples and $L$ is the number of lights. Each sample requires a shadow ray to each light [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for volume storage, where $V$ is the volume resolution.
*   **Performance:** With equi-angular sampling, rendering can be achieved in less than 10 minutes per image with <100 spp and spr=4, using acceleration structures like Embree [S1, unofficial].
*   **Strengths:** Physically plausible for thin or sparse media; efficient for scenes with few lights.
*   **Limitations:** Fails for dense or highly scattering media where multiple bounces are significant [S3, book].
*   **When to Use:** Thin fog, smoke, or when real-time performance is critical and multiple scattering is negligible.
*   **When Not to Use:** Dense clouds, water, or scenes with strong indirect lighting effects from scattering.

## Background
Single scattering is a fundamental approximation in volume rendering, derived from the radiative transfer equation. It is widely used in computer graphics for rendering participating media such as fog, smoke, and clouds. The method is detailed in various graphics literature, including CMU lecture notes on the volume rendering equation [S1, unofficial]. The specific implementation using Next Event Estimation is common in path tracers [S1, unofficial]. The monochromatic Mie scattering approach was proposed for real-time spectral scattering in natural media [S2]. The simple compositing approximation is discussed in VTK documentation [S3, book].

## Sources
[S1] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S2] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
[S3] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
