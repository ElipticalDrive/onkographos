# X-ray simulation via ray integration

Also called: CT projection rendering, Digitally Reconstructed Radiograph, Simulated radiography

## Overview
X-ray simulation via ray integration generates a Digitally Reconstructed Radiograph (DR) by integrating the linear attenuation coefficient along viewing rays from a source to a film plane, producing an X-ray-like projection image [S2, unofficial]. Unlike standard direct volume rendering which uses transfer functions to map scalar values to opacity for compositing, this method assumes a single energy level with no scatter, no beam hardening, and no detector response, relying strictly on the Beer-Lambert law to simulate physical attenuation [S1, unofficial]. It is distinguished from its siblings by using actual Hounsfield Unit (HU) values directly in the attenuation calculation rather than arbitrary opacity mappings [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset where the scalar field $f$ represents X-ray absorption density reconstructed by Computed Tomography (i.e., Hounsfield Units) [S2, unofficial].
*   **Input:** Camera parameters defining the source position $s=0$ and the film plane orientation.
*   **Input:** Exposure control, which acts as a combined multiplier for tube output and detector gain [S1, unofficial].
*   **Output:** A 2D simulated radiograph image.

**Data Structures:**
*   **Volume:** A 3D grid of floating-point values representing HU.
*   **Ray:** Defined by origin (source) and direction vector.
*   **Step Size:** A uniform constant `uMmPerLangkah` representing the integration step size in millimeters [S1, unofficial].

**Formulas:**
1.  **Linear Attenuation Coefficient ($\mu$):**
    Derived from the Hounsfield scale definition:
    $$ \mu = \mu_{water} \times \left(1 + \frac{HU}{1000}\right) $$
    Where $\mu_{water} = 0.0193 \, \text{mm}^{-1}$ at approximately 70 keV [S1, unofficial].

2.  **Attenuation Integral:**
    The optical depth $\tau$ along a ray is the integral of $\mu$ over the path length $l$:
    $$ \tau = \int_{0}^{L} \mu(\mathbf{r}(l)) \, dl \approx \sum_{i} \mu_i \cdot \Delta l $$
    Where $\Delta l$ is the step size in millimeters (`uMmPerLangkah`) [S1, unofficial].

3.  **Image Intensity ($I$):**
    Using the Beer-Lambert law:
    $$ I = I_0 \cdot \text{Exposure} \cdot \exp(-\tau) $$
    Where $I_0$ is the initial intensity (often normalized to 1) and Exposure is the scalar control [S1, unofficial].

**Parameters:**
*   **$\mu_{water}$:** $0.0193 \, \text{mm}^{-1}$ [S1, unofficial].
*   **Exposure:** A scalar multiplier. It is NOT a dose in milligrays; no dose is computed [S1, unofficial].
*   **Step Size:** Must be in millimeters. The multiplication by `uMmPerLangkah` is critical; removing it breaks the physics even if the uniform variable exists [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Step Size Integration:** The integral must explicitly multiply the attenuation coefficient by the millimeter step size. A common implementation error is declaring the step size uniform but failing to use it in the summation, which causes incorrect attenuation values [S1, unofficial].
*   **Irrelevant Controls:** Threshold controls and opacity-per-step sliders are irrelevant and should be hidden. Thresholds do not apply to attenuation-based calculation, and opacity-per-step is a volume-compositing factor not used in this single-pass integration [S1, unofficial].
*   **Physical Assumptions:** The method assumes single energy. It does not account for scatter, beam hardening, or detector response characteristics [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simulate_xray_projection(volume, camera, exposure, mu_water=0.0193, step_size_mm=1.0):
    """
    Simulates a radiograph by integrating attenuation along rays.
    
    Args:
        volume: 3D array of Hounsfield Units (HU).
        camera: Object with 'ray_for_pixel(u, v)' method returning (origin, direction).
        exposure: Scalar multiplier for tube output and detector gain.
        mu_water: Linear attenuation coefficient of water in mm^-1.
        step_size_mm: Integration step size in millimeters.
        
    Returns:
        2D image array.
    """
    height, width = camera.image_size
    image = zeros((height, width))
    
    for y in range(height):
        for x in range(width):
            origin, direction = camera.ray_for_pixel(x, y)
            
            # Initialize Beer-Lambert integral
            integral_mu_dl = 0.0
            
            # Traverse the volume along the ray
            # Note: Implementation of ray marching bounds depends on volume geometry.
            # This loop assumes a simple bounding box intersection or pre-computed ray steps.
            for step in range(num_steps):
                point = origin + direction * (step * step_size_mm)
                
                # Check if point is inside volume bounds
                if not volume.contains(point):
                    break
                
                # Sample HU value at current point
                hu = volume.sample(point)
                
                # Calculate linear attenuation coefficient from HU
                # mu = mu_water * (1 + HU / 1000)
                mu = mu_water * (1.0 + hu / 1000.0)
                
                # Accumulate integral: sum(mu * dl)
                # Critical: Must multiply by step_size_mm
                integral_mu_dl += mu * step_size_mm
            
            # Apply Beer-Lambert law
            # I = I0 * Exposure * exp(-integral)
            # Assuming I0 = 1.0
            intensity = exposure * exp(-integral_mu_dl)
            
            image[y, x] = intensity
            
    return image
```

## Subtopics

### [Hardware-Accelerated Simulated Radiography (Non-Emissive)](10237-hardware-accelerated-simulated-radiography-non-emissive.md)
A GPU-based technique that renders geometric cells (hexahedra or tetrahedra) as separate layers using hardware rasterization to accumulate attenuation in a P-buffer. It handles unsorted, overlapping geometry by iteratively rendering non-overlapping subsets, minimizing expensive synchronization copies compared to sorted front-to-back methods.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of pixels and $R$ is the number of steps per ray (or number of cells intersected in hardware-accelerated variants).
*   **Memory Complexity:** $O(V)$ for the volume data, plus $O(P)$ for the output image, where $V$ is volume size and $P$ is pixel count.
*   **Performance:** Measured on a synthetic 700 HU sphere at 390x844 resolution, the method produces monotonic intensity changes with exposure (e.g., exposure x0.4 yields mean 7.7, x1 yields 14.6, x3 yields 22.2) [S1, unofficial].
*   **Strengths:** Physically accurate projection based on actual HU values; simple implementation; no need for complex transfer function tuning [S1, unofficial].
*   **Limitations:** Assumes single energy; ignores scatter and beam hardening; does not model detector response [S1, unofficial].
*   **When to use:** When generating standard X-ray projections from CT data for comparison with real radiographs or for basic visualization.
*   **When not to use:** When high-fidelity spectral effects, scatter, or detector physics are required [S1, unofficial].

## Background
The method originates from the application of optical models for direct volume rendering, specifically the absorption-only model where the transfer function is set to the identity function for X-ray absorption density [S2, unofficial]. It leverages the definition of the Hounsfield scale to derive linear attenuation coefficients, allowing for the simulation of X-ray images from arbitrary directions [S2, unofficial]. The specific implementation details, including the use of `mu_water = 0.0193 /mm` at ~70 keV and the handling of exposure as a combined tube/detector gain factor, are documented in recent software development efforts for DICOM volume rendering [S1, unofficial].

## Sources
[S1] feat(body-exposure): render a DICOM series in 3D, not a description of one, https://github.com/rizkyazhar486/Panacea/pull/1585, fetched 2026-10-26 (unofficial)
[S2] Optical Models for Direct Volume Rendering - DocsLib, https://docslib.org/doc/3914352/optical-models-for-direct-volume-rendering, fetched 2026-10-27 (unofficial)
