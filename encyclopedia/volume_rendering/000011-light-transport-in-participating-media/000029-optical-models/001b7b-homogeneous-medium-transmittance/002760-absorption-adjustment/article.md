# Absorption Adjustment

Also called: Opacity Adjustment

## Overview
Absorption Adjustment corrects the opacity parameter $\alpha$ used in planar smoke rendering to account for non-orthogonal viewing angles and variable plane spacing [S1, unofficial]. It adjusts the absorption parameter $\alpha$ for view directions not aligned with the axis orthogonal to the viewing planes, and when the distance between adjacent smoke planes changes or viewing planes are skipped [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Base absorption coefficient $\alpha_0$ (derived from the medium's extinction coefficient and plane spacing), viewing direction vector, and plane normal vector [S1, unofficial][G].
*   Output: Adjusted absorption parameter $\alpha'$ suitable for OpenGL blending [S1, unofficial].

**Data Structures:**
The method operates on scalar values. In the source, $\alpha$ is a float in the range $[0.0, 1.0]$, where $0.0$ represents completely transparent and $1.0$ represents completely opaque [S1, unofficial].

**Steps:**
1.  Determine the angle $\theta$ between the viewing direction and the plane normal [G].
2.  Calculate the effective path length through the plane based on $\theta$ [G].
3.  Adjust the absorption coefficient if the viewing direction is not aligned with the axis orthogonal to the viewing planes [S1, unofficial].
4.  Adjust the absorption coefficient if the distance between adjacent smoke planes changes or viewing planes are skipped [S1, unofficial].
5.  Compute the final opacity for blending using the adjusted coefficient [G].

**Formulas:**
The base absorption parameter $\alpha$ relates to the total mass extinction coefficient $\sigma_t$ and the distance between grid planes $\Delta x$ via the Beer-Lambert law:
$$ \alpha = 1 - e^{-\sigma_t \Delta x} $$
When adjusting for view direction, the effective thickness scales by $1/\cos(\theta)$, yielding:
$$ \alpha' = 1 - e^{-\sigma_t \frac{\Delta x}{\cos(\theta)}} $$
If planes are skipped or spacing changes to $\Delta x'$, the coefficient is recalculated using the new distance [S1, unofficial].

**Parameters:**
*   $\sigma_t$: Total mass extinction coefficient ($\sigma_a + \sigma_s$) [S1, unofficial].
*   $\Delta x$: Distance between adjacent grid planes [S1, unofficial].
*   $\theta$: Angle between viewing direction and plane normal [G].

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation requires exponential operations. The source notes that ten million exponential operations per second are required to display smoke with corrected $\alpha$'s at 10 frames per second for a simulation with grid dimensions of $100 \times 100 \times 100$ [S1, unofficial].
*   **Boundary Conditions:** When $\theta$ approaches $90^\circ$ (grazing angle), $\cos(\theta)$ approaches $0$, causing $\alpha'$ to approach $1.0$ (fully opaque), which requires clamping or capping to avoid numerical instability [G].
*   **Plane Skipping:** If viewing planes are skipped, $\alpha$ must be adjusted proportionally to the change in distance to maintain visual consistency [S1, unofficial].

## Subtopics

### [Extinction Falloff](10700-extinction-falloff.md)
Extinction Falloff is a rendering shortcut that approximates the visual contribution of hundreds of additional scattering bounces by gently reducing the extinction coefficient with each real bounce. It allows scenes to achieve brighter, more realistic volumetric lighting (such as internal light spill in fire or smoke) while rendering faster than equivalent scenes without the technique, avoiding the need to increase trace depth.

## Operating characteristics
*   **Time Complexity:** The method requires exponential operations for each plane intersection. The source cites a requirement of ten million exponential operations per second for a $100 \times 100 \times 100$ grid at 10 fps [S1, unofficial].
*   **Memory Complexity:** Minimal, as it operates on scalar coefficients [G].
*   **Strengths:** Corrects visual artifacts caused by oblique viewing angles and variable grid spacing in planar smoke rendering [S1, unofficial].
*   **Limitations:** Computationally expensive due to the need for many exponential operations. Requires knowledge of the plane spacing and viewing geometry [G].
*   **When to use:** When rendering smoke or fire using planar alpha-blending techniques where view direction varies or grid spacing is non-uniform [S1, unofficial].
*   **When not to use:** When using continuous volume rendering methods that compute transmittance directly along the ray, or when performance is critical and visual accuracy of planar blending is not required [G].

## Background
The method originates from the visualization of fire dynamics simulation data, specifically within the Smokeview tool [S1, unofficial]. It addresses the need for accurate visual representation of smoke planes when viewed from arbitrary angles [S1, unofficial]. The technique is described in the technical reference guide for Smokeview (Version 6) [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
