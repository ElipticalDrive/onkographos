# Multiple Scattering via Volume Trace Depth

Also called: Volume Trace Depth, Volume Trace Depth with Multiple Scattering, Volumetric Multiple Scattering

## Overview
This method enables volumetric multiple scattering by controlling the maximum number of scattering bounces computed within participating media, transforming dense volumes from opaque grey masses into luminous, believable objects [S1, unofficial][S2]. It is distinct from single-scattering approximations by explicitly simulating indirect light transport within the volume, but it is computationally more expensive and requires Brute Force Global Illumination, unlike faster heuristic shortcuts [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** The Volume Trace Depth parameter (integer), scene lighting, and volume density/emission properties.
*   **Output:** Radiance values for volume pixels, including indirect illumination contributions from multiple bounces.

**Data Structures:**
Not given in the sources. The implementation relies on the underlying Brute Force Global Illumination engine's path tracing or photon mapping structures.

**Steps:**
1.  Initialize the renderer with Brute Force Global Illumination enabled. This is a strict prerequisite; volumetric multiple scattering will not function otherwise [S1, unofficial][S2].
2.  Set the `Volume Trace Depth` parameter to an integer value $N > 1$. The default value is not specified in the sources [S1, unofficial][S2].
3.  For each ray intersection with a volume:
    a. Trace the path through the volume, computing scattering events up to depth $N$.
    b. Accumulate radiance from direct lighting and indirect lighting from previous bounces.
    c. If the volume is emissive, include self-illumination contributions that scatter into surrounding media [S1, unofficial][S2].
4.  Output the final accumulated radiance.

**Formulas:**
Not given in the sources. The method is described as a render setting rather than a standalone mathematical formula.

**Parameters:**
*   **Volume Trace Depth:** Controls the maximum number of scattering bounces. Raising this value increases the number of simulated bounces, leading to exponentially growing render cost [S1, unofficial][S2]. Typical values are not specified, but higher values are needed for forward-scattering clouds [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Performance:** Render cost grows exponentially with increased trace depth [S1, unofficial][S2].
*   **GI Requirement:** Must use Brute Force Global Illumination; other GI methods are incompatible [S1, unofficial][S2].
*   **Diminishing Returns:** Isotropic smoke shows diminishing returns at lower trace depths compared to forward-scattering clouds [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for enabling and using Volume Trace Depth
# Note: This is a high-level representation of the render setting interaction.
# The actual scattering simulation is handled by the Brute Force GI engine.

def render_volume_with_multiple_scattering(scene, volume_trace_depth):
    # Step 1: Ensure Brute Force Global Illumination is enabled
    # This is a strict requirement for volumetric multiple scattering
    if not scene.gi_method == "Brute Force":
        raise Error("Volumetric multiple scattering requires Brute Force Global Illumination")
    
    # Step 2: Set the Volume Trace Depth parameter
    # Default value is not specified in sources
    scene.volume_trace_depth = volume_trace_depth
    
    # Step 3: Render the scene
    # The renderer will internally handle the multiple scattering bounces
    # up to the specified depth
    image = render(scene)
    
    return image

# Example usage
# scene = Scene()
# scene.gi_method = "Brute Force"
# scene.volume_trace_depth = 5  # Arbitrary value, default unknown
# image = render_volume_with_multiple_scattering(scene, 5)
```

## Subtopics

### Extinction Falloff
A shortcut that approximates hundreds of additional bounces by reducing extinction per bounce, with a default value of 0.2 [S1, unofficial][S2].

### Anisotropy Falloff
A shortcut mentioned alongside Extinction Falloff, though details are not provided in the sources [S1, unofficial][S2].

## Operating characteristics
*   **Time Complexity:** Exponentially growing render cost as trace depth increases [S1, unofficial][S2].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
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
