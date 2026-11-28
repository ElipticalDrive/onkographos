# ShadedVolumeStyle
Also called: Blender Volume Rendering (2.6), ISOSurfaceVolumeStyle, Volume rendering (Blender)

## Overview
ShadedVolumeStyle is a rendering style node that calculates and applies lighting effects to the visual output of a volume [S2, unofficial]. It builds on [Optical models](33-optical-models.md) and operates within an emission-absorption model using a single scattering approximation [S2, unofficial]. It is distinct from its siblings in that it applies a traditional local illumination model to volume data, ignoring source voxel values except to determine if a voxel represents a surface needing shading and to obtain the normal at that surface [S3]. It is implemented as a composable style node, allowing it to be combined with other styles such as `OpacityMapVolumeStyle` or `BoundaryEnhancementVolumeStyle` in a single rendering pass [S2, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a scalar volume field, a material definition, and optional pre-calculated surface normals as input [S3]. It outputs a shaded color and opacity for each sample point, which are then used in the front-to-back compositing equation defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) [G].

**Data Structures:**
The style is defined by the following fields [S3]:
*   `enabled` (SFBool): Enables/disables the rendering style. Default is TRUE. If FALSE, no volume data is rendered [S3].
*   `lighting` (SFBool): Controls whether shading effects are calculated. Default is FALSE. When TRUE, colour and opacity are determined based on the `material` field [S3].
*   `material` (SFNode): Specifies the material for lighting calculations. Default is NULL. If NULL, voxels are unlit and fully transparent [S3].
*   `shadows` (SFBool): Controls whether shadows are calculated. Default is FALSE. Ignored if `lighting` is FALSE. Not yet implemented [S2, unofficial].
*   `phaseFunction` (SFString): Defines the scattering model for global illumination. Default is 'Henyey-Greenstein'. Valid values include 'None' and 'Henyey-Greenstein'. Not yet implemented [S3].
*   `surfaceNormals` (SFNode): Provides pre-calculated surface normals as a 3D texture with at least 3 components. Must match base volume data dimensions exactly. If missing or mismatched, the implementation generates normals algorithmically [S3].

**Parameters:**
*   `enabled`: Default TRUE [S3].
*   `lighting`: Default FALSE [S3].
*   `shadows`: Default FALSE [S3].
*   `phaseFunction`: Default 'Henyey-Greenstein' [S3].
*   `surfaceNormals`: Default NULL [S3].

**Numbered Steps:**
1. **Check Enablement:** If `enabled` is FALSE, the style is skipped [S3].
2. **Check Lighting:** If `lighting` is FALSE, the style applies flat shading using the surface colour instead of calculating lighting effects [S3].
3. **Surface Determination:** Determine if a voxel should be shaded; this determination is the responsibility of the volume data definition, not the style itself [S3].
4. **Normal Estimation:** If the surface is determined, calculate its normal. If `surfaceNormals` provides pre-calculated normals, use them; otherwise, generate them using an implementation-specific method [S3]. The implementation supports different algorithms configured in `NormalVolumeSettings`: `precomputedCentralDifferences` (fastest), `onlineCentralDifferences` (usually most appropriate), and `onlineSobel` (nicest results) [S2, unofficial].
5. **Lighting Calculation:** Compute the lighting term using a local illumination model. The exact shading equations depend on the specified `material` node [S3].
6. **Color Application:** Apply the lighting to determine the final colour and opacity based on the `material` field [S3].
7. **Compositing:** Accumulate the shaded colour and opacity using the front-to-back alpha compositing equation [G].

**Formulas:**
The change in radiance over a differential path length $dt$ follows the general radiative transfer equation, combining extinction (loss) and source terms (gain from emission and in-scattering) [G]. Transmittance through the medium follows the Beer-Lambert law: $T = e^{-\tau}$, where $\tau$ is the optical depth [S3].

