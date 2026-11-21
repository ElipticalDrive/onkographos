# Absorption Adjustment

Also called: Opacity Adjustment

## Overview
Absorption Adjustment corrects the opacity parameter $\alpha$ used in planar smoke rendering to account for non-orthogonal viewing angles and variable plane spacing. It ensures that the visual density of smoke planes remains consistent regardless of the camera's orientation or the simulation grid's resolution, preventing artifacts where smoke appears too thin or too thick when viewed from oblique angles. It differs from [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md) by operating on pre-computed planar alpha values rather than computing transmittance from first principles for every ray, and from [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) by being a geometric correction factor applied to discrete planes rather than a continuous medium model.

## How it works
**Inputs and Outputs:**
*   Input: Base absorption coefficient $\alpha_0$ (derived from the medium's extinction coefficient $\sigma_t$ and plane spacing $\Delta x$), viewing direction vector $\mathbf{v}$, and plane normal vector $\mathbf{n}$.
*   Output: Adjusted absorption parameter $\alpha'$ suitable for blending.

**Data Structures:**
The method operates on scalar values. In the context of the source, $\alpha$ is a float in the range $[0.0, 1.0]$, where $0.0$ is completely transparent and $1.0$ is completely opaque [S1, unofficial].

**Steps:**
1.  **Calculate Geometric Path Length Correction:** Determine the angle $\theta$ between the viewing direction $\mathbf{v}$ and the plane normal $\mathbf{n}$. The effective distance light travels through the plane is scaled by $1/\cos(\theta)$.
2.  **Adjust Absorption Coefficient:** If the viewing direction is not aligned with the axis orthogonal to the viewing planes, scale the absorption. The source notes that $\alpha$ must be adjusted for view directions not aligned with the axis orthogonal to the viewing planes [S1, unofficial].
3.  **Handle Variable Spacing:** If the distance between adjacent smoke planes changes or viewing planes are skipped, adjust $\alpha$ proportionally to the change in distance. The absorption coefficient is adjusted when the distance between adjacent smoke planes changes [S1, unofficial].
4.  **Compute Final Alpha:** Apply the Beer-Lambert law using the adjusted coefficient to determine the final opacity for blending.

**Formulas:**
The base absorption parameter $\alpha$ is related to the total mass extinction coefficient $\sigma_t$ and the distance between grid planes $\Delta x$ by the Beer-Lambert law:
$$ \alpha = 1 - e^{-\sigma_t \Delta x} $$
When adjusting for view direction, the effective path length increases. If $\theta$ is the angle between the view vector and the plane normal, the effective thickness is $\Delta x / \cos(\theta)$. The adjusted absorption $\alpha'$ becomes:
$$ \alpha' = 1 - e^{-\sigma_t \frac{\Delta x}{\cos(\theta)}} $$
If planes are skipped or spacing changes to $\Delta x'$, the coefficient is scaled accordingly [S1, unofficial].

**Parameters:**
*   $\sigma_t$: Total mass extinction coefficient ($\sigma_a + \sigma_s$).
*   $\Delta x$: Distance between adjacent grid planes.
*   $\theta$: Angle between viewing direction and plane normal.

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation requires exponential operations. The source notes that ten million exponential operations per second are required to display smoke with corrected $\alpha$'s at 10 frames per second for a simulation with grid dimensions of $100 \times 100 \times 100$ [S1, unofficial].
*   **Boundary Conditions:** When $\theta$ approaches $90^\circ$ (grazing angle), $\cos(\theta)$ approaches 0, causing $\alpha'$ to approach 1.0 (fully opaque). This must be handled to avoid numerical instability or incorrect blending.
*   **Plane Skipping:** If viewing planes are skipped, the distance $\Delta x$ effectively increases, requiring a corresponding increase in $\alpha$ to maintain visual consistency [S1, unofficial].

## Subtopics

### [Extinction Falloff](10700-extinction-falloff.md)
Extinction Falloff is a rendering shortcut that approximates the visual contribution of hundreds of additional scattering bounces by gently reducing the extinction coefficient with each real bounce. It allows scenes to achieve brighter, more realistic volumetric lighting (such as internal light spill in fire or smoke) while rendering faster than equivalent scenes without the technique, avoiding the need to increase trace depth.

## Operating characteristics
*   **Time Complexity:** The method requires exponential operations for each plane intersection. The source cites a requirement of ten million exponential operations per second for a $100 \times 100 \times 100$ grid at 10 fps [S1, unofficial].
*   **Memory Complexity:** Minimal, as it operates on scalar coefficients.
*   **Strengths:** Corrects visual artifacts caused by oblique viewing angles and variable grid spacing in planar smoke rendering.
*   **Limitations:** Computationally expensive due to the need for many exponential operations. Requires knowledge of the plane spacing and viewing geometry.
*   **When to use:** When rendering smoke or fire using planar alpha-blending techniques where view direction varies or grid spacing is non-uniform.
*   **When not to use:** When using continuous volume rendering methods that compute transmittance directly along the ray, or when performance is critical and visual accuracy of planar blending is not required.

## Background
The method originates from the visualization of fire dynamics simulation data, specifically within the Smokeview tool. It addresses the need for accurate visual representation of smoke planes when viewed from arbitrary angles. The technique is described in the technical reference guide for Smokeview (Version 6) [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
