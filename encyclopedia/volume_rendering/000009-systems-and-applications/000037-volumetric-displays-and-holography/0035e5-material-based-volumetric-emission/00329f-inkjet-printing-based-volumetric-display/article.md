# Inkjet Printing-Based Volumetric Display

Also called: Fluorescent Ink Volumetric Display, Volumetric Display via Inkjet Printing

## Overview
This method constructs a physical volumetric display by printing photoreactive luminescence materials (fluorescent inks) onto multiple layers of transparent films using a commercial inkjet printer, which are then stacked to form a 3D volume [S1]. The volume elements (voxels) are excited by external ultraviolet light to emit visible light, forming a 3D image [S1]. It differs from sibling methods like [Quantum-Dot-Based Volumetric Display](6943-quantum-dot-based-volumetric-display.md) by relying on pre-printed static layers rather than dynamic optical addressing or volumetric scattering within a single bulk medium [G].

## How it works
**Inputs:** 3D object data or 2D patterns to be projected, printer resolution settings, and stack configuration parameters (layer count, spacing) [S1].
**Outputs:** A physical stack of transparent films containing printed fluorescent patterns, and a 3D image visible when excited by UV light [S1].
**Data Structures:** The display consists of a stack of $L$ layers, where $L=20$ in the prototype [S1]. Each layer $l$ (where $l=0 \dots L-1$) is a 2D grid of pixels mapped to physical film coordinates [G]. The 3D position of a voxel is $(x, y, z_l)$, where $z_l = l \times \Delta z$ and $\Delta z = 0.5$ mm [S1, G]. The rendering space measures $35 \times 35 \times 12.5$ mm [S1].

**Algorithm for Single 3D Image:**
1. **Slicing:** Convert the 3D object into a series of 2D cross-sectional images, each corresponding to one layer of the stack [S1].
2. **Printing:** Print each 2D image onto a separate transparent film using an inkjet printer [S1]. The inks comprise europium complex (red), $\beta$-quinophthalone (green), and coumarin dyes (blue) [S1].
3. **Stacking:** Stack the printed films at regular vertical intervals $\Delta z = 0.5$ mm to form the volumetric display [S1].

**Algorithm for Multi-Directional Projection:**
To project multiple different 2D patterns from different viewing angles, a design algorithm determines the voxel values for the 3D structure [S1].
1. **Define Projection Axes:** Select $K$ viewing directions (e.g., perpendicular and $\pm 30^\circ$ around the Y-axis for three patterns, or $\pm 20^\circ$ around both X- and Y-axes for four patterns) [S1].
2. **Back-Projection:** For each target 2D pattern $P_k$ at angle $\theta_k$, project the pattern back onto the 3D volume grid [S1].
3. **Voxel Optimization:** Determine the ink color and intensity for each voxel $(x,y,z)$ such that the superposition of emissions from all layers, when viewed from direction $\theta_k$, approximates pattern $P_k$ [S1].
4. **Printing:** Print the resulting 3D voxel data onto the films as described in the Single 3D Image algorithm [S1].

