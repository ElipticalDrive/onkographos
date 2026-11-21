# Single Scattering Models

Also called: Single Scattering Approximation, First-Order Scattering

## Overview
Single Scattering Models compute the radiance reaching the viewer by accounting for light that travels directly from a source to a volume and scatters exactly once toward the observer, while ignoring multiple scattering events and internal emission [G]. This method solves the problem of rendering volumetric effects like smoke or clouds efficiently by assuming that for low-density media, single scattering dominates the radiance contribution [G]. It sets itself apart from the [Absorption-only model](839-absorption-only-model.md) by including in-scattering, and from the full [Radiative Transfer Equation Formulation](1194-radiative-transfer-equation-formulation.md) by truncating the scattering series after the first order, thereby enabling faster computation at the cost of accuracy in dense media.

## Shared principles
All optical models in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [G]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [G]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [G].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [G].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray [G].
    *   *Out-scattering:* Light from the view ray reflected away [G].
In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [G].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [G].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [G]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [G].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [G].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [G].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [G]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [G].

## Subtopics

### [Single Scattering Model](1053-single-scattering-model.md)
The Single Scattering Model approximates light transport in participating media by accounting for light that travels directly from a source to a point in the volume and scatters exactly once toward the observer, while ignoring multiple scattering events and internal emission. It solves the problem of rendering volumetric effects like smoke or clouds efficiently by assuming that for low-density media, single scattering dominates the radiance contribution. It sets itself apart from the [Absorption-only model](839-absorption-only-model.md) by including in-scattering, and from the full [Radiative Transfer Equation Formulation](1194-radiative-transfer-equation-formulation.md) by truncating the scattering series after the first order, thereby enabling faster computation at the cost of accuracy in dense media. Choose it when rendering low-density participating media where single scattering dominates and computational efficiency is prioritized over full physical accuracy. See [Single Scattering Model](1053-single-scattering-model.md).

### [Gradient-guided single scattering approximation](11126-gradient-guided-single-scattering-approximation.md)
A GPU-based direct volume rendering method that approximates single scattering and shadowing on-the-fly for multiple intersecting volumetric datasets. It improves visual quality over local illumination models by using local gradients to guide scatter sample positions and in-scatter coefficients, while maintaining interactive frame rates. It builds on [Emission-Absorption Volume Rendering](4040-emission-absorption-volume-rendering.md) by adding a simplified single scattering term derived from [Max95]. Choose it for interactive rendering of multiple intersecting volumetric datasets where on-the-fly approximation is required. See [Gradient-guided single scattering approximation](11126-gradient-guided-single-scattering-approximation.md).

### [Sabella Cloud Scattering](2029-sabella-cloud-scattering.md)
Generates cloud-like images that illustrate internal structure, instead of contour surfaces of the data. It uses an accurate light-scattering model to model scattering effects of light in a cloud, and uses view rays as the basis for computing line integrals through the data to generate the color for each ray. Choose it for generating cloud-like images that emphasize internal structure over contour surfaces. See [Sabella Cloud Scattering](2029-sabella-cloud-scattering.md).

## Background
Sabella introduced a ray-casting method that uses an accurate light-scattering model to model the scattering effects of light in a cloud. Based on foundational work by Blinn, Kajiya, and Max, Sabella's model uses the view rays as the basis for computing line integrals through the data to generate the color for each ray [G].

## Sources