**Pitfalls:**
*   **Missing Features:** The fields `shadows` and `phaseFunction` are not yet implemented [S2, unofficial]. Consequently, the method does not account for light attenuation from sources or angle-dependent scattering in participating media [S2, unofficial].
*   **Dimension Mismatch:** If `surfaceNormals` dimensions do not match the base volume data, a warning is generated and internal normals are auto-generated [S3].
*   **Energy Conservation:** In Blender's implementation, reflection parameters do not conserve energy; scattered light color does not affect the remaining transmitted light color [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ShadedVolumeStyleRender(volume_data, material, surface_normals, enabled, lighting, shadows, phase_function):
    if not enabled:
        return []  # Style skipped [S3]

    samples = []
    for point in volume_data.ray_march():
        # Determine if voxel is a surface; responsibility of volume data definition [S3]
        if not volume_data.is_surface(point):
            continue

        # Normal estimation
        if surface_normals is not None:
            normal = surface_normals.sample(point)
        else:
            # Implementation-specific gradient estimation [S3]
            # Supports: precomputedCentralDifferences, onlineCentralDifferences, onlineSobel [S2, unofficial]
            normal = normalize(gradient(volume_data, point))

        # Lighting calculation
        if lighting and material is not None:
            # Source does not specify exact lighting formula; uses local illumination model from material [G]
            color, opacity = material.compute_lighting(point, normal)
        else:
            # Flat shading using surface colour [S3]
            color = volume_data.get_surface_color(point)
            opacity = 0.0

        samples.append((color, opacity))

    return front_to_back_composite(samples)  # Standard compositing [G]
```

## Subtopics

### Cool/Warm Shading
An illumination model where orientation with respect to the light direction is encoded by warm or cool colors, representing orientation using (almost) isoluminant colors [S5]. It sets itself apart from intensity-based shading by preserving brightness for other attributes like depth via color-based depth cueing [S5]. Choose it when visual clarity of surface orientation is required without sacrificing intensity cues [S5]. See [Cool/Warm Shading](cool-warm-shading.md).

## Operating characteristics
**Time and Memory Complexity:**
The complexity depends on the chosen normal estimation algorithm. `precomputedCentralDifferences` is the fastest execution method. `onlineCentralDifferences` is usually the most appropriate. `onlineSobel` produces the nicest results but has a higher constant factor due to its larger kernel [S2, unofficial].

**Measured Performance and Scaling:**
Performance scales with the resolution of the normal estimation grid and the step size of the ray marcher. Higher step sizes reduce render times but increase noise [S4]. Light cache resolution trades memory for shading speed [S4].

**Strengths and Limitations:**
*   **Strengths:** Easily composable with other styles; allows separation of orientation cues (via color) from intensity cues (via brightness) in cool/warm variants [S5].
*   **Limitations:** Lacks shadowing and phase function support, limiting physical accuracy [S2, unofficial]. Reflection parameters in some implementations do not conserve energy [S4].

**When to use:**
Use when visual clarity of surface orientation is required and the rendering environment supports composable style nodes [S2, unofficial]. Avoid when physically accurate shadows or complex scattering are needed [S2, unofficial].

## Background
ShadedVolumeStyle is part of the InstantReality 1.0 volume rendering framework [S2, unofficial]. The underlying concept of shading volumes to improve perception of structure dates back to early volume rendering research, with Phong shading being a standard technique [G]. Cool/warm shading was introduced as an alternative to intensity-based shading to preserve brightness for other attributes [S5].

## Sources
[S1] DTIC ADA469447: Development of an Amendment to X3D to Create a Standard Specification of Medical Image Volume Rendering, Segmentation, and Registration, https://archive.org/details/DTIC_ADA469447, fetched 2026-10-26 (unofficial)
[S2] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
[S3] ShadedVolumeStyle, http://doc.instantreality.org/documentation/nodetype/ShadedVolumeStyle, fetched 2026-10-29
[S4] Doc:2.6/Manual/Materials/Special Effects/Volume - wiki, https://wiki.blender.jp/Doc%3A2.6/Manual/Materials/Special_Effects/Volume, fetched 2026-11-03
[S5] Real-Time Advection and Volumetric Illumination for the Visualization of 3D Unsteady Flow, https://doi.org/10.2312/vissym/eurovis05/013-020, fetched 2026-10-30
