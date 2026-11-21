# Holographic Reconstruction System Setup

Also called: Experimental setup for holographic reconstruction, Optical setup for holographic reconstruction, Phase-Only Holographic Display

## Overview
This article describes the physical optical apparatus required to reconstruct a complex wavefield computed numerically (see [Digital Hologram Numerical Reconstruction](13798-digital-hologram-numerical-reconstruction.md)) into a visible image. It solves the problem of converting digital hologram matrices into optical fields using spatial light modulators (SLMs), handling the constraints of phase-only or amplitude-only modulation, and filtering out diffraction artifacts. It differs from pure numerical reconstruction by introducing physical components (lasers, lenses, SLMs, cameras) and their calibration.

## How it works
The system converts a digital hologram into a physical wavefront using coherent illumination, spatial light modulation, and optical filtering.

**Inputs and Outputs:**
*   **Input:** Digital hologram matrix (complex or real-valued), system parameters (wavelength, pixel pitch, distance).
*   **Output:** Reconstructed wavefront captured by a camera or viewed by an eye.

**Data Structures:**
*   **Hologram Matrix:** 2D array of complex numbers $U_h(x,y)$ or real-valued intensity/phase maps.
*   **SLM Pixel Map:** 2D array of values mapped to SLM physical pixels.

**Steps:**

1.  **Illumination Preparation:**
    *   Filter the laser beam using a pinhole to remove spatial noise.
    *   Collimate the beam using a lens to ensure plane wave illumination on the SLM [S1].
    *   If speckle reduction is required, rotate a holographic diffuser at the focused spot of the laser beams [S2].

2.  **Hologram Encoding (if Phase-Only SLM):**
    *   If the SLM is phase-only but the hologram is complex-valued, encode the complex field into real values using the Burch encoding method [S2].
    *   For off-axis holograms, set the grating period to 0.25 of the maximum period to avoid unwanted noise [S2].

3.  **SLM Modulation:**
    *   Upload the encoded hologram to the SLM.
    *   If using an LCoS SLM with small pixel pitch, upsample by putting the same pixel value in 2x2 nearest pixels to minimize pixel crosstalk [S2].
    *   Calibrate the amplitude response: measure output intensity for each input pixel value, as modulated intensity depends nonlinearly on assigned values [S2].

4.  **Optical Propagation and Filtering:**
    *   Illuminate the SLM with the collimated laser beam.
    *   Form the Fourier plane using a first lens group.
    *   Block the zeroth-order and higher-order diffraction terms using an adjustable 2D slit at the Fourier plane [S2].

5.  **Reconstruction and Capture:**
    *   The reconstructed wavefront propagates to the target plane (distance $z$).
    *   Capture the wavefront using a digital camera (e.g., Canon 60D) positioned at the shooting distance [S1].
    *   Alternatively, view directly with the human eye or a near-eye display system.

**Parameters:**
*   **Wavelength ($\lambda$):** Must match the recording or intended reconstruction wavelength. Common values: 638 nm (red), 515 nm (green), 460 nm (blue) [S2]. Single wavelength example: 532 nm [S1].
*   **Pixel Pitch ($\Delta x$):** Determines spatial frequency sampling. Example: 3.6 $\mu$m for IRIS-U62 LCoS, effectively 7.2 $\mu$m when upsampled [S2]. Example: 3.74 $\mu$m [S1].
*   **Reconstruction Distance ($z$):** Axial distance from SLM to camera/eye. Examples: 155, 160, 165, 170 mm [S1]. Minimum required: > 1.7 mm to avoid excessive defocus blur from pixel pitch diffraction [S2].
*   **Grating Period:** For off-axis holograms, set to 0.25 of maximum period [S2].

