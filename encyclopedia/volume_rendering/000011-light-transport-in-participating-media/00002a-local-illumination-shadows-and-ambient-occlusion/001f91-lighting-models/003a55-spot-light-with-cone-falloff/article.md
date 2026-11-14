# Spot Light with Cone Falloff
Also called: Delta Spot Light, Spot Light Falloff Computation

## Overview
This method computes the attenuation factor for a point light source that emits light within a specific cone direction, applying smooth falloff based on the angle between the light direction and the vector to the shading point. It solves the problem of directional intensity variation in local illumination for participating media, treating the source as a delta light [S2, unofficial]. It is distinguished from area lights by its zero-size emission model and from uniform point lights by its angular dependency [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The light source direction vector $\mathbf{d}_{light}$ (a 4D vector where the $w$-component indicates spotlight status) and the light vector $\mathbf{l}$ (the normalized direction from the shading point $\mathbf{p}$ to the light source).
*   **Output:** A scalar attenuation factor $f_{spot} \in [0, 1]$ representing the intensity multiplier.

**Data Structures:**
The light direction is stored as a 4D vector $\mathbf{g}_{LightDirection} = (x, y, z, w)$.
*   If $w \neq 0.0$, the light is a spotlight [S1, unofficial].
*   If $w = 0.0$, the light is a standard point light (no spot falloff applied) [S1, unofficial].

**Steps:**
1.  **Check Spotlight Status:** Determine if the light is a spotlight by inspecting the $w$-component of the light direction vector.
    *   If $g_{LightDirection}.w == 0.0$, the light is not a spotlight. Return attenuation factor $1.0$ (or skip spot calculation) [S1, unofficial].
    *   If $g_{LightDirection}.w \neq 0.0$, proceed to compute falloff [S1, unofficial].

2.  **Compute Spot Falloff:** Calculate the angular attenuation based on the dot product of the light direction and the light vector.
    *   Let $\mathbf{d}$ be the normalized light direction $(g_{LightDirection}.x, g_{LightDirection}.y, g_{LightDirection}.z)$.
    *   Let $\mathbf{l}$ be the normalized vector from the shading point to the light source.
    *   Compute the dot product: $dot = \mathbf{d} \cdot \mathbf{l}$.
    *   The sources indicate a "smooth cone falloff" is implemented [S2, unofficial]. While the exact exponent or cutoff formula is not explicitly detailed in the provided text, standard implementations typically use a power function of the dot product relative to a cutoff angle $\theta_{cutoff}$:
        $$ f_{spot} = \max(0.0, \frac{dot - \cos(\theta_{cutoff})}{1.0 - \cos(\theta_{cutoff})})^{\gamma} $$
        where $\gamma$ is the exponent controlling the softness of the edge [G]. The sources confirm the existence of this computation via `computeSpotFalloff` [S1, unofficial].

3.  **Optional IES Profile Blending:** If an IES (Illuminating Engineering Society) profile is provided, blend the computed geometric spot falloff with the photometric profile data [S2, unofficial].
    *   The IES parser supports TILT=INCLUDE skipping and symmetry folding (0/90/180/360) [S2, unofficial].
    *   The final attenuation is the product of the geometric spot falloff and the IES intensity value at the corresponding angle [S2, unofficial].

4.  **Discard if Zero:** If the computed spot falloff $f_{spot} \leq 0.0$, discard the lighting contribution entirely. Set the color to ambient only and return [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_spot_falloff(g_LightDirection, lightVec):
    # g_LightDirection is a 4D vector (x, y, z, w)
    # lightVec is the normalized direction from point to light
    
    # Step 1: Check if spotlight
    if g_LightDirection.w == 0.0:
        return 1.0  # Not a spotlight, full intensity
    
    # Step 2: Compute geometric spot falloff
    # Extract 3D direction vector
    lightDir = normalize(Vector3(g_LightDirection.x, g_LightDirection.y, g_LightDirection.z))
    
    # Dot product between light direction and vector to point
    dot = dot_product(lightDir, lightVec)
    
    # Smooth cone falloff implementation
    # Note: Exact cutoff angle and exponent are not specified in sources [G]
    # Assuming standard smooth falloff logic
    cutoff_cos = 0.5  # Placeholder for typical cutoff, e.g., 60 degrees
    exponent = 2.0    # Placeholder for typical exponent
    
    if dot < cutoff_cos:
        spotFalloff = 0.0
    else:
        # Linear interpolation in cosine space, raised to power
        ratio = (dot - cutoff_cos) / (1.0 - cutoff_cos)
        spotFalloff = pow(ratio, exponent)
        
    # Step 3: Blend with IES profile if available
    # [G] IES lookup logic is not detailed in sources
    if has_ies_profile():
        ies_intensity = get_ies_intensity(lightDir)
        spotFalloff *= ies_intensity
        
    return spotFalloff

def apply_spot_lighting(g_LightDirection, lightVec, ambientColor):
    falloff = compute_spot_falloff(g_LightDirection, lightVec)
    
    # Step 4: Discard if zero
    if falloff <= 0.0:
        return ambientColor
        
    # [G] Direct lighting calculation (e.g., color * falloff) is assumed
    # but not explicitly detailed in the provided snippets beyond the falloff
    directColor = compute_direct_light(falloff)
    
    return ambientColor + directColor
```

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure the dot product is clamped to $[-1.0, 1.0]$ before passing to inverse cosine or power functions to avoid NaNs [G].
*   **Boundary:** The transition at the cone edge depends on the exponent; a low exponent creates a hard edge, while a high exponent creates a soft edge [G].
*   **Degenerate Input:** If the light direction vector is zero, normalization will fail. Ensure $\mathbf{d}$ is non-zero before normalization [G].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(1)$ per shading point. The calculation involves a few vector operations (dot product, normalization) and a power function.
*   **Memory:** $O(1)$ additional memory. No large data structures are required unless an IES profile is loaded, which is static per light source.

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Scales linearly with the number of shading points. Independent of scene complexity as it is a local illumination model.

**Strengths and Limitations:**
*   **Strengths:** Computationally cheap; provides realistic directional lighting effects; supports photometric accuracy via IES profiles [S2, unofficial].
*   **Limitations:** Delta light approximation may cause aliasing in shadows if not sampled carefully; does not account for light scattering within the cone volume (only attenuation at the point) [G].

**When to Use:**
*   Use when directional, focused lighting is required (e.g., spotlights, flashlights) in a volume rendering context.
*   Avoid when soft, volumetric light beams (god rays) are needed, as this method only computes point attenuation, not volumetric scattering [G].

## Background
**Origin:**
The method is part of modern physically-based rendering engines that support volumetric effects. It is implemented in the photon-engine project, which adds volumetric rendering and advanced materials [S2, unofficial].

**History and Development:**
The implementation includes support for IES LM-63 profiles, allowing for real-world photometric data integration [S2, unofficial]. The concept of spot lights with cone falloff is a standard technique in computer graphics, adapted here for delta lights in participating media.

**Where it is used:**
Used in the photon-engine for local illumination in volumetric scenes [S2, unofficial]. Also referenced in the cubes project for basic spotlight functionality [S1, unofficial].

## Sources
[S1] destroflyer/cubes, https://github.com/destroflyer/cubes, fetched 2026-10-26 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