**Excitation and Viewing:**
1. **Excitation:** Irradiate the stack with ultraviolet light peaked at 365 nm [S1].
2. **Emission:** The fluorescent inks absorb UV light and emit visible light (red, green, blue) [S1].
3. **Viewing:** Observe the emitted light from the desired direction [S1]. The highest image quality is obtained at perpendicular viewing ($\theta = 0^\circ$), while diagonal directions ($\theta = \pm 30^\circ$) preserve motion parallax but produce blurred images [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def blend_colors(current, new):
    """Simple additive blending clamped to 255."""
    return [min(255, current[i] + new[i]) for i in range(3)]

def compute_layer_images(patterns, projection_angles, num_layers, layer_spacing_mm, film_width_mm, film_height_mm, pixel_size_mm):
    """
    Computes the 2D ink patterns for each film layer to project multiple 2D patterns
    from different viewing angles.
    """
    # Initialize empty layers (RGB values 0-255)
    width_px = int(film_width_mm / pixel_size_mm)
    height_px = int(film_height_mm / pixel_size_mm)
    layers = [[[0, 0, 0] for _ in range(height_px)] for _ in range(num_layers)]

    # For each target pattern and its projection angle
    for k, (pattern_data, angle_deg) in enumerate(zip(patterns, projection_angles)):
        angle_rad = math.radians(angle_deg)
        pat_h, pat_w = pattern_data.height, pattern_data.width
        # Project pattern back onto 3D volume grid
        for y_pat in range(pat_h):
            for x_pat in range(pat_w):
                color = pattern_data.get_pixel(x_pat, y_pat)
                # Map 2D pattern coordinates to 3D voxel coordinates
                # Assuming pattern is centered and projected along Y-axis rotation
                x_3d = x_pat * pixel_size_mm
                z_3d = y_pat * pixel_size_mm * math.tan(angle_rad)
                # Determine which layer this 3D point falls into
                layer_idx = int(abs(z_3d) / layer_spacing_mm)
                if 0 <= layer_idx < num_layers:
                    # Accumulate color contribution to the layer
                    layers[layer_idx][x_pat][y_pat] = blend_colors(layers[layer_idx][x_pat][y_pat], color)
    return layers
```

**Parameters:**
*   **Film Material:** Polyester transparent film (e.g., 'BG-32' from Folex imaging Inc.) [S1].
*   **Film Thickness:** 0.1 mm [S1].
*   **Layer Spacing:** 0.5 mm [S1].
*   **Printer Resolution:** Up to 5,760 × 1,440 dpi [S1].
*   **Ink Types:** Europium complex (Red), $\beta$-quinophthalone (Green), Coumarin dyes (Blue) [S1].
*   **Excitation Wavelength:** 365 nm UV light [S1].
*   **Rendering Space:** Example dimensions 35 mm × 35 mm × 12.5 mm [S1].
*   **Original Pattern Resolution:** 512 × 512 pixels, 3 channels (RGB), 256 gradations per channel [S1].

**Edge Cases and Pitfalls:**
*   **UV Attenuation:** UV light is absorbed by the transparent films [S1]. The transmittance of the film at 365 nm is 82% [S1]. If $N_{UV}$ is the number of films between the UV source and the printed layer, the intensity drops exponentially [S1]. For $N_{UV}=20$, intensity is ~2%; for $N_{UV}=25$, it is <1% [S1].
*   **Mitigation:** Use UV light sources on both the top and bottom of the display to reduce attenuation effects [S1].
*   **Visible Light Attenuation:** Visible light emitted by the inks is also attenuated by films above it, but less severely (transmittance ~90%) [S1].
*   **Image Quality Degradation:** Images viewed from diagonal directions (e.g., $\theta = \pm 30^\circ$) are blurred compared to the perpendicular view ($\theta = 0^\circ$), although motion parallax is preserved [S1].
*   **Cross-talk:** In multi-pattern projections, signals from other patterns can interfere (cross-talk), but the original patterns remain recognizable [S1].

## Subtopics

### Multi-Directional Projection
A variant where the 3D structure is designed to project different 2D patterns from different angles [S1]. It differs from standard volumetric displays by optimizing voxel values for multiple viewing axes simultaneously [S1]. Choose this when interactive or multi-viewpoint information display is required [G]. See [Material-Based Volumetric Emission](7614-material-based-volumetric-emission.md).

### Full-Color Volumetric Display
The base method using RGB fluorescent inks to create full-color 3D images [S1]. Choose this for high-fidelity color representation [G]. See [Material-Based Volumetric Emission](7614-material-based-volumetric-emission.md).

## Operating characteristics
*   **Time Complexity:** The fabrication time is dominated by the inkjet printing process [G]. Printing a 300x300 pixel layer at high resolution takes significant time, but the method is batch-processable [G].
*   **Memory Complexity:** The data structure size is proportional to the number of layers and the resolution per layer [G]. For a 20-layer, 300x300 pixel display, the data is manageable [G].
*   **Measured Performance:**
    *   **Resolution:** Limited by printer DPI (5,760 dpi) and layer spacing (0.5 mm) [S1]. The vertical resolution is significantly lower than horizontal resolution [S1].
    *   **Brightness:** Limited by UV attenuation through the stack [S1]. Brightness decreases exponentially with depth [S1].
    *   **Viewing Angle:** Best quality at perpendicular view ($\theta = 0^\circ$) [S1]. Quality degrades at diagonal angles ($\theta = \pm 30^\circ$) due to blurring [S1].
*   **Strengths:**
    *   Uses commercially available inkjet printers and materials [S1].
    *   Capable of full-color 3D images [S1].
    *   Supports motion parallax [S1].
    *   Can project multiple different patterns from different directions [S1].
*   **Limitations:**
    *   Static display (cannot change content dynamically) [G].
    *   Limited vertical resolution due to layer spacing [S1].
    *   UV light attenuation limits the maximum number of layers [S1].
    *   Image quality degrades at oblique viewing angles [S1].
*   **When to Use:** For static, high-quality, full-color 3D displays where dynamic updating is not required, and where motion parallax is beneficial (e.g., digital signage, art installations) [S1].
*   **When Not to Use:** For dynamic content, high vertical resolution requirements, or environments where UV safety is a concern [G].

## Background
This method was introduced by Hirayama et al. in 2017 [S1]. It builds on the principles of volumetric displays by using physical layers to represent 3D space [S1]. The use of fluorescent inks allows for passive emission (no need for active matrix addressing per voxel), simplifying the hardware [G]. The extension to multi-directional projection demonstrates the flexibility of the design algorithm [S1]. The method leverages widely deployed inkjet printing technology to fabricate novel information display devices [S1].

## Sources
[S1] Inkjet printing-based volumetric display projecting multiple full-colour 2D patterns., https://doi.org/10.1038/srep46511, fetched 2026-10-26
