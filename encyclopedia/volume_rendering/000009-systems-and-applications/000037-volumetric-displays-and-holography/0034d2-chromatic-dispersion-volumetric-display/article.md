# Chromatic Dispersion Volumetric Display

Also called: DOE-based real-space volumetric display, Polychromatic Diffractive Lens Volumetric Display, Polychromatic volumetric display, Volumetric 3D display using diffractive lens, Wavelength-multiplexed free-space volumetric display

## Overview
A method for realizing a solid-state volumetric display in real space by using the chromatic dispersion properties of a diffractive lens to image a series of planar patterns from a projector [S1]. Each image plane's position is defined by a distinct wavelength of light from a narrowband polychromatic source [S1]. Unlike physically swept displays, this method produces an image within the user’s reach, has no moving parts, and can produce significantly larger images compared to laser plasma approaches [S1]. Unlike holographic solutions, this method avoids concerns of mechanical reliability and high-power lasers [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A sequence of $N$ 2D images (one per wavelength), a polychromatic light source with $N$ discrete narrowband wavelengths, and a diffractive optical element (DOE) lens.
*   **Output:** A 3D volume of $N$ discrete, wavelength-dependent image planes floating in free space.

**Data Structures**
*   **LED Array:** A grid of $N$ resonant cavity light-emitting diodes (RCLEDs). Each LED $i$ has a peak wavelength $\lambda_i$, a linewidth (FWHM) $\Delta \lambda_i$, and a power output $P_i$.
*   **Image Buffer:** $N$ 8-bit grayscale images, each corresponding to the spatial pattern for wavelength $\lambda_i$.
*   **DOE Profile:** A continuous kinoform surface profile defined by depth $z(r)$, where $r$ is the radial distance from the optical axis.

**Optical Configuration**
The system uses a 4f optical configuration [S1]:
1.  LED Array -> Lens Triplet -> Projector -> Diffuser -> Fresnel Lens -> DOE Lens.
2.  The LED array is coupled into the projector via a lens triplet to concentrate and collimate light [S1].
3.  The projector displays the image on a Digital Micro-mirror Device (DMD) [S1].
4.  Light exits the projector (inverted image) and hits a ground glass diffuser [S1].
5.  A Fresnel lens is placed one focal length away from the diffuser to collimate the light towards the DOE [S1].
6.  The DOE lens is placed at a distance $d_{F-DOE}$ from the Fresnel lens and focuses the image into free space [S1].

**Numbered Steps**
1.  **LED Selection and Characterization:** Select $N$ LEDs from the array such that their peak wavelengths are uniformly spaced across the desired range, excluding LEDs with wide linewidths, multiple peaks, or excessive power variation [S1].
2.  **Image Preparation:** Prepare $N$ 2D images. Apply image processing to reduce off-axis distortion and compensate for wavelength-dependent magnification via scaling [S1].
3.  **Upload Images:** Upload the $N$ 8-bit images to the projector's internal memory [S1].
4.  **Synchronization Setup:** Connect the projector's clock output to an Arduino board (or equivalent controller) to trigger LED progression [S1].
5.  **Sequence Execution:**
    a.  The projector displays image $i$ at frame rate $f_{proj}$ [S1].
    b.  Upon each image update (triggered by the projector clock), the controller activates LED $i$ [S1].
    c.  LEDs are addressed in order of their peak wavelengths [S1].
    d.  Apply calibrated pulse width modulation (PWM) to LED $i$ to homogenize plane intensity [S1].
    e.  Repeat for all $N$ wavelengths. The sequence repeats at a volume refresh rate $f_{vol} \ge 20$ Hz to avoid visible flicker [S1].

**Formulas**
*   **DOE Zone Boundary:** The boundary $r_m$ of each diffractive zone $m$ is found using the non-approximated formula [S1]:
    $$ r_m = \sqrt{2m\lambda_0 f + (m\lambda)^2} $$
    Where:
    *   $m$ is the zone number (integer).
    *   $\lambda_0$ is the design wavelength.
    *   $f$ is the focal length at $\lambda_0$.
    *   $\lambda$ is the operating wavelength.
    *   *Note:* The paraxial approximation fails for low f-numbers; this full formula must be used [S1].

*   **Focal Length Variation:** The focal length varies linearly with wavelength in the measured range [S1]:
    $$ f(\lambda) \approx f_0 \frac{\lambda_0}{\lambda} $$
    The measured variation is approximately $0.26$ mm/nm [S1].