**Edge Cases and Pitfalls:**
*   **Pixel Crosstalk:** Small pixel pitch on LCoS SLMs causes crosstalk. Mitigation: upsample by 2x2 [S2].
*   **Diffraction Orders:** Zeroth and higher orders create noise. Mitigation: use a 2D slit at the Fourier plane [S2].
*   **Speckle:** Caused by laser coherence. Mitigation: rotate holographic diffuser [S2].
*   **Defocus Blur:** Pixel pitch diffraction causes blur. At 2 mm propagation, dispersion diameters are 25 pixels (red), 20 (green), 18 (blue) [S2]. If max defocus blur diameter is 15 pixels, propagation must be > 1.7 mm [S2].
*   **Nonlinear Response:** SLM intensity response is nonlinear. Calibration is required [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_holographic_reconstruction(laser_wavelength, slm_pixel_pitch, slm_resolution, reconstruction_distance):
    """
    Configures the physical parameters for holographic reconstruction.
    """
    # 1. Illumination Setup
    pinhole_filter = PinholeFilter()
    collimating_lens = Lens(focal_length=calculate_focal_length(laser_wavelength, slm_pixel_pitch))
    
    # 2. SLM Configuration
    slm = LCOS_SLM(model="IRIS-U62", native_pitch=slm_pixel_pitch, resolution=slm_resolution)
    
    # Upsample if necessary to reduce crosstalk (example for 2x2)
    slm.upsample_factor = 2
    
    # 3. Calibration (Amplitude)
    # Measure output intensity for each input pixel value
    calibration_curve = measure_slm_response(slm)
    
    # 4. Hologram Encoding (if complex to phase-only)
    # Use Burch encoding for off-axis hologram
    grating_period_ratio = 0.25
    encoded_hologram = burch_encode(complex_hologram, grating_period_ratio)
    
    # 5. Optical Filtering Setup
    fourier_lens = Lens(focal_length=calculate_focal_length(laser_wavelength, slm_pixel_pitch))
    slit_filter = Adjustable2DSlit()
    
    # 6. Capture Setup
    camera = DigitalCamera(sensor_size=calculate_sensor_size(reconstruction_distance, slm_pixel_pitch))
    camera.distance = reconstruction_distance
    
    return {
        "illumination": (pinhole_filter, collimating_lens),
        "slm": slm,
        "encoding": encoded_hologram,
        "filtering": (fourier_lens, slit_filter),
        "capture": camera
    }

def calculate_focal_length(wavelength, pixel_pitch):
    """
    Placeholder for lens selection based on diffraction limits.
    """
    # Not given in the sources.
    return 100.0 # mm

def measure_slm_response(slm):
    """
    Measures the nonlinear intensity response of the SLM.
    """
    response = []
    for pixel_value in range(256):
        slm.set_pixel_value(pixel_value)
        intensity = slm.measure_output_intensity()
        response.append(intensity)
    return response

def burch_encode(hologram, grating_period_ratio):
    """
    Encodes complex hologram into real values using Burch method.
    """
    # Not given in the sources.
    return hologram.real # Placeholder
```

## Subtopics

### Calibration via Coefficient Adjustment
A user-driven calibration method for correcting aberrations in holographic near-eye displays. The display presents a single object point $o_i$ to the user, represented in the hologram as $f_z(\rho, \theta)_{o_i}$. The user adjusts the coefficients $a_{jo}$ of $f_z(\rho, \theta)_{o_i}$ until the most tightly focused spot is seen. This is repeated for additional object points. For true 3D holograms, calibration is performed for a small number of focal positions (e.g., four) and coefficients are interpolated over the complete focal range. For highly aberrated systems, 15 measured points are sufficient; for displays without significant aberrations, a single object point at the center is sufficient. It is impractical to measure for all millions of object points, so interpolation is used [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In one experiment, shooting distances of 155, 160, 165, and 170 mm were used to achieve 3D effects [S1].
*   **Scaling:** Propagation distance must be adjusted to avoid defocus blur from pixel pitch diffraction. Minimum distance > 1.7 mm for 2 mm depth range [S2].
*   **Strengths:** Allows physical reconstruction of complex wavefields; calibration can correct for system-specific aberrations.
*   **Limitations:** Requires precise optical alignment; sensitive to laser coherence (speckle); nonlinear SLM response requires calibration; diffraction orders must be filtered.
*   **When to Use:** When physical holographic reconstruction is needed for volumetric displays or near-eye displays.
*   **When Not to Use:** When purely numerical simulation is sufficient.

## Background
Holographic reconstruction systems have been developed for various applications, including 2D-to-3D rendering and near-eye displays for VR/AR. Early systems used phase-only SLMs with Gaea-2 systems [S1]. More recent developments include diffraction-engineered holography using LCoS SLMs to overcome depth representation limits [S2]. Calibration techniques using user feedback have been proposed for correcting aberrations in holographic near-eye displays [S3].

## Sources
[S1] Holographic 3D Display Using Depth Maps Generated by 2D-to-3D Rendering Approach, https://doi.org/10.3390/app11219889, fetched 2026-10-26
[S2] Diffraction-engineered holography: Beyond the depth representation limit of holographic displays, https://www.nature.com/articles/s41467-022-33728-5, fetched 2026-10-29
[S3] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
