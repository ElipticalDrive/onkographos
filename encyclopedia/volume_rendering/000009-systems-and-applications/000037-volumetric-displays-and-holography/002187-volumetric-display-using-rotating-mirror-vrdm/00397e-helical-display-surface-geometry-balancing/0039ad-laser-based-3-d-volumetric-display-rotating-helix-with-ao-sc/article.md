# Laser-based 3-D Volumetric Display (Rotating Helix with AO Scanner)

Also called: AO-Scanned Helix Display, Rotating Helix Volumetric Display, Helical Volumetric Display, Helix-based 3-D Volumetric Display, Double Helix Laser Display, Laser-based rotating double helix volumetric display, 3-D Volumetric Display (Helix-based Laser Projection), Rotating Helical Volumetric Display, Volumetric Display using Helical Surface, Helix-based 3-D Display, Helical Display System

## Overview
A hardware-based volumetric display technique that projects 3-D image points (voxels) onto the surface of a rapidly rotating helical structure using computer-directed laser beams, creating a true 3-D image space visible to multiple viewers without eyewear [S1, S2, unofficial]. It solves the problem of displaying true 3-D data with physiological depth cues (accommodation, convergence, motion parallax) by forming physical light points in mid-air, unlike 2-D displays that rely on psychological cues [S2, unofficial]. It is distinguished from sibling methods like [Phase-Controlled Laser Pointing for Swept Displays](10333-phase-controlled-laser-pointing-for-swept-displays.md) by its use of random-access AO scanning for voxel placement rather than phase-synchronized continuous lines, and from [Spot Size Reduction Methods](11725-spot-size-reduction-methods.md) by its reliance on the helical geometry and AO deflection rather than purely geometric optics for spot definition [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3-D Cartesian coordinates $(x, y, z)$ for each desired voxel, a synchronization pulse from a sensor on the helix hub, and control signals for laser intensity [S1, S2, unofficial].
*   **Output:** A volumetric image composed of discrete voxels visible in real space, formed by laser light scattering from the rotating helix surface [S2, unofficial].

**Data Structures:**
*   **Frame Buffer:** Stores voxel coordinates. In second-generation systems, the z-axis is mapped from the spatial domain to a temporal domain (height maps to memory location) [S2, unofficial].
*   **Control Memory:** Capacity of 65,000 voxels per color in second-generation implementations [S2, unofficial]. Addressability is $4096 \times 4096$ per display volume [S1, unofficial].
*   **Scanner State:** Tracks the current RF frequency sweep for X and Y deflectors to determine beam angle [G].

**Numbered Steps:**
1.  **Coordinate Translation:** Translate raw $(x, y, z)$ coordinates into the computer memory such that the z-axis (height) is mapped to a temporal domain corresponding to the helix's rotational phase [S2, unofficial].
2.  **Synchronization:** Wait for a synchronization pulse from a sensor on the helix hub indicating a known rotational angle (e.g., top dead center) [S2, unofficial].
3.  **Surface Position Calculation:** Calculate the position of the rotating helix surface at the exact moment the point needs to be displayed based on the rotational speed and the target $(x, y)$ coordinates [S1, unofficial]. The vertical voxel position $z$ is determined by the intersection of the beam and the helix surface, defined by the rotation angle $\theta$: $z = f(x, y, \theta)$ [S1, unofficial].
4.  **Beam Steering:** Send $(x, y)$ coordinate pairs to the Acousto-Optic (AO) scanner in sequence to address the correct radial position. The AO scanner uses two deflectors for independent X and Y deflection [S1, unofficial].
5.  **Intensity Modulation:** Modulate the laser intensity to illuminate only specific voxels, leaving dark areas unaddressed [S1, unofficial].
6.  **Projection:** The laser beam illuminates the corresponding discrete points on the rotating helix surface. Light scatters from the surface, making voxels appear to emanate from specific points in real space [S2, unofficial].
7.  **Refresh:** Repeat at the refresh rate (typically 20 Hz) to maintain persistence of vision [S1, unofficial].

**Formulas:**
*   **Voxel Rate:** The maximum rate at which voxels can be addressed is limited by the access time of the AO deflector.
    $$ \text{Voxel Rate} = \frac{1}{\text{Access Time}} $$
    For an access time of $10 \, \mu\text{sec}$, the max voxel rate is $100,000$ voxels/sec [S1, unofficial].
*   **Usable Voxels per Frame:**
    $$ \text{Usable Voxels per Frame} = \frac{\text{Voxel Rate}}{\text{Refresh Rate}} - \text{Dwell Time Voxels} $$
    For $100,000$ voxels/sec and $20$ Hz refresh, the theoretical max is $5,000$ voxels [S1, unofficial].
*   **Half-Angle Scan Range:** The deflection angle of an AO deflector.
    $$ \theta_{AO} = \frac{\lambda \Delta f}{v_s} $$
    Where $\lambda$ is optical wavelength, $\Delta f$ is RF frequency sweep, and $v_s$ is acoustic velocity [S1, unofficial].
*   **Beam Divergence Constant:**
    $$ \theta = \frac{\lambda}{d} $$
    Where $\theta$ is initial beam divergence angle and $d$ is laser beam waist diameter [S1, unofficial].
*   **Vertical Velocity of Helix Surface:**
    $$ v_z = \frac{dz}{dt} = \frac{dz}{d\theta} \frac{d\theta}{dt} = \frac{h}{2\pi} \cdot 2\pi f = h f $$
    Where $h$ is helix height and $f$ is rotation rate [S1, unofficial].

**Parameters:**
*   **Helix Dimensions:** Laboratory model: 36-inch diameter, 18-inch height, 36-inch pitch, 2 mm blade thickness [S1, unofficial]. Transportable/Commercial: 12-inch diameter, 8-inch height [S1, unofficial].
*   **Rotation Speed:** 600 RPM for double-helix (2 blades) to achieve 20 Hz refresh ($f = \text{Refresh Rate} / \text{Blades} = 20/2 = 10$ rev/sec = 600 RPM) [S1, unofficial]. Single-spiral commercial variant: 1200 RPM [S1, unofficial].
*   **Refresh Rate:** 20 Hz [S1, unofficial].
*   **Access Time:** $10 \, \mu\text{sec}$ for single-channel; $5 \, \mu\text{sec}$ for four-channel systems (achieved by reducing beam diameter from 6 mm to 3 mm) [S1, unofficial].
*   **Deflector Sweep Range:** 75 to 125 MHz ($\Delta f = 50$ MHz) [S1, unofficial].
*   **Acoustic Velocity ($v_s$):** $0.65$ mm/ms [S1, unofficial].
*   **Input Beam Diameter:** $1.6$ mm at the scanner input [S1, unofficial].
*   **Voxel Size:** $1.5$ mm (24-inch system); $0.7$ mm (compact 12-inch system) [S1, unofficial].
*   **Brightness:** 256 levels per color [S1, unofficial].
*   **Laser Sources:**
    *   Green: 2x 250 mW diode-pumped doubled Nd:YAG (532 nm) [S1, unofficial].
    *   Red: 8x 30 mW single-mode red diode lasers (633 nm) or HeNe (633 nm) [S1, unofficial].
*   **Laser Safety:** <15 mW average per image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time

def wait_seconds(t):
    time.sleep(t)

def set_ao_deflector_y(angle_rad):
    # Convert angle to RF frequency; exact synthesis algorithm not given in sources [G]
    pass

def set_ao_deflector_x(angle_rad):
    # Convert angle to RF frequency; exact synthesis algorithm not given in sources [G]
    pass

def set_laser_intensity(level):
    # Set AO modulator intensity via RF drive [G]
    pass

def pulse_laser():
    # Trigger laser pulse synchronized with deflection [G]
    pass

def render_volumetric_display(frame_buffer, helix_params, scanner_params):
    PI = 3.141592653589793
    h_mm = helix_params['helix_height_in'] * 25.4
    rpm = helix_params['rotation_speed_rpm']
    f_rps = rpm / 60.0
    refresh_rate = helix_params['refresh_rate']
    access_time_s = scanner_params['access_time_us'] / 1e6
    max_voxel_rate = 1.0 / access_time_s
    theoretical_max_voxels = max_voxel_rate / refresh_rate
    
    wait_seconds(0.1) # Wait for sync pulse from hub sensor
    
    for voxel in frame_buffer:
        x, y, z, intensity = voxel
        theta_rad = (2 * PI * z) / h_mm
        omega = 2 * PI * f_rps
        time_delay_s = theta_rad / omega
        wait_seconds(time_delay_s)
        
        lambda_m = scanner_params['wavelength_nm'] / 1e9
        delta_f_hz = scanner_params['deflector_sweep_range_MHz'] * 1e6
        v_s_ms = scanner_params['acoustic_velocity_mm_ms']
        total_deflection_rad = (2 * lambda_m * delta_f_hz) / v_s_ms
        
        angle_x_rad = x * (total_deflection_rad / 2)
        angle_y_rad = y * (total_deflection_rad / 2)
        
        set_ao_deflector_y(angle_y_rad)
        set_ao_deflector_x(angle_x_rad)
        set_laser_intensity(intensity)
        pulse_laser()
        wait_seconds(access_time_s)
```

## Subtopics

### [Phase-Controlled Laser Pointing for Swept Displays](10333-phase-controlled-laser-pointing-for-swept-displays.md)
An interaction method that resolves the ambiguity of laser pointers projecting continuous lines by synchronizing pulse frequency with sweep frequency and adjusting phase offset via joystick for precise z-coordinate control. Choose when manual selection ray positioning is required.

### [Spot Size Reduction Methods](11725-spot-size-reduction-methods.md)
Optical and mechanical strategies to reduce laser spot diameter and addressable area, addressing excessive voxel size. Choose when higher spatial resolution is needed than the base AO scanner provides.

## Operating characteristics
*   **Time Complexity:** The voxel rate is limited by the AO access time. Single-channel systems achieve ~100,000 voxels/sec. Four-channel systems achieve ~200,000 voxels/sec per channel, totaling up to 1.6 million voxels/sec for the first commercial model or 3.2 million voxels/sec for the transportable dual-volume system [S1, unofficial].
*   **Memory Complexity:** Frame buffer size is limited by control card memory, such as 65,000 voxels per color in second-generation systems [S2, unofficial].
*   **Measured Performance:** The laboratory model (36-inch) produces 120,000 voxels per frame (RGB) at 2.4 million voxels/sec [S1, unofficial]. The transportable model (12-inch) produces 160,000 total voxels per frame at 1.6 million voxels/sec [S1, unofficial]. The measured spot size is ~4 ± 0.5 mm at the helix for the 36-inch system, and 0.7 mm for the 12-inch system [S1, unofficial].
*   **Scaling:** Voxel count scales with the number of parallel AO scanner channels. Beam diameter reduction from 6 mm to 3 mm halves access time, doubling voxel throughput per channel [S1, unofficial].
*   **Strengths:** The system provides true physiological depth cues (accommodation, convergence, motion parallax) for group viewing without eyewear [S2, unofficial]. It is highly compatible with the human visual system compared to stereoscopic or holographic displays [S1, unofficial].
*   **Limitations:** Reflective helix blades can block image portions, creating occlusion zones; this is mitigated by using a translucent helix and bottom projection [S1, unofficial]. The center shaft creates a dead zone where voxels cannot be placed [S1, unofficial]. Steep helical slope near the central axis causes spot elongation; primary image creation is restricted to regions midway between the center and the rim [S1, unofficial]. The addressable volume is constrained by scanner optics and occlusions (e.g., 10" x 10" x 15" in second-generation systems) [S2, unofficial]. Early systems used ion lasers requiring water cooling, though solid-state lasers reduce power consumption by ~1000x [S1, unofficial].
*   **When to Use:** Applications requiring true 3D visualization for multiple viewers, such as air traffic control, submarine navigation, battle management, and medical imaging [S1, S2, unofficial].
*   **When Not to Use:** Applications requiring extremely high spatial resolution (sub-millimeter) without hardware modification, or where the central dead zone is critical [G].

## Background
The concept was invented by Professor Rüdiger Hartwig at the University of Stuttgart in the early 1980s [S1, S2, unofficial]. Feasibility was demonstrated by Dr. R. D. Williams and team at Texas Instruments in late 1989 [S1, S2, unofficial]. R&D began at SSC San Diego (SPAWAR Systems Center, San Diego) in early 1990 [S1, S2, unofficial]. First-generation prototypes were built by NCCOSC RDT&E Division in the early 1990s [S2, unofficial]. Second-generation development objectives were established in mid-1992, with a report published in April 1995 [S2, unofficial]. The technology was commercialized via a 1995 CRADA with NEOS Technologies, Inc. and RGB Technology, Inc. for military and government use [S1, S2, unofficial]. It was deployed on the USS Stennis for the Joint Warrior Interoperability Demonstration (JWID 97) and installed in the SPAWAR Command Center of the Future [S1, unofficial].

## Sources
[S1] DTIC ADA358303: Improved Second-Generation 3-D Volumetric Display System. Revision 2, https://archive.org/details/DTIC_ADA358303, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA306215: Laser-Based 3-D Volumetric Display System (The Improved Second Generation)., https://archive.org/details/DTIC_ADA306215, fetched 2026-10-26 (unofficial)
[S3] DTIC ADA264825: Laser Based 3D Volumetric Display System, https://archive.org/details/DTIC_ADA264825, fetched 2026-10-26 (unofficial)
