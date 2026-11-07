# Drawing Space Separation (DSS)

Also called: Drawing Space Separation (DSS) Volumetric Display, Holographic-laser-excited graphics

## Overview
Drawing Space Separation (DSS) is a volumetric display technique that physically separates the region where voxels are generated (the drawing space) from the region where the user views the image (the viewing space) using an optical re-projection system [S1, S2]. This architecture allows the drawing space to be filled with a material, such as xenon gas, that significantly lowers the energy threshold for voxel generation and provides broadband emission, without creating a physical barrier between the user and the graphics [S1, S2]. Unlike sibling methods that rely on capturing scattering particles or direct projection, DSS generates robust, non-contact aerial voxels that can be colored arbitrarily by synchronizing temporal color filtering with spatial beam scanning [S1, S2].

## How it works

### Inputs and Outputs
**Inputs:**
*   3D point cloud data representing the volumetric graphic, defined by vertex coordinates $(x_i, y_i, z_i)$ and target color values $(r_i, g_i, b_i)$ for each voxel $i$ [S1, S2].
*   A femtosecond laser source with center wavelength $\lambda = 800$ nm, pulse duration $\tau = 34$ fs, repetition frequency $f_{rep} = 1$ kHz, and average power $P_{avg} = 7$ W [S1].
*   A Liquid Crystal on Silicon Spatial Light Modulator (LCOS-SLM) with resolution $1272 \times 1024$ pixels, pixel pitch $p = 12.5$ $\mu$m, 8-bit depth, and refresh rate $\approx 10$ Hz [S1].
*   A 3D beam scanning system comprising a galvanometer scanner (deflection angle $\pm 40^\circ$) and a varifocal lens (aperture 16 mm, focal length range $-500$ to $+333$ mm) [S1].
*   An LC color filter capable of switching transmission wavelengths [S1, S2].

**Outputs:**
*   A real, aerial volumetric image in the viewing space, visible from a 360° horizontal angle, composed of voxels with arbitrary colors [S1, S2].

### Data Structures
The system processes 3D point cloud data. Each voxel $i$ is defined by:
*   Position: $\mathbf{p}_i = (x_i, y_i, z_i)$ in the drawing space coordinate system.
*   Color: $\mathbf{c}_i = (r_i, g_i, b_i)$, where $r, g, b$ correspond to the transmission states of the LC color filter [S2].

The LCOS-SLM stores a Computer-Generated Hologram (CGH) phase pattern $\Phi(x, y)$ calculated to diffract the laser beam into the required focal points [S1].

### Optical Layout
The re-projection system consists of two parabolic mirrors with their concave sides facing each other [S1, S2]:
1.  **Bottom Mirror:** Contains a central aperture. The drawing space (a cylindrical glass cell, diameter 30 mm, height 30 mm, filled with Xenon gas) is located at this aperture [S1].
2.  **Top Mirror:** Contains a central aperture through which the user views the image.
3.  **Alignment:** The vertices of the two mirrors are separated by a distance equal to the focal length $f_{mirror}$ of the mirrors [S1, S2].

### Numbered Steps

