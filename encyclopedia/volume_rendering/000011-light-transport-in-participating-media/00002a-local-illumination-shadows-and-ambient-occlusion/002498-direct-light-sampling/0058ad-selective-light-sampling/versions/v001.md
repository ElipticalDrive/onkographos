# Selective Light Sampling

Also called: Fire Lights, emLightsCountEnbl, Create Fire Lights

## Overview
Selective Light Sampling is a technique used in volumetric fire simulation to approximate the illumination of participating media and surrounding geometry by the fire itself, without relying on computationally expensive Global Illumination (GI) solvers. It solves the problem of fire appearing dark or non-illuminating in scenes where GI is disabled or unavailable. It differs from standard Direct Light Sampling by automatically generating discrete light sources within the volumetric fire volume rather than sampling existing geometric lights, and from Light Sampling for NEE by targeting specific emissive phenomena (fire) rather than general scene lights.

## How it works
The method operates within the Phoenix for Maya (Phoenix FD) environment, integrated with V-Ray rendering. It approximates GI by placing virtual point lights in the brightest regions of the fire volume.

**Inputs and Outputs:**
*   **Input:** Fire simulation data (temperature/density fields), scene geometry, V-Ray renderer settings.
*   **Output:** Illuminated smoke and scene geometry, rendered image.

**Data Structures:**
The specific internal data structure for the generated lights is not given in the sources. However, the system generates a set of light sources dynamically.

**Parameters:**
*   `emLightsCountEnbl` (Create Fire Lights): Boolean. When enabled, generates lights in bright parts of the fire. [S1]
*   `emLightsPowerOnSelfMult` (Light Power on Self): Float. Multiplier for light intensity affecting the simulator's own smoke. Does not affect the fire's self-emission color/intensity. [S1]
*   `emLightsPowerMult` (Light Power on Scene): Float. Multiplier for light intensity affecting all scene objects except the Phoenix Simulator. [S1]
*   `emLightsSelfShadow` (Self Shadowing): Enum. Controls how smoke obstructs light from the fire.
    *   `None`: Smoke does not obstruct light; smoke is brightly lit. [S1]
    *   `Ray-traced`: Smoke obstructs light using ray tracing. Realistic but slower. [S1]
    *   `Grid-based`: Smoke obstructs light using a grid approximation. Faster. Requires `Light Grid Resolution (%)` parameter adjustment. [S1]
*   `Light Grid Resolution (%)`: Integer/Float. Used when `emLightsSelfShadow` is `Grid-based`. Decreasing this reduces the number of generated lights to gain speed. [S1]

**Steps:**
1.  **Check Rendering Context:** Determine if the renderer is V-Ray GPU.
    *   If V-Ray GPU: Ignore all Fire Lights options. Enable Global Illumination in V-Ray Settings if fire illumination is required. [G] (Guard based on source note).
    *   If V-Ray CPU or other supported renderer: Proceed.
2.  **Evaluate `emLightsCountEnbl`:**
    *   If `False`: Do not generate fire lights. Rely on scene lights or GI. [S1]
    *   If `True`: Proceed to generate lights.
3.  **Generate Light Sources:**
    *   Identify "bright parts" of the fire volume. [S1]
    *   Place light sources in these regions. [S1]
    *   Automatically adjust color and power of these lights based on fire properties. [S1]
4.  **Apply Intensity Multipliers:**
    *   Apply `emLightsPowerOnSelfMult` to the illumination contribution of these lights onto the simulator's own smoke. [S1]
    *   Apply `emLightsPowerMult` to the illumination contribution of these lights onto all other scene objects. [S1]
5.  **Handle Self-Shadowing:**
    *   If `emLightsSelfShadow` is `None`: Lights propagate through smoke without obstruction. [S1]
    *   If `emLightsSelfShadow` is `Ray-traced`: Perform ray tracing from light sources through smoke to calculate shadowing. [S1]
    *   If `emLightsSelfShadow` is `Grid-based`: Use grid-based approximation for shadowing. Adjust `Light Grid Resolution (%)` to balance quality and speed. [S1]
6.  **Render:**
    *   Render the scene with the generated lights. [S1]

**Edge Cases and Pitfalls:**
*   **V-Ray GPU:** All Fire Lights options are ignored. [S1]
*   **GI Conflict:** If `emLightsCountEnbl` is enabled while rendering with GI, additional light sources are still created. This may cause double illumination. Disable `emLightsCountEnbl` if natural GI illumination is desired. [S1]
*   **V-Ray Object Properties:** `Generate GI` and `Receive GI` options from V-Ray Object Properties are ignored while `emLightsCountEnbl` is enabled. [S1]

## Operating characteristics
*   **Time Complexity:** Rendering is significantly faster than using Global Illumination for fire illumination. [S1] Speed can be further improved by using `Grid-based` self-shadowing and lowering `Light Grid Resolution (%)`. [S1]
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Rendering with Create Fire Lights is "much faster" than with GI. [S1] Ray-traced self-shadowing decreases rendering performance compared to None or Grid-based. [S1]
*   **Strengths:** Simulates GI effects for fire without the computational cost of full GI. Allows control over fire's illumination of smoke and scene. [S1]
*   **Limitations:** Approximation; may not capture complex indirect lighting effects as accurately as GI. Ignored in V-Ray GPU. [S1]
*   **When to use:** When fire needs to illuminate smoke and scene objects, but GI is too slow or unavailable (e.g., V-Ray GPU). [S1]
*   **When not to use:** When high-fidelity indirect lighting is required and V-Ray GPU is not being used (use GI instead). When using V-Ray GPU. [S1]

## Background
Selective Light Sampling, referred to as "Fire Lights" in Phoenix for Maya, is a feature developed by Chaos Group for their Phoenix FD plugin. It is designed to work with V-Ray rendering. The feature allows users to simulate the illuminating effect of fire on smoke and surrounding objects without the performance penalty of Global Illumination. [S1]

## Sources
[S1] Voxel Shader - Phoenix for Maya, https://documentation.chaos.com/space/PHX4MAYA/125768009, fetched 2026-10-29
