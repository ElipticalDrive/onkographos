# Direct Light Contribution Estimation

Also called: Direct Light Contribution

## Overview
Direct Light Contribution Estimation computes the direct illumination intensity at a specific point in a participating medium by combining local high-resolution occlusion data with global attenuation data [S1]. It solves the problem of balancing computational efficiency with visual fidelity in volume rendering, where calculating exact transmittance for every voxel is prohibitively expensive [S1]. It differs from standard global light propagation methods by explicitly separating the near-field occlusion (handled via local piecewise integration) from the far-field attenuation (handled via a Sparse Voxel Representation), ensuring that local shadows remain sharp while global lighting remains consistent [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Light source intensity $I_0$, local opacity $\alpha_l(s_0)$ at the current voxel position $s_0$, and global opacity $\alpha_g(s_1)$ representing attenuation from a distant point $s_1$ to the light source [S1].
*   **Output:** Direct light intensity $I_d(s_0)$ at position $s_0$ [S1].

**Data Structures:**
*   **Local Opacity ($\alpha_l$):** Derived from local piecewise integration [S1]. This represents the opacity of the ray segment from the viewpoint to the current voxel $s_0$ [S1]. It is stored at the highest resolution available for the local geometry [S1].
*   **Global Opacity ($\alpha_g$):** Interpolated from a Sparse Voxel Representation (SVR) [S1]. This represents the accumulated opacity along the ray segment from a reference point $s_1$ to the light source [S1].

**Steps:**
1.  Retrieve the local opacity $\alpha_l(s_0)$ for the current voxel $s_0$ from the previously computed local piecewise integration results [S1].
2.  Identify the reference point $s_1$ along the ray towards the light source [S1].
3.  Interpolate the global opacity $\alpha_g(s_1)$ from the SVR for the segment extending from $s_1$ to the light source [S1].
4.  Calculate the direct light contribution using the multiplicative attenuation model [S1]:

$$I_d(s_0) = I_0 \cdot (1 - \alpha_g(s_1)) \cdot (1 - \alpha_l(s_0))$$

Where:
*   $I_d(s_0)$ is the resulting direct light intensity at $s_0$ [S1].
*   $I_0$ is the user-defined light source intensity [S1].
*   $\alpha_g(s_1)$ is the global opacity (attenuation) from $s_1$ to the light [S1].
*   $\alpha_l(s_0)$ is the local opacity from the viewpoint to $s_0$ [S1].

**Pitfalls:**
*   **Local Occlusion Accuracy:** Using only $\alpha_g$ for the entire ray path reduces visual quality because the SVR typically has lower resolution than the local piecewise integration [S1]. Neglecting $\alpha_l(s_0)$ results in blurred or incorrect local shadows [S1].
*   **Boundary Definition:** The choice of $s_1$ determines where the local piecewise integration ends and the SVR interpolation begins [S1].

## Subtopics

### [Direct Illumination Derivative (Spatial Form)](7662-direct-illumination-derivative-spatial-form.md)
This method computes the derivative of the direct illumination integral with respect to scene parameters $\pi$ by differentiating the surface integral directly over the evolving surface geometry. It is distinguished from the Material-Form variant by integrating over the physical surface domain $L(\pi)$, which depends on $\pi$, thereby requiring the estimation of a boundary term that accounts for the motion of the integration domain's edge.

### [Incident Light Field Simulation](8137-incident-light-field-simulation.md)
Incident Light Field Simulation estimates the total incident radiance $L_i$ at a surface point by combining direct illumination from light sources, attenuated direct light, and indirect illumination reflected from other surfaces. It differs from sibling methods like [Direct Illumination Derivative (Spatial Form)](7662-direct-illumination-derivative-spatial-form.md) by explicitly modeling indirect light contributions via neural networks or spherical harmonics, rather than relying solely on spatial derivatives of direct light.

### Sun Light Estimation
**Definition:** Estimation of lighting from a delta light (sun) using multiple importance sampling (MIS) [S2, unofficial]. Since the light is a delta function, the light PDF is always 1.0 and is not taken into account in the calculation [S2, unofficial].
**Steps:**
1.  Find the phase function PDF using the sun light direction and the ray direction [S2, unofficial].
2.  Calculate the transmittance to the light source [S2, unofficial].
3.  Multiply the phase function PDF, transmittance, and the sun's power [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is efficient because it avoids full path tracing for direct light [S1]. It requires one interpolation from the SVR and one lookup from local data per voxel [S1].
*   **Memory Complexity:** Requires storage for the SVR (global opacity) and local integration results [S1].
*   **Strengths:** Improves final quality compared to using only global opacity by ensuring local occlusion is calculated at the highest possible resolution [S1].
*   **Limitations:** Requires pre-computation of local piecewise integration and maintenance of the SVR [S1].
*   **When to use:** When high-fidelity local shadows are required in volume rendering without the cost of full global illumination [S1].

## Background
The method originates from the work on Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration [S1]. It addresses the limitations of earlier global light propagation techniques by introducing a hybrid approach that leverages local piecewise integration for near-field accuracy [S1].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
