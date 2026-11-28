# Klassen's Spectral Atmospheric Method

Also called: Klassen's atmospheric model

## Overview
Klassen's Spectral Atmospheric Method is a physically-based rendering technique for simulating sky color and solar illumination in planetary atmospheres by explicitly treating spectral scattering. It solves the problem of inaccurate color reproduction in atmospheric rendering, where standard RGB-based approximations fail to capture the wavelength-dependent nature of Rayleigh and Mie scattering. It sets itself apart from its siblings by introducing a full spectral treatment (33 wavelengths) for the sky while retaining RGB efficiency for non-selective fog, and by distinguishing between single-scattering haze layers and multiple-scattering sky color contributions.

## How it works
The method computes the appearance of the sky and sun for an observer on the ground, and the appearance of fog for objects in the scene.

**Inputs:**
- Observer position: Ground level.
- Light source: Sun, treated as parallel rays due to large distance.
- Atmospheric layers:
  - Haze-free layer: Contains air molecules (Rayleigh scattering).
  - Haze-filled layer: Contains aerosols/particles (Mie scattering).
- Spectral data: Scattering coefficients and phase functions for 33 wavelengths ($\lambda$), sampled every 10 nm.
- Fog parameters: Density, extinction coefficient (RGB space).

**Outputs:**
- Sky color at each pixel (spectral).
- Sun disk color/intensity (spectral).
- Fog appearance for surface objects (RGB).

**Data Structures:**
- Spectral Sample Array: An array of 33 values representing intensity at wavelengths $\lambda_i = \lambda_{start} + i \cdot 10\text{nm}$.
- Layer Density Profile: Exponential decay of density with altitude $h$: $\rho(h) = \rho_0 e^{-h/H}$, where $H$ is the scale height.

**Steps:**

1.  **Define Geometry:**
    - Assume a flat earth approximation for local geometry and fog calculations.
    - Assume parallel light rays from the sun.
    - Define two spherical shells centered at the earth's center:
      - Inner shell (haze-free): Height $h_1$ to $h_2$.
      - Outer shell (haze-filled): Height $h_2$ to $h_3$.
    - *Note:* The sources describe "spherical layers" for the sky model but "flat earth" for the fog model. The planar model is noted to have large errors near the horizon, addressed by variants.

2.  **Compute Sky Color (Spectral):**
    - For each pixel direction $\vec{v}$ (view vector):
      - Calculate the optical depth $\tau(\lambda)$ along the ray through the atmosphere for each of the 33 wavelengths.
      - Compute single scattering contribution $L_{ss}(\lambda)$ in the haze-filled layer using the single-scattering approximation:
        $$ L_{ss}(\lambda) = \int_{ray} \sigma_s(\lambda, h) \Phi(\lambda, \theta) I_{sun}(\lambda) e^{-\tau(\lambda)} dh $$
        Where $\sigma_s$ is the scattering coefficient, $\Phi$ is the phase function, and $I_{sun}$ is solar spectral irradiance.
      - Compute multiple scattering contribution $L_{ms}(\lambda)$ for the sky color. The sources state that multiple scattering is not negligible for sky color and refers to Bohren [12] for the specific formulation. The exact formula is not provided in the primary source text, but it is added to the single scattering result.
      - Result: $L_{sky}(\lambda) = L_{ss}(\lambda) + L_{ms}(\lambda)$.

3.  **Compute Sun Disk (Spectral):**
    - For pixels intersecting the sun disk:
      - Calculate direct transmission through the atmosphere.
      - Apply spectral extinction based on optical depth.
      - Result: $I_{sun}(\lambda)$.

4.  **Compute Fog Appearance (RGB):**
    - Fog is assumed to be non-wavelength-selective.
    - Use standard RGB space.
    - Assume fog is lit only by the sun (no indirect skylight contribution to fog color in this basic model).
    - Assume no shadows cast by fog.
    - Use parallel light rays.
    - Approximate geometry as flat earth.
    - Compute fog color using standard volumetric rendering equations in RGB:
      $$ C_{fog} = C_{sun} \cdot (1 - e^{-\sigma_t d}) + C_{background} \cdot e^{-\sigma_t d} $$
      Where $\sigma_t$ is the extinction coefficient in RGB, $d$ is the distance through fog, and $C_{background}$ is the background color.

**Parameters:**
- Wavelengths: 33 samples, 10 nm spacing.
- Scattering types: Rayleigh (molecules), Mie (aerosols).
- Density distribution: Exponential with altitude.

**Edge Cases and Pitfalls:**
- **Horizon Error:** The planar atmosphere model results in large errors near the horizon. This is a known limitation of the base method.
- **Spectral vs RGB:** Using RGB for sky color is explicitly stated as wrong due to selective scattering. RGB is only used for fog.
- **Multiple Scattering:** Ignoring multiple scattering for sky color is incorrect; it must be included as per Bohren.

## Subtopics

### Concentric Spherical Layers Model
An extension of Klassen’s planar atmosphere model to concentric spherical layers to reduce large error near the horizon. It accounts for both air molecules (Rayleigh scattering) and aerosols (Mie scattering), with density distributions varying exponentially with altitude. It is used for generating outdoor scenes including buildings, calculating both sunlight and skylight. This variant addresses the geometric inaccuracies of the flat-earth approximation in the base method.

## Operating characteristics
- **Accuracy:** High spectral accuracy for sky color, avoiding RGB artifacts. Better geometric accuracy near the horizon in the spherical variant.
- **Performance:** Computationally expensive due to 33-wavelength sampling and integration of scattering. The RGB fog component is efficient.
- **Limitations:** The base planar model has significant errors near the horizon. The method assumes parallel sunlight and does not handle complex shadowing in fog.
- **Use Cases:** Realistic sky and sun rendering for outdoor scenes, driving simulators, and scenes requiring accurate atmospheric color.

## Background
The method was introduced by Klassen [44] in the context of atmospheric rendering. It is cited in surveys of participating media rendering techniques [S1]. The approach builds on earlier work in atmospheric optics, referencing Bohren [12] for multiple scattering in the sky. It was later extended by Kaneda et al. [42] to use concentric spherical layers to improve horizon accuracy.

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
