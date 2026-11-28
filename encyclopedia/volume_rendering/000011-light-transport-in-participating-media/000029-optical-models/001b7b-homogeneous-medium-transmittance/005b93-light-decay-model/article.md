# Light Decay Model

Also called: Light Decay Type, emLightsDecayType

## Overview
The Light Decay Model controls the rate at which emissive light from a fire source fades as it travels through a participating medium or space. It defines the geometric attenuation function applied to light intensity based on distance from the emitter. It differs from [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md), which handles absorption and scattering within the medium itself, by handling the vacuum-like geometric falloff of the source emission before medium interaction.

## How it works
The method applies a scalar decay factor to the emitted radiance based on the Euclidean distance $d$ between the light source and the point of evaluation.

**Inputs:**
- $I_{emit}$: The initial intensity of the light source.
- $d$: The distance from the light source to the point being shaded.

**Outputs:**
- $I_{decay}$: The attenuated intensity at distance $d$.

**Data Structures:**
The decay type is represented by an integer or enumeration value:
- `0`: None
- `1`: Inverse
- `2`: Inverse Square

**Steps:**
1. Calculate the distance $d$ from the light source origin to the current shading point.
2. Determine the decay factor $F(d)$ based on the selected decay type.
3. Compute the final intensity: $I_{decay} = I_{emit} \times F(d)$.

**Formulas:**
The decay factor $F(d)$ is defined as:

For **None**:
$$ F(d) = 1 $$
The light intensity remains constant regardless of distance, unless obstructed by other geometry or medium transmittance.

For **Inverse**:
$$ F(d) = \frac{1}{d} $$
At a distance of 5 units, the intensity is $1/5$ of the emitter intensity.

For **Inverse Square**:
$$ F(d) = \frac{1}{d^2} $$
At a distance of 5 units, the intensity is $1/25$ of the emitter intensity. This model is physically correct for point sources in a vacuum.

**Parameters:**
- `emLightsDecayType`: The selected decay mode. Typical values are `Inverse Square` for physical accuracy or `Inverse`/`None` for artistic control or specific simulation requirements.

**Edge Cases and Pitfalls:**
- **Zero Distance:** If $d=0$, the Inverse and Inverse Square models result in division by zero. Implementations must clamp $d$ to a minimum epsilon value or handle the singularity explicitly [G].
- **Performance:** Inverse Square decay is computationally slightly more expensive than Inverse due to the squaring operation, though negligible in modern hardware.
- **Artifacts:** Sharp circular banding artifacts may appear if the decay is too aggressive or if the sampling rate is insufficient. This is often addressed by adjusting the [Light Cut-off](#light-cut-off) parameter.

## Subtopics

### Light Cut-off
Also called: emLightsCutoff
A V-Ray specific control that specifies a threshold for light intensity below which the light will not be computed. This limits the effect of the Phoenix light to a certain distance around the simulator. Larger values cut away more light; lower values make the light range larger. At 0.0, the light is calculated for all surfaces, but rendering may slow down significantly. If sharp circular banding artifacts become visible, this parameter should be reduced.

## Operating characteristics
- **Time Complexity:** $O(1)$ per shading point for the decay calculation itself. However, the effective range of computation is limited by the [Light Cut-off](#light-cut-off), potentially reducing the number of rays traced or surfaces illuminated.
- **Memory Complexity:** $O(1)$. No additional data structures are required beyond the distance calculation.
- **Strengths:** Provides physically accurate lighting falloff (Inverse Square) or artistic control (Inverse/None).
- **Limitations:** Does not account for medium absorption or scattering; must be combined with [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md) for full participating media effects.
- **When to use:** Use Inverse Square for physically based rendering. Use Inverse or None for stylized effects or when the fire is embedded in a medium where geometric falloff is less relevant than volumetric scattering.

## Background
The Light Decay Model is a feature of the Phoenix for Maya plugin, specifically within the V-Ray integration. It allows artists to control the visual behavior of emissive fire sources. The concept of inverse square law decay is a fundamental principle in physics (radiometry), while inverse and none are artistic approximations.

## Sources
[S1] Voxel Shader - Phoenix for Maya, https://documentation.chaos.com/space/PHX4MAYA/125768009, fetched 2026-10-29