*   **Magnification:** The magnification of image planes varies with wavelength due to the wavelength-dependent focal length of the DOE lens [S1].

**Parameters**
*   **DOE Lens:**
    *   Diameter: Ø150 mm [S1].
    *   Material: Acrylic [S1].
    *   Focal Length: 150 mm at design wavelength 500 nm (f/1) [S1].
    *   Surface Profile Depth: 1 μm [S1].
    *   Feature Width: As small as 1 μm wide at the lens edge [S1].
    *   Diffraction Efficiency: 13% [S1].
*   **Light Source:**
    *   Type: Gallium nitride (GaN) RCLEDs [S1].
    *   Array Size: 4 × 8 array, spanning 13.55 mm × 10.10 mm [S1].
    *   Wavelength Range (Selected): 469.82–538.68 nm (20 LEDs used) [S1].
    *   Linewidth (FWHM): Average 6.9 nm [S1].
    *   Power Range: 3.44–9.97 mW [S1].
    *   Modulation Frequency: 400 Hz [S1].
*   **Projector:**
    *   Model: Texas Instruments DLinnovations CEL 5500 [S1].
    *   Resolution: 1024 × 768 (XGA) [S1].
    *   Frame Rate: 500 Hz for 8-bit images [S1].
*   **Optics:**
    *   Fresnel Lens: f/1, diameter 152 mm, focal length 152 mm [S1].
    *   Distance from diffuser to Fresnel lens: One focal length (152 mm) [S1].
    *   Distance from Fresnel lens to DOE lens: 252 mm [S1].
