# Translucency Tinting

Also called: Colored Light Through Glass

## Overview
Translucency Tinting is a technique that tints volumetric light beams with the color of light passing through transparent objects (such as glass or curtains) by capturing a color map during the shadow capture pass [S1, unofficial]. It differs from standard ambient occlusion methods by incorporating spectral information from occluders into the light transport calculation, rather than merely attenuating intensity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volumetric light parameters, transparent renderer materials, camera position (for culling).
*   **Outputs:** An HDR translucency map (color + depth) cached alongside the shadow map [S1, unofficial].

**Data Structures:**
*   **Translucency Map:** An HDR texture with the same resolution as the associated shadow map. It stores RGB color data and depth information [S1, unofficial].
*   **Light Volume Bounds:** Used to cull objects; only objects whose bounds intersect the light's volume are processed [S1, unofficial].

**Steps:**
1.  **Enable Translucency:** Activate the Translucency option on the light source and attach the `Volumetric Lights Translucency` component to transparent renderers (e.g., stained glass, curtains) [S1, unofficial].
2.  **Culling:** Identify transparent renderers whose bounding volumes intersect the light's volume. Distant objects outside this volume are excluded to save cost [S1, unofficial].
3.  **Render Pass:** During the OcclusionCam render pass (shared with shadow map capture), the selected transparent renderers draw their color into the HDR translucency map [S1, unofficial].
4.  **Depth Storage:** The pass simultaneously stores depth values in the translucency map [S1, unofficial].
5.  **Caching:** The translucency map is cached together with the shadow map [S1, unofficial].
6.  **Application:** During volume rendering, the beam color is tinted using the translucency map. The depth buffer ensures that only the color *behind* the glass is applied, leaving the volume in front of the glass untinted [S1, unofficial].

**Parameters:**
*   **Intensity (Light):** Scales the overall translucency effect on the light [S1, unofficial].
*   **Blending (Light):** Sets the amount of colorization applied to the light [S1, unofficial].
*   **Intensity Multiplier (Object):** Adjusts the tinting amount for a specific object [S1, unofficial].
*   **Base Texture Property Name:** Material property holding the texture (default `_BaseMap`, fallback `_MainTex`) [S1, unofficial].
*   **Base Color Property Name:** Material property holding the tint (default `_BaseColor`, fallback `_Color`) [S1, unofficial].
*   **Preserve Original Shader:** If enabled, renders the object with its own shader instead of the internal fast shader. Required for custom or animated glass shaders [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Platform Limitation:** Translucent shadow maps are not available on DirectX 12 with Forward+ or Deferred+ rendering paths. Users must use DirectX 11 or another rendering path [S1, unofficial].
*   **Front/Back Ambiguity:** Without depth storage, the tint might incorrectly apply to the volume in front of the glass. The depth buffer resolves this [S1, unofficial].

## Operating characteristics
*   **Performance:** Cost is proportional to the number of transparent objects intersecting the light volume. Distant windows outside the volume incur no cost [S1, unofficial].
*   **Memory:** Requires additional HDR texture storage equivalent to the shadow map resolution [S1, unofficial].
*   **Strengths:** Provides realistic colored light patches on surfaces and within the beam volume [S1, unofficial].
*   **Limitations:** Restricted to specific rendering paths (DX11 preferred) and requires specific component setup [S1, unofficial].

## Background
This method is part of the Volumetric Lighting system in Unity URP, as documented by Kronnect [S1, unofficial]. It extends standard shadow occlusion to include color information from translucent occluders.

## Sources
[S1] Volumetric Lighting in Unity URP: Shadows, Translucency ..., https://kronnect.com/company/blog/volumetric-lighting-unity-urp-guide/, fetched 2026-10-28 (unofficial)