1.  **Laser Preparation:** The femtosecond laser beam passes through a half-wave plate and polarization beamsplitter to control pulse energy, then through the LCOS-SLM [S1].
2.  **Phase Modulation (Parallel Generation):** The LCOS-SLM displays a CGH phase pattern $\Phi(x, y)$ that modulates the phase of the laser pulses. This enables the parallel generation of multiple focal points if the point cloud requires simultaneous voxel rendering [S1].
3.  **3D Beam Scanning:** The modulated beam enters the 3D beam scanning system. The galvanometer scanner adjusts the deflection angle to control the horizontal ($x, y$) position of the focal point, while the varifocal lens adjusts the focal length to control the axial ($z$) position [S1].
4.  **Voxel Generation in Drawing Space:** The focused laser irradiates the Xenon gas in the aperture of the bottom mirror. The incident fluence must exceed the emission threshold. In Xenon, the threshold is $19$ mJ/cm$^2$, which is $34\%$ lower than in air, and the resulting emission is $\approx 200$ times brighter than air luminescence at this intensity [S1, S2]. The laser pulse energy for rendering typically ranges from $117$ $\mu$J to $271$ $\mu$J depending on the scene [S1].
5.  **Re-projection:** The light emitted from the voxel in the bottom mirror's aperture travels to the top mirror. Due to the parabolic geometry and vertex separation equal to the focal length, the emission is re-projected as a real image through the aperture of the top mirror into the viewing space [S1, S2].
6.  **Color Extraction:** As the light passes through the aperture of the top mirror, it traverses the LC color filter. The filter selectively extracts specific wavelengths from the broad-spectrum emission of the femtosecond-laser-excited Xenon [S1, S2].
7.  **Temporal Synchronization:** The system synchronizes the spatial position of the voxel generation (controlled by the scanner) with the color state of the LC filter. By switching the filter color in time with the arrival of the laser pulse at a specific voxel location, arbitrary colors $(r_i, g_i, b_i)$ are assigned to each voxel [S1, S2].
8.  **Persistence of Vision:** The scanning speed is high enough that the human eye perceives the sequence of emitted voxels as a continuous volumetric image [S1, S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
LAMBDA_CENTER = 800e-9  # meters
PULSE_DURATION = 34e-15  # seconds
REP_FREQ = 1000  # Hz
THRESHOLD_FLUENCE_XE = 19e-3  # J/cm^2
MIRROR_FOCAL_LENGTH = 0.05  # meters (Example value, source says separation = focal length)
DRAWING_SPACE_CENTER = (0, 0, 0)  # meters, relative to bottom mirror aperture

def calculate_cgh_phase(points, wavelength, pixel_pitch, distance):
    """
    Calculate the phase pattern for an LCOS-SLM to focus light at specific 3D points.
    This implements the CGH beam design technique.
    """
    # Implementation details for CGH calculation are not provided in the source.
    # The source states CGH is used for parallel generation of focal points.
    # A standard approach involves calculating the interference pattern between a reference plane wave
    # and spherical waves originating from the target points.
    raise NotImplementedError("CGH phase calculation algorithm not detailed in source.")

def scan_voxel_sequence(point_cloud):
    """
    Renders a volumetric graphic by scanning voxels and synchronizing color.
    """
    # Sort points into layers for efficient scanning
    layers = slice_point_cloud_into_layers(point_cloud)
    
    for layer in layers:
        # Calculate shortest path between vertices in the layer (TSP-like heuristic)
        scan_path = calculate_shortest_scan_path(layer.vertices)
        
        for voxel in scan_path:
            # 1. Set Color Filter
            # The LC filter switches to the color corresponding to the voxel's target color
            # The source mentions switching between cyan, magenta, yellow, or RGB values.
            set_lc_filter_color(voxel.color)
            
            # 2. Update SLM (if parallel generation is used, this might be batched)
            # If single voxel scanning, SLM might be static or updated for deflection
            update_slm_for_deflection(voxel.position)
            
            # 3. Move Scanners
            # Galvanometer sets x,y; Varifocal lens sets z
            move_galvanometer(voxel.position.x, voxel.position.y)
            set_varifocal_lens_focal_length(voxel.position.z)
            
            # 4. Emit Laser Pulse
            # Pulse energy is adjusted if necessary, though source implies fixed settings per scene
            emit_laser_pulse()
            
            # 5. Wait for persistence of vision / frame rate
            # The laser repetition rate is 1 kHz, but the visual refresh depends on scan speed
            wait_for_next_pulse_or_sync()

def set_lc_filter_color(color):
    """
    Sets the transmission wavelength of the LC color filter.
    """
    # The LC filter is placed immediately after the top mirror's aperture.
    # It switches transmission wavelength over time.
    pass

def slice_point_cloud_into_layers(point_cloud):
    """
    Slices 3D point cloud data into 2D cross-sections.
    """
    # Group points by z-coordinate
    pass

def calculate_shortest_scan_path(vertices):
    """
    Sets scanning paths to the shortest scanning distances between vertices in each layer.
    """
    # Heuristic for nearest-neighbor path
    pass
```

### Parameters and Typical Values
*   **Laser:** 800 nm center wavelength, 34 fs pulse duration, 1 kHz repetition, 7 W average power [S1].
*   **Drawing Space:** Cylindrical glass cell, 30 mm diameter, 30 mm height, filled with Xenon (Xe) gas [S1].
*   **Threshold:** Xe emission threshold is $19$ mJ/cm$^2$; Air emission threshold is higher (Xe reduces threshold by up to 34%) [S1, S2].
*   **Brightness:** Xe emission is $\approx 200$ times brighter than air at $19$ mJ/cm$^2$ [S1, S2].
*   **Mirrors:** Two parabolic mirrors, vertices separated by focal length. Apertures are central [S1, S2].
*   **LCOS-SLM:** $1272 \times 1024$ resolution, 12.5 $\mu$m pitch, 8-bit, ~10 Hz refresh [S1].
*   **Scanner:** Galvanometer $\pm 40^\circ$ deflection; Varifocal lens 16 mm aperture, $-500$ to $+333$ mm focal range [S1].
*   **Rendering Energies:** $271$ $\mu$J (Fig 5a), $203$ $\mu$J (Fig 5c, 5f), $219$ $\mu$J (Fig 5e), $117$ $\mu$J (Fig 5g) [S1].

### Edge Cases and Pitfalls
*   **Physical Barrier:** Previous systems filling the space with gas to lower energy thresholds created a physical wall (glass cell) between the user and the image, reducing realism. DSS solves this by re-projecting the image out of the cell, leaving the viewing space as open air [S1, S2].
*   **Energy Density vs. Size:** In air, high pulse energy is required to form high-energy density, necessitating short focal length lenses, which restricts graphics size to several millimeters. DSS allows the use of lower energy densities in Xe, enabling larger graphics (centimeter-order) [S1, S2].
*   **Monochromatic Emission:** Femtosecond-laser-excited voxels in gas emit bluish-white broadband light. Color is not intrinsic to the voxel but is extracted optically. Synchronization errors between scanner position and filter state will result in incorrect colors or ghosting [S1, S2].
*   **Ambient Light:** While Xe is 200x brighter than air, the system is demonstrated under ambient room lighting. High ambient light may reduce contrast [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The rendering time is dominated by the scanning speed of the galvanometer and varifocal lens, and the refresh rate of the LCOS-SLM and LC filter. The laser repetition rate (1 kHz) sets an upper bound on voxel emission frequency. The source notes real-time AR demonstration (3D saber) at 20 fps [S1].
*   **Memory Complexity:** Depends on the size of the point cloud and the CGH pattern stored on the SLM ($1272 \times 1024$ pixels) [S1].

### Measured Performance
*   **Resolution:** Demonstrated with up to 5734 voxels (Armadillo model) [S1].
*   **Size:** Centimeter-order images (e.g., 30 mm drawing space) [S1].
*   **Color:** Arbitrary colors achieved via LC filter switching (Cyan, Magenta, Yellow, RGB) [S1].
*   **Brightness:** Visible under ambient room lighting due to Xe enhancement [S1].

### Strengths and Limitations
*   **Strengths:**
    *   No physical barrier between user and image (open viewing space) [S1, S2].
    *   Robust graphics that do not disappear upon contact with users or objects (unlike particle traps) [S2].
    *   Full-color capability from a monochromatic laser source via spectral filtering [S1, S2].
    *   Lower energy threshold for voxel generation compared to air [S1, S2].
*   **Limitations:**
    *   Requires complex optical alignment (two parabolic mirrors) [S1].
    *   Limited by the scanning speed and SLM refresh rate for dynamic content [S1].
    *   Drawing space is confined to the re-projection volume defined by the mirrors [S1].

### When to Use / Not Use
*   **Use:** When robust, interactive, full-color volumetric graphics are needed in open space without physical screens or particle traps. Suitable for AR demonstrations and detailed static/dynamic models [S1, S2].
*   **Not Use:** When extremely high voxel counts or large-scale graphics (meter-scale) are required, as the scanning mechanism and re-projection optics limit the size and density [S1]. Not suitable for applications requiring intrinsic color emission without filtering [S1].

## Background

### Origin and History
Drawing Space Separation (DSS) was introduced in a 2021 paper by researchers publishing in *Scientific Reports* (Nature Portfolio) [S1, S2]. The work builds upon previous systems that used femtosecond lasers to create aerial voxels in air, which were limited to monochromatic bluish-white light and small sizes due to high energy requirements [S1, S2]. The aerial re-projection method using two parabolic mirrors was reported in a US patent 50 years prior to the DSS paper [S2].

### Development
The development addressed two main limitations of prior femtosecond-laser volumetric displays:
1.  **Color:** Previous systems emitted monochromatic light. DSS introduced an LC color filter in the re-projection path to extract arbitrary colors [S1, S2].
2.  **Energy/Size:** Previous systems required high energy in air, limiting size. DSS introduced a material-assisted drawing space (Xe gas) separated from the viewing space, lowering the threshold and allowing larger, brighter images [S1, S2].

### Usage
The system has been demonstrated for:
*   Centimeter-order static images (Armadillo, Stanford Bunny) [S1].
*   Real-time Augmented Reality (3D light saber) [S1].
*   Point cloud visualization with per-voxel coloring [S1].
*   Interaction with physical objects (graphics touching a 3D-printed bunny) [S1].

## Sources

[S1] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation, https://www.youtube.com/watch?v=W-CbMq23NKk, fetched 2026-10-28
