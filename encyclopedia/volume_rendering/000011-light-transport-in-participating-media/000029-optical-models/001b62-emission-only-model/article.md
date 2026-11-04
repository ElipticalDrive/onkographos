# Emission-only Model

Also called: Direct Volume Rendering (Emission Model), Emission Model, Glow Only Model, Glow model, Glowing transparent gas

## Overview
The Emission-only Model simulates a participating medium that adds light to a viewing ray via emission (e.g., hot tenuous gas, fluorescent stains) while neglecting absorption and scattering [S1, unofficial][S3, unofficial]. It solves for the accumulated glow intensity along a ray, distinguishing itself from the [Absorption-only model](839-absorption-only-model.md) by adding radiance rather than attenuating it, and from the [Standard volume rendering equation](1165-standard-volume-rendering-equation.md) by omitting the exponential attenuation term [S1, unofficial][S2, unofficial]. Unlike the [Single Scattering Model](1053-single-scattering-model.md), it assumes isotropic emission independent of incoming light [S3, unofficial].

## How it works
**Inputs:**
- Scalar field $f(\mathbf{x})$ defined on a 3D grid.
- Viewing ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Initial intensity $I_0$ (background radiance).
- Emission constant $C$ (or function $C(s)$).

**Outputs:**
- Final radiance $I(s)$ at distance $s$ along the ray.

**Data Structures:**
- Scalar field $f$: Typically a 3D array or texture. Values are interpolated (e.g., trilinear) to obtain $f(\mathbf{x})$ at arbitrary positions.
- Source term $g(s)$: Derived from $f$. Typically $g(s) = C \cdot f(\mathbf{x}(s))$ or $g(s) = C \cdot \tau(\mathbf{x}(s))$, where $\tau$ acts as a density proxy for emission [S1, unofficial][S3, unofficial].

**Steps:**
1. **Interpolate:** For a sample point $\mathbf{x}$ on the ray, retrieve the scalar value $f(\mathbf{x})$ from the volume data.
2. **Map to Source Term:** Compute the emission source term $g(s)$. In the simplest case, $g(s)$ is proportional to the scalar value: $g(s) = C \cdot f(\mathbf{x}(s))$ [S1, unofficial][S3, unofficial]. Here, $C$ is a constant scaling factor, and $f$ represents the density or intensity of emitting particles [S5].
3. **Integrate:** Accumulate the source term along the ray from the entry point ($s=0$) to the current distance $s$. The differential equation is:
   $$ \frac{dI}{ds} = g(s) $$
   The solution is:
   $$ I(s) = I_0 + \int_0^s g(t) dt $$
   In discrete ray marching with step size $\Delta s$, this becomes:
   $$ I_{k+1} = I_k + g(s_k) \Delta s $$

**Parameters:**
- $C$: Emission constant. Must be carefully scaled to prevent overflow [S1, unofficial].
- $I_0$: Background intensity.

**Pitfalls:**
- **Unbounded Intensity:** Because there is no attenuation, the integral has no upper bound. Accumulated intensity can easily exceed the representable range of the output device (clipping to 1.0 or max uint8) [S1, unofficial][S3, unofficial].
- **Scaling:** The constant $C$ must be set small enough so that the center of dense regions does not saturate, which may cause tenuous edges to appear too dark or invisible [S1, unofficial].
- **Order Independence:** Like Maximum Intensity Projection (MIP), the emission-only model is order-independent because addition is commutative [S4, unofficial]. This means spatial ordering of features along the ray is lost; a small amount of bright emission can overwhelm larger amounts of dimmer emission [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_emission_only(ray_origin, ray_direction, volume_field, C, I0, num_steps, step_size):
    """
    Renders a ray using the emission-only model.
    
    Args:
        ray_origin: (x, y, z) origin of the ray.
        ray_direction: (dx, dy, dz) normalized direction vector.
        volume_field: Function or object providing f(x, y, z) scalar values.
        C: Emission constant (scalar).
        I0: Initial background intensity (scalar or RGB).
        num_steps: Number of samples along the ray.
        step_size: Distance between samples.
        
    Returns:
        Final intensity I(s).
    """
    I = I0
    current_pos = ray_origin
    
    for i in range(num_steps):
        # 1. Interpolate scalar field f(x)
        # Assuming volume_field.interpolate(x, y, z) returns f(x)
        f_val = volume_field.interpolate(current_pos.x, current_pos.y, current_pos.z)
        
        # 2. Map to source term g(s)
        # g(s) = C * f(x) [S1, unofficial][S3, unofficial]
        g = C * f_val
        
        # 3. Integrate: I_new = I_old + g * ds
        I = I + g * step_size
        
        # Advance position
        current_pos = current_pos + ray_direction * step_size
        
    return I
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. Identical to basic ray marching.
- **Memory Complexity:** $O(1)$ per ray (excluding volume storage).
- **Performance:** Very fast, as it avoids computing transmittance (exponentials) and scattering integrals [S1, unofficial].
- **Strengths:** Simple to implement; order-independent; useful for simulating tenuous glowing media like fluorescent stains or hot gases where absorption is negligible [S1, unofficial].
- **Limitations:** Physically inaccurate for dense media; prone to saturation/clipping due to lack of attenuation; loses depth ordering information [S1, unofficial][S4, unofficial].
- **When to use:** When simulating transparent, glowing materials where absorption is negligible, or for quick visualization of scalar field intensity without occlusion effects.
- **When not to use:** For dense media like smoke or clouds where occlusion is significant; when depth ordering is required.

## Background
The emission-only model is a foundational optical model in direct volume rendering, introduced by Max in 1995 as a limiting case of the more general emission-absorption model [S1, unofficial]. It arises when the particle size or number density approaches zero while emission intensity compensates, effectively neglecting absorption [S1, unofficial]. It is often used in conjunction with Maximum Intensity Projection (MIP) concepts, as both are order-independent [S4, unofficial]. The model is particularly relevant in medical imaging for simulating fluorescent stains and in scientific visualization for neutron transport or hot tenuous gases [S1, unofficial][S3, unofficial].

## Sources
[S1] Optical Models for Direct Volume Rendering, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S4] Direct Volume Rendering, https://cscheid.net/writing/data_visualization/direct_volume_rendering/, fetched 2026-10-29 (unofficial)
[S5] Direct_Volume_Rendering.md, https://github.com/pyvista/pyvista/files/9562449/Direct_Volume_Rendering.md, fetched 2026-10-29
