# Particle Fog Density Injection

Also called: Particle Emission Update (for fog), Volume Fog Injection

## Overview
Particle Fog Density Injection is a procedural method for adding fog density to a specific area by associating a particle emitter with volumetric fog generation. It solves the problem of defining localized, dynamic fog volumes within a scene without requiring explicit voxel data. It is distinct from [Height Fog](14108-height-fog.md), which defines density based on world height, and [Procedural Volume Data Creation](13065-procedural-volume-data-creation.md), which generally refers to broader procedural generation techniques; this method specifically relies on the particle system's spatial distribution and attributes to drive the density field.

## How it works
**Inputs and Outputs:**
- **Input:** A particle emitter instance configured in the Legacy Particle Editor.
- **Output:** A volumetric fog density field defined by the spatial extent and properties of the emitted particles.

**Data Structures:**
The method relies on the internal state of the Legacy Particle Editor. Key parameters are stored as follows:
- `Volume Fog`: Boolean flag (enabled/disabled).
- `Volume Thickness`: Float value, range [0, 10].
- `Wind Influence`: Float value, range [0, 20].
- `Enabled`: Boolean flag for the entity state.
- `Texture`: Reference to a density/alpha texture.
- `Alpha`: Float value controlling transparency.
- `Alpha Clip`: Boolean or float threshold for clipping.
- `Size`: Float value for particle dimensions.
- `Fill Rate Cost`: Integer or float metric for performance estimation.
- `Plane Align Blend Distance`: Float value for blending.
- `Tessellation Parameter`: Not supported.

**Steps:**
1. **Enable Volume Fog:** Set the `Volume Fog` parameter in the particle emitter's "Advanced" group to true.
2. **Configure Thickness:** Set `Volume Thickness` to define the density magnitude. The default or typical range is 0 to 10 [S1, S2].
3. **Apply Wind Influence:** Set `Wind Influence` to control how global wind affects the fog volume's shape and movement. Range is 0 to 20 [S1, S2].
4. **Adjust Visual Attributes:** Configure `Texture`, `Alpha`, `Alpha Clip`, `Size`, `Fill Rate Cost`, and `Plane Align Blend Distance` to refine the visual appearance and performance cost. Note that `Tessellation Parameter` is explicitly not supported [S1, S2].
5. **Render:** The renderer samples the particle positions and attributes to generate the volumetric density field during ray marching or volume rendering passes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def configure_particle_fog(particle_emitter):
    # Enable fog injection
    particle_emitter.Volume_Fog = True
    
    # Set density thickness (0-10)
    particle_emitter.Volume_Thickness = 5.0  # Example value
    
    # Set wind influence (0-20)
    particle_emitter.Wind_Influence = 10.0   # Example value
    
    # Set visual properties
    particle_emitter.Texture = "fog_texture.png"
    particle_emitter.Alpha = 0.5
    particle_emitter.Alpha_Clip = False
    particle_emitter.Size = 1.0
    particle_emitter.Fill_Rate_Cost = 100
    particle_emitter.Plane_Align_Blend_Distance = 0.0
    
    # Tessellation Parameter is not supported, do not set
    
    # Ensure emitter is enabled
    particle_emitter.Enabled = True
```

**Edge Cases and Pitfalls:**
- **Legacy Editor Only:** This method only works in the Legacy Particle Editor. It is not compatible with newer particle systems [S1, S2].
- **Tessellation:** The `Tessellation Parameter` is explicitly not supported. Attempting to use it may have no effect or cause errors [S1, S2].
- **Performance:** `Fill Rate Cost` is a parameter, indicating that high particle counts or large sizes can significantly impact rendering performance.

## Subtopics

### [Height Fog](14108-height-fog.md)
Defines fog density based on world height rather than particle positions. Use when fog needs to be tied to terrain elevation.

### [Procedural Volume Data Creation](13065-procedural-volume-data-creation.md)
A broader category of methods for generating volume data. Use when more complex procedural logic is needed beyond particle emission.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Allows for dynamic, localized fog volumes that can be animated via particle systems.
- **Limitations:** Restricted to the Legacy Particle Editor. Performance is tied to particle count and fill rate cost.
- **When to use:** When localized, dynamic fog is needed and the Legacy Particle Editor is available.
- **When not to use:** When using modern particle systems or when fog needs to be height-based.

## Background
This method is documented in the CRYENGINE documentation for both CRYENGINE 3 and CRYENGINE 5 [S1, S2]. It is part of the engine's volumetric fog system, allowing developers to use particle emitters as sources for fog density.

## Sources
[S1] CRYENGINE | Documentation - Volumetric Fog, https://www.cryengine.com/docs/static/engines/cryengine-5/categories/23756816/pages/26215326, fetched 2026-10-29
[S2] CRYENGINE | Documentation - Volumetric Fog, https://www.cryengine.com/docs/static/engines/cryengine-3/categories/1114113/pages/21268322, fetched 2026-10-29