*   **Performance:**
    *   Number of Image Planes: 20 [S1].
    *   Volume Refresh Rate: 20 Hz [S1].
    *   Depth of Field: 13 mm (without brightness compensation) [S1].
    *   Overall Image Volume: 17.4 cm³ [S1].
    *   Floating Distance: 145 mm above the lens [S1].
    *   Field of View: 10° [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration
N = 20  # Number of LEDs/planes
wavelengths = [469.82, ..., 538.68]  # nm, sorted ascending
led_powers = [3.44, ..., 9.97]  # mW, corresponding to wavelengths
images = load_8bit_images(N)  # List of N images
proj_frame_rate = 500  # Hz
vol_refresh_rate = 20  # Hz
led_mod_freq = 400  # Hz

# Pre-compute PWM duty cycles to homogenize intensity
# Assuming target intensity I_target and max brightness B_max
# duty_cycle_i = (I_target / led_powers[i]) * scaling_factor
# Clamped to [0, 1]
pwm_duties = homogenize_intensity(led_powers, target_intensity=1.0)

# Synchronization Loop
# The projector triggers an interrupt/callback on each frame update
def on_projector_frame_update(frame_index):
    # frame_index cycles from 0 to N-1
    current_led_index = frame_index % N
    
    # Get the LED corresponding to this wavelength
    led = leds[current_led_index]
    
    # Apply PWM to the LED
    led.set_pwm(pwm_duties[current_led_index])
    
    # Ensure LED is active only for the duration of the frame
    # The hardware/controller handles the timing relative to the projector clock
    pass

# Main Execution
while True:
    # Projector runs continuously at proj_frame_rate
    # The callback 'on_projector_frame_update' handles LED switching
    # The sequence repeats, creating a volume refresh rate of:
    # vol_refresh_rate = proj_frame_rate / N
    # 500 / 20 = 25 Hz (satisfies >= 20 Hz requirement)
    pass
```

**Edge Cases and Pitfalls**
*   **Paraxial Approximation:** The paraxial approximation fails for low f-numbers (e.g., f/1). The full non-approximated formula for zone boundaries must be used [S1].
*   **LED Spectral Purity:** LED wavelength distributions may have overlapping peaks and gaps. LEDs with wide linewidths or multiple peaks must be excluded to prevent plane overlap and gaps in the volume [S1].
*   **Brightness Variation:** LED power varies significantly (3.44–9.97 mW). Without compensation, brighter wavelengths dominate, causing a fragmented volume perception [S1].
*   **Coupling Efficiency:** The distributed nature of the LED array causes varying coupling efficiencies into the projector, leading to uneven illumination [S1].
*   **Blue End Sensitivity:** The blue end of the spectrum suffers from low brightness, exacerbated by the human eye's lower sensitivity at these wavelengths [S1].
*   **Off-Axis Distortion:** Off-axis distortion appears as spherical aberration caused by highly non-paraxial conditions near the lens edge. Larger patterns projected off-axis are clipped at the edges by the lens aperture [S1].
*   **Viewing Angle:** The viewing angle decreases with image size on the diffuser, tending towards the paraxial condition. For a 40x40 mm image, the viewing angle is only 4° [S1].

## Subtopics

### [Galvanometer Scanner Volume Rendering](5277-galvanometer-scanner-volume-rendering.md)
Uses mechanical scanning mirrors to sweep a point or line across a volume. Chose when high resolution per plane is needed and mechanical complexity is acceptable.

### [Drawing Space Separation (DSS)](5290-drawing-space-separation-dss.md)
Separates the drawing space from the viewing space using optical elements. Chose when decoupling the display optics from the viewer's position is critical.

### [Dual-Path Holographic Laser Rendering](5627-dual-path-holographic-laser-rendering.md)
Uses two laser paths to create holographic voxels. Chose when full color and high coherence are required, accepting higher complexity.

### [Multi-view Lenticular Volume Rendering](6636-multi-view-lenticular-volume-rendering.md)
Uses lenticular lenses to direct different views to different angles. Chose for autostereoscopic 3D without head-tracking.

### [Photophoretic-Trap Volumetric Display](6753-photophoretic-trap-volumetric-display.md)
Traps particles in a laser beam to emit light. Chose for true volumetric point sources but limited by particle dynamics.

### [Holographic Computer-Generated Holography (CGH) Calculation](6884-holographic-computer-generated-holography-cgh-calculation.md)
Calculates holograms for spatial light modulators. Chose for high-quality wavefront reconstruction but high computational cost.

### [Quantum-Dot-Based Volumetric Display](6943-quantum-dot-based-volumetric-display.md)
Uses quantum dots for color conversion in volumetric structures. Chose for specific color properties in solid-state media.

### [Material-Based Volumetric Emission](7614-material-based-volumetric-emission.md)
Uses materials that emit light when excited. Chose for passive volumetric displays without active scanning.

### [Dynamic Projection Mapping for Rotating Volumetric Display](7619-dynamic-projection-mapping-for-rotating-volumetric-display.md)
Projects onto a rotating screen. Chose for simple mechanics but limited to rotational symmetry.

### [Rotational Swept-Screen Projection](8589-rotational-swept-screen-projection.md)
Sweeps a screen rotationally. Chose for large volumes but requires mechanical rotation.

## Operating characteristics
*   **Time Complexity:** The system operates at a volume refresh rate of 20 Hz, requiring $N$ image updates per volume frame [S1]. With $N=20$ and a projector rate of 500 Hz, the system is not computationally bottlenecked by the projector [S1].
*   **Memory Complexity:** Requires storage for $N$ 8-bit images. For $N=20$ and 1024x768 resolution, this is minimal [S1].
*   **Measured Performance:** Achieved 20 image planes at 20 Hz, creating a volume of 17.4 cm³ with 13 mm depth and 10° FOV, floating 145 mm above the lens [S1].
*   **Scaling:** The volume size is limited by the lens diameter and focal length. Larger volumes require larger lenses or different optical configurations [S1].
*   **Strengths:** Solid-state (no moving parts), eye-safe, produces images within reach, larger images than laser plasma approaches [S1].
*   **Limitations:** Restricted field of view (defined by lens f-number), inability to display color images (due to wavelength multiplexing), fragmented volume perception due to brightness variations, off-axis distortion [S1].
*   **When to Use:** Reviewing medical scans or viewing point cloud data for LIDAR/RADAR, where occlusion is not critical [S1].
*   **When Not to Use:** Applications requiring full color, large fields of view, or high uniformity without post-processing [S1].

## Background
*   **Origin:** Proposed by researchers from the University of Edinburgh and Heriot-Watt University, published in *Optics Letters* in 2019 [S1].
*   **History:** The method builds on Fourier imaging theory and the use of diffractive optical elements (DOEs) for chromatic dispersion [S1]. It addresses limitations of mechanically swept displays and high-power laser holographic solutions [S1].
*   **Usage:** Demonstrated with medical MRI image slices [S1].

## Sources
[S1] Volumetric 3D display in real space using a diffractive lens, fast projector, and polychromatic light source, https://doi.org/10.1364/ol.44.004901, fetched 2026-10-30
