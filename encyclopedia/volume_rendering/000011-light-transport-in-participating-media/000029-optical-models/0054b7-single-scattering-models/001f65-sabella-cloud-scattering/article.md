# Sabella Cloud Scattering

Also called: Sabella 88

## Overview
Sabella Cloud Scattering is a ray-casting method that generates cloud-like images illustrating internal structure by using an accurate light-scattering model to compute line integrals through volumetric data along view rays [S1, unofficial]. It distinguishes itself from sibling methods like the [Single Scattering Model](1053-single-scattering-model.md) by explicitly modeling the scattering effects of light within a cloud to reveal internal density variations, rather than relying on contour surfaces or simpler compositing techniques [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset representing cloud density or scalar fields; view rays defined by camera position and direction.
*   **Output:** A 2D image where each pixel color is the result of integrating radiance along the corresponding view ray.

**Data Structures**
The method operates on a volumetric grid. The specific layout of the scalar field (e.g., 3D array of floats) is not detailed in the sources, but it is sampled along the view ray [S1, unofficial].

**Steps**
1.  **Ray Casting:** For each pixel in the image, construct a view ray originating from the camera position and passing through the pixel center [S1, unofficial].
2.  **Integration:** Compute the line integral of the radiance transfer equation along the ray as it traverses the volumetric data [S1, unofficial].
3.  **Scattering Model:** Apply an accurate light-scattering model to calculate the contribution of light scattering within the cloud medium at each sample point along the ray [S1, unofficial]. This involves calculating in-scattering and out-scattering terms based on the local density and the scattering phase function.
4.  **Accumulation:** Accumulate the scattered radiance and transmittance along the ray. The process continues until the ray exits the volume or the accumulated opacity equals one [S1, unofficial].
5.  **Termination:** If the accumulated opacity reaches one, no further data points affect the ray's color, allowing for early termination optimization [S1, unofficial].

**Formulas**
The sources describe the method conceptually as using "line integrals" and an "accurate light-scattering model" but do not provide the explicit mathematical formulation of the scattering phase function or the specific integration scheme (e.g., midpoint vs. trapezoidal) [S1, unofficial]. The general form of the radiance accumulation along a ray $r(t)$ is:
$$ L(r) = \int_{t_{near}}^{t_{far}} T(t) \sigma_s(r(t)) L_{in}(r(t)) dt $$
where $T(t)$ is the transmittance, $\sigma_s$ is the scattering coefficient, and $L_{in}$ is the incoming radiance [G]. The specific implementation of $L_{in}$ and the phase function is attributed to Sabella's model but not detailed in the provided text [S1, unofficial].

**Parameters**
*   **Scattering Model Accuracy:** The method emphasizes using an "accurate" model, implying a higher computational cost for fidelity [S1, unofficial].
*   **Early Termination Threshold:** Opacity = 1.0 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Early Termination:** The optimization of stopping when opacity is one is unique to the feed-forward method described in the source context, but must be handled carefully to avoid artifacts if the density estimation is inaccurate [S1, unofficial].
*   **Internal Structure:** The method is designed to show internal structure; failure to sample densely enough may result in loss of this detail [S1, unofficial].

## Subtopics

### Volumetric Riley et al. Particle Scattering
A method for weather-field data based on particle scattering properties of constituent fields, translating input data to particle concentration and using volumetric transfer functions [S2]. It breaks light into three categories: unscattered light, central forward scattering (0 to $\theta/4$), and peripheral forward scattering ($\theta/4$ to $\theta/2$), each with respective transparencies [S2]. It calculates intensity attenuation as a function of particle extinction and concentration, and assumes the cloud is composed of water and ice particles with different phase functions applied to each [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The use of an "accurate light-scattering model" and line integrals suggests higher computational cost compared to simpler approximations.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Generates cloud-like images that illustrate internal structure, providing more detail than contour surfaces [S1, unofficial].
*   **Limitations:** The reliance on accurate scattering models and line integrals likely limits performance compared to faster, approximate methods.
*   **When to use:** When visual fidelity of internal cloud structure is prioritized over rendering speed.
*   **When not to use:** When real-time performance is required or when simple contour visualization is sufficient.

## Background
Sabella [Sabella 88] introduced this ray-casting method in 1988 [S1, unofficial]. It builds on the work of Blinn [Blinn 82], Kajiya [Kajiya 84], and Max [Max 86] [S1, unofficial]. The method was developed to address the need for visualizing internal structures in volumetric data, specifically for cloud rendering, moving beyond contour surface representations [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S2] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
