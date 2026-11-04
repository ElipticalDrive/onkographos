# X-ray simulation via ray integration

Also called: CT projection rendering, Digitally Reconstructed Radiograph, Simulated radiography

## Overview
X-ray simulation via ray integration generates a simulated radiograph (Digitally Reconstructed Radiograph, DR) by integrating attenuation along viewing rays through a volume, accounting for density between the source and film plane [S2, unofficial]. It differs from standard direct volume rendering by using the Beer-Lambert law on actual Hounsfield Unit (HU) values rather than arbitrary transfer functions for opacity, producing a single-energy, projection-style image without scatter or beam hardening [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume where voxel values $f$ represent X-ray absorption density (Hounsfield Units) reconstructed by Computed Tomography [S2, unofficial].
- Input: Camera parameters defining the ray origin (source) and direction.
- Input: Exposure control $E$, which scales the final intensity [S1, unofficial].
- Output: A 2D image representing the simulated X-ray projection.

**Data Structures:**
- Volume: 3D grid of scalar values (HU).
- Ray: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Step size: Uniform physical step size in millimeters, $u_{mm}$ [S1, unofficial].

**Parameters:**
- $\mu_{water}$: Linear attenuation coefficient of water at ~70 keV, $0.0193 \, \text{mm}^{-1}$ [S1, unofficial].
- $E$: Exposure control (dimensionless scaling factor for tube output and detector gain) [S1, unofficial].

**Formulas:**
1.  **Linear Attenuation Coefficient:** Convert HU to linear attenuation coefficient $\mu$ using the definition of the Hounsfield scale [S1, unofficial]:
    $$ \mu = \mu_{water} \times \left(1 + \frac{HU}{1000}\right) $$
2.  **Ray Integration:** Integrate $\mu$ along the ray path $l$ from source to detector:
    $$ A = \int \mu \, dl \approx \sum_{i} \mu_i \cdot u_{mm} $$
    where $u_{mm}$ is the step size in millimeters [S1, unofficial].
3.  **Intensity Calculation:** Apply the Beer-Lambert law to determine transmitted intensity $I$, scaled by exposure $E$ [S1, unofficial]:
    $$ I = E \cdot I_0 \cdot e^{-A} $$
    where $I_0$ is the initial intensity (typically normalized to 1 or max value).

**Steps:**
1.  For each pixel in the output image, cast a ray from the X-ray source through the volume.
2.  Sample the volume along the ray at uniform intervals.
3.  For each sample, retrieve the Hounsfield Unit value $HU$.
4.  Compute the linear attenuation coefficient $\mu$ for the sample using $\mu = \mu_{water} \times (1 + HU/1000)$ [S1, unofficial].
5.  Accumulate the optical depth $A$ by summing $\mu \times u_{mm}$ for all steps along the ray [S1, unofficial].
6.  Compute the final pixel intensity $I = E \cdot e^{-A}$ (assuming $I_0=1$) [S1, unofficial].

**Pitfalls:**
- The integral must explicitly use the millimeter step size ($u_{mm}$). Omitting the multiplication by $u_{mm}$ breaks the physics, as the exponent becomes dimensionless but incorrect, leading to wrong attenuation values [S1, unofficial].
- Threshold controls and opacity-per-step sliders are irrelevant and should be hidden, as they are volume-compositing factors not used in attenuation-based radiograph mode [S1, unofficial].
- The method assumes single energy, no scatter, no beam hardening, and no detector response [S1, unofficial].
- Exposure is a control for tube output and detector gain, NOT a dose in milligrays; no dose is computed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simulate_xray(volume, camera, exposure, mu_water=0.0193):
    """
    Simulate an X-ray projection (DR) from a CT volume.
    
    Args:
        volume: 3D array of Hounsfield Units (HU).
        camera: Object with method get_ray(u, v) returning (origin, direction).
        exposure: Float, scaling factor for intensity.
        mu_water: Float, attenuation coefficient of water in mm^-1.
        
    Returns:
        2D image array.
    """
    height, width = camera.resolution
    image = zeros((height, width))
    
    # Step size in millimeters. Assumed uniform and known from volume spacing.
    # In a real implementation, this would be derived from the volume's physical spacing.
    # Here we assume a uniform step size u_mm is available or passed.
    u_mm = volume.spacing_x # Assuming isotropic or using x-spacing as representative
    
    for y in range(height):
        for x in range(width):
            origin, direction = camera.get_ray(x, y)
            
            # Ray marching parameters
            t_min = 0.0
            t_max = camera.far_plane # Or distance to exit volume
            
            # Simple ray-box intersection or fixed range for demonstration
            # In practice, clip ray to volume bounds
            t_start, t_end = clip_ray_to_volume(origin, direction, volume.bounds)
            
            optical_depth = 0.0
            t = t_start
            while t < t_end:
                # Sample volume at current position
                pos = origin + direction * t
                hu = sample_volume(volume, pos)
                
                # Convert HU to linear attenuation coefficient
                mu = mu_water * (1.0 + hu / 1000.0)
                
                # Accumulate optical depth with physical step size
                optical_depth += mu * u_mm
                
                # Advance ray by physical step size
                t += u_mm
            
            # Apply Beer-Lambert law and exposure
            # I = I0 * exp(-optical_depth) * exposure
            # Assuming I0 = 1.0
            intensity = exposure * exp(-optical_depth)
            
            image[y, x] = intensity
            
    return image
```

## Subtopics

### Prism Model
- **What it is:** A sibling optical model for volume rendering.
- **Difference:** Uses a different geometric approximation for light transport compared to the absorption-only ray integration.
- **When to choose:** When simulating different scattering or geometric properties than simple absorption.
- **Link:** [Prism Model](3153-prism-model.md)

## Operating characteristics
- **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of pixels and $R$ is the number of samples per ray. Linear with respect to resolution and ray length.
- **Memory Complexity:** $O(V)$, where $V$ is the volume size, for storing the CT data.
- **Performance:** Measured on a synthetic 700 HU sphere at 390x844 resolution. Exposure scaling is monotonic (e.g., x0.4, x1, x3) as required by the exponential function [S1, unofficial].
- **Strengths:** Physically based on HU values; fast for single-energy projection; no need for complex transfer functions.
- **Limitations:** Single energy assumption; no scatter, beam hardening, or detector response modeling; not suitable for multi-energy or spectral CT simulation [S1, unofficial].
- **When to use:** Generating simulated radiographs from CT data for visualization or comparison with real X-rays.
- **When not to use:** When scatter, beam hardening, or multi-energy effects are significant.

## Background
- **Origin:** The method is based on the Beer-Lambert law and the definition of Hounsfield Units in Computed Tomography [S1, S2, unofficial].
- **History:** Digitally Reconstructed Radiographs (DR) have been used in medical imaging to simulate X-ray projections from 3D CT data [S2, unofficial].
- **Usage:** Used in medical imaging for treatment planning, visualization, and education [S1, S2, unofficial].

## Sources
[S1] feat(body-exposure): render a DICOM series in 3D, not a description of one, https://github.com/rizkyazhar486/Panacea/pull/1585, fetched 2026-10-26 (unofficial)
[S2] Optical Models for Direct Volume Rendering - DocsLib, https://docslib.org/doc/3914352/optical-models-for-direct-volume-rendering, fetched 2026-10-27 (unofficial)
