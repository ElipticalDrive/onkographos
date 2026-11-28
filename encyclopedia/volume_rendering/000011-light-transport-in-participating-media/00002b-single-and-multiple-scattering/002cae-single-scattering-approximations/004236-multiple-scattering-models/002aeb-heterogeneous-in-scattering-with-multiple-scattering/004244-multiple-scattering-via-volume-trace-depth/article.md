# Multiple Scattering via Volume Trace Depth

Also called: Volume Trace Depth, Volume Trace Depth with Multiple Scattering, Volumetric Multiple Scattering

## Overview
This method enables volumetric multiple scattering by controlling the maximum number of scattering bounces computed within participating media, transforming dense volumes from opaque grey masses into luminous, believable objects [S1, unofficial][S2]. It is distinct from single-scattering approximations by explicitly simulating indirect light transport within the volume, but it is computationally more expensive and requires Brute Force Global Illumination, unlike faster heuristic shortcuts [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume Trace Depth parameter (integer), scene lighting, and volume density/emission properties [S1, unofficial][S2].
*   **Output:** Radiance values for volume pixels, including indirect illumination contributions from multiple bounces [S1, unofficial][S2].

**Data Structures:**
Not given in the sources [S1, unofficial][S2].

**Steps:**
1.  Enable Brute Force Global Illumination, which is a strict requirement for volumetric multiple scattering to function [S1, unofficial][S2].
2.  Set the `Volume Trace Depth` parameter to an integer value $N$ [G]. The default value is not specified in the sources [S1, unofficial][S2].
3.  Trace the path through the volume, computing scattering events up to depth $N$ [G].
4.  Accumulate radiance from direct lighting and indirect lighting from previous bounces [G].
5.  Output the final accumulated radiance [G].

**Formulas:**
Not given in the sources. The method is described as a render setting rather than a standalone mathematical formula [S1, unofficial][S2].

**Parameters:**
*   **Volume Trace Depth:** Controls the maximum number of scattering bounces [S1, unofficial]. Raising this value increases the number of simulated bounces, leading to exponentially growing render cost [S1, unofficial][S2]. Typical values are not specified, but higher values are needed for forward-scattering clouds [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Performance:** Render cost grows exponentially with increased trace depth [S1, unofficial][S2].
*   **GI Requirement:** Must use Brute Force Global Illumination; other GI methods are incompatible [S1, unofficial][S2].
*   **Diminishing Returns:** Isotropic smoke shows diminishing returns at lower trace depths compared to forward-scattering clouds [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volume_radiance(scene, volume_trace_depth):
    # Step 1: Validate Brute Force Global Illumination requirement
    if scene.gi_method != "Brute Force":
        raise ValueError("Volumetric multiple scattering requires Brute Force Global Illumination")
    
    # Step 2: Set the Volume Trace Depth parameter
    scene.volume_trace_depth = volume_trace_depth
    
    # Step 3: Trace paths through the volume up to depth N
    radiance = 0.0
    for bounce in range(scene.volume_trace_depth):
        # Engine-specific scattering calculation up to the current bounce depth [G]
        # The sources do not provide the internal radiance accumulation algorithm [S1, unofficial][S2]
        pass
    
    return radiance
```

## Subtopics

### Extinction Falloff
A shortcut that approximates hundreds of additional bounces by reducing extinction per bounce, with a default value of 0.2 [S1, unofficial]. Choose it when rendering dense volumes where full multiple scattering is too expensive, as it provides a brighter, more realistic result at lower render times [S1, unofficial].

### Anisotropy Falloff
A shortcut mentioned alongside Extinction Falloff, though details are not provided in the sources [S1, unofficial]. Choose it when anisotropic scattering behavior requires adjustment, but refer to engine documentation for exact implementation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Exponentially growing render cost as trace depth increases [S1, unofficial][S2].
*   **Memory Complexity:** Not given in the sources [S1, unofficial][S2].
*   **Measured Performance:** Not given in the sources [S1, unofficial][S2].
*   **Scaling:** Performance degrades significantly with higher trace depth values [S1, unofficial][S2].
*   **Strengths:** Transforms dense volumes into luminous, believable objects; essential for forward-scattering clouds and emissive fire [S1, unofficial][S2].
*   **Limitations:** High computational cost; requires Brute Force Global Illumination; diminishing returns for isotropic smoke [S1, unofficial][S2].
*   **When to use:** When realistic, luminous dense volumes are required, especially for forward-scattering materials like clouds or emissive fire [S1, unofficial][S2].
*   **When not to use:** When performance is critical and isotropic smoke is used, or when Brute Force Global Illumination is not available [S1, unofficial][S2].

## Background
This method is exposed as a render setting in Redshift, developed by Maxon [S1, unofficial][S2]. It is part of the broader effort to make dense volumes appear realistic by simulating multiple scattering events, which are necessary for dense volumes to come alive [S1, unofficial][S2].

## Sources
[S1] What Is Volumetric Rendering? Volume Shaders Explained, from ... - Maxon, https://www.maxon.net/en/article/volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
