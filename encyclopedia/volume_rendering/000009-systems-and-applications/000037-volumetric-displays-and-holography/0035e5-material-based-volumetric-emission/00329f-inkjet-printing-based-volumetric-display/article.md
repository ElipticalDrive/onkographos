# Inkjet Printing-Based Volumetric Display

Also called: Fluorescent Ink Volumometric Display, Volumetric Display via Inkjet Printing

## Overview
This method constructs a physical volumetric display by printing photoreactive luminescent materials (fluorescent inks) onto stacked transparent films, which are then excited by external ultraviolet (UV) light to emit visible light [S1]. It differs from [Downing Three-Color Display](7247-downing-three-color-display.md) by using a static, additive manufacturing approach (inkjet printing) to create discrete voxel layers, whereas Downing’s method typically relies on rotating or scanning mechanisms to synthesize volume [S1]. The system solves the problem of creating full-color 3D images with motion parallax using commercially available printing hardware, supporting the projection of multiple distinct 2D patterns from different viewing angles [S1].

## How it works
**Inputs:**
- 3D object data or multiple 2D pattern sets (e.g., $N$ patterns for $N$ viewing directions).
- Original pattern resolution: $512 \times 512$ pixels per channel (Red, Green, Blue), 8-bit depth (256 gradations) [S1].
- Physical parameters: Film thickness $t_f = 0.1$ mm, stack interval $t_s = 0.5$ mm, rendering volume $35 \text{ mm} \times 35 \text{ mm} \times 12.5$ mm [S1].

**Outputs:**
- A physical stack of printed films forming a 3D volume.
- Excitation light source configuration (UV wavelength $\lambda_{UV} = 365$ nm) [S1].

**Data Structures:**
The core data structure is a 3D voxel grid representing the volumetric display.
- Dimensions: $W \times H \times D$, where $W=35$ mm, $H=35$ mm, $D=12.5$ mm.
- Voxel size: Determined by the printer resolution and film spacing. The prototype uses 20 layers.
- Voxel content: RGB color values $(R, G, B)$ corresponding to the concentration of specific fluorescent dyes.
- Dye mapping:
  - Red: Europium complex [S1].
  - Green: $\beta$-quinophthalone [S1].
  - Blue: Coumarin dyes [S1].

**Algorithm for Multi-View Pattern Projection:**
To project $N$ different 2D patterns from specific viewing angles, a design algorithm determines the voxel values for the 3D structure [S1]. This extends previous work to full-color representations [S1].

1.  **Define Projection Geometry:**
    - Let $P_i$ be the $i$-th target 2D pattern with resolution $W_p \times H_p$.
    - Let $\mathbf{v}_i$ be the viewing direction vector for pattern $P_i$.
    - Example 1 (3 patterns): $\mathbf{v}_B = (0, 0, 1)$ (perpendicular), $\mathbf{v}_A = (\sin(30^\circ), 0, \cos(30^\circ))$, $\mathbf{v}_C = (-\sin(30^\circ), 0, \cos(30^\circ))$ [S1].
    - Example 2 (4 patterns): Rotate Z-axis $\pm 20^\circ$ around X and Y axes [S1].

2.  **Voxel Value Determination:**
    - For each voxel $V(x,y,z)$ in the 3D grid, calculate its contribution to each target pattern $P_i$ based on the projection ray from $\mathbf{v}_i$.
    - The algorithm solves for the ink density at each voxel such that the superposition of emitted light from all voxels along the projection ray matches the target pixel color of $P_i$ [S1].
    - Note: The specific mathematical optimization function is not detailed in the provided notes, but it is described as an extension of a previous algorithm [13] [S1].

3.  **Layer Generation:**
    - Slice the 3D voxel grid into 2D layers corresponding to the film positions.
    - Each layer $L_k$ contains the 2D ink distribution for that depth.

**Fabrication Steps:**
1.  **Print:** Use a commercial inkjet printer (e.g., SO-KEN Inc., ‘TPW-105PB’) to print the 2D patterns of fluorescent inks onto polyester transparent films (Folex imaging Inc., ‘BG-32’) [S1].
    - Printer resolution: Max $5,760 \times 1,440$ dpi [S1].
    - Ink composition: Europium complex (Red), $\beta$-quinophthalone (Green), Coumarin dyes (Blue) [S1].
2.  **Stack:** Stack the printed films at regular intervals ($0.5$ mm) to form the volumetric display. In the prototype, 20 layers were used [S1].
3.  **Excite:** Irradiate the stack with UV light peaked at 365 nm [S1].
    - To mitigate UV absorption by the films, use two UV light sources placed on the top and bottom of the display [S1].
4.  **View:** Observe the emitted visible light from the designated viewing angles [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_volumetric_display(target_patterns, viewing_angles, film_count, film_spacing_mm):
    """
    Creates the voxel data for an inkjet-printed volumetric display.
    
    Args:
        target_patterns: List of 2D arrays (H x W x 3), RGB values 0-255.
        viewing_angles: List of 3D vectors (nx, ny, nz) for each pattern.
        film_count: Number of layers (e.g., 20).
        film_spacing_mm: Distance between layers (e.g., 0.5).
        
    Returns:
        voxels: 3D array (film_count x H_layer x W_layer x 3) representing ink density.
    """
    # Define rendering volume dimensions based on prototype
    width_mm = 35.0
    height_mm = 35.0
    depth_mm = film_count * film_spacing_mm
    
    # Initialize voxel grid with zero ink density
    # Resolution of layers matches the printer's effective resolution on the film
    # For the prototype, each layer is 300x300 pixels [S1]
    layer_w = 300
    layer_h = 300
    voxels = np.zeros((film_count, layer_h, layer_w, 3))
    
    # Map viewing angles to projection rays
    # Note: The exact optimization algorithm from source [13] is not provided.
    # This step represents the core design algorithm mentioned in [S1].
    for i, pattern in enumerate(target_patterns):
        angle = viewing_angles[i]
        # Project the 3D volume onto the 2D plane defined by angle
        # The algorithm determines the voxel values that sum up to the target pattern
        # when viewed from this angle.
        # This involves solving an inverse problem:
        # sum(voxel_intensity * attenuation) = target_pixel_color
        # Due to cross-talk, this is a multi-constraint optimization.
        voxels = optimize_voxel_colors(voxels, pattern, angle, film_count, film_spacing_mm)
        
    return voxels

def optimize_voxel_colors(current_voxels, target_pattern, angle, film_count, spacing):
    """
    Placeholder for the specific design algorithm [13] extended for full color.
    In the prototype, this ensures that the projected image matches the target
    despite cross-talk from other patterns [S1].
    """
    # Implementation details of the specific solver are not in the provided text.
    # The source states: "The extended algorithm is used to determine the voxel values..." [S1]
    # We assume a standard iterative back-projection or linear system solver
    # constrained by the physical properties of the inks.
    return current_voxels
```

**Edge Cases and Pitfalls:**
- **UV Attenuation:** The transmittance of the polyester film in the UV region (365 nm) is 82% [S1]. As the number of films between the UV source and the printed layer ($N_{UV}$) increases, the excitation light is significantly attenuated. For $N_{UV} = 20$, UV light is attenuated to 2%; for $N_{UV} = 25$, it is less than 1% [S1]. This limits the maximum depth of the display or requires stronger/dual-sided UV sources [S1].
- **Visible Light Attenuation:** The transmittance in the visible region is higher (~90%), but increasing the number of films between the printed layer and the camera ($N_{VIS}$) leads to a decrease in sharpness due to scattering/absorption, though less severe than UV attenuation [S1].
- **Image Quality Degradation:** Images viewed from diagonal directions (e.g., $\theta = \pm 30^\circ$) are blurred compared to the perpendicular view ($\theta = 0^\circ$) [S1].
- **Cross-talk:** Simulations show deterioration caused by other patterns (cross-talk), but the projected patterns remain recognizable as the original patterns [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources. The fabrication time depends on the inkjet printer speed and the number of layers.
- **Memory Complexity:** Not given in the sources. The data size is proportional to the number of layers and pixel resolution ($300 \times 300$ per layer in the prototype) [S1].
- **Measured Performance:**
    - Prototype: 20 layers, $0.5$ mm spacing, $35 \times 35 \times 12.5$ mm volume [S1].
    - Resolution: $300 \times 300$ pixels per layer in the prototype [S1].
    - Viewing Angles: Best quality at $\theta = 0^\circ$ (perpendicular). Blurred at $\theta = \pm 30^\circ$ [S1].
    - Multi-view: Successfully demonstrated 3 patterns at $\pm 30^\circ$ and 4 patterns at $\pm 20^\circ$ around X and Y axes [S1].
- **Scaling:** Increasing the number of layers improves depth resolution but exacerbates UV attenuation and visible light scattering [S1].
- **Strengths:**
    - Uses commercially available inkjet printers and transparent films [S1].
    - Capable of full-color 3D images [S1].
    - Supports multiple distinct 2D patterns from different angles (digital signage, security) [S1].
    - Preserves motion parallax [S1].
- **Limitations:**
    - Static display (no refresh rate, fixed image) [S1].
    - Blurring at oblique viewing angles [S1].
    - UV light absorption limits depth and requires careful illumination setup (dual-sided UV) [S1].
    - Cross-talk between projected patterns [S1].
- **When to use:** Applications requiring static, full-color 3D visualizations with motion parallax, such as digital signage, media art, entertainment, and security features [S1].
- **When not to use:** Applications requiring dynamic content, high-resolution oblique viewing, or large-scale depth without significant brightness loss [S1].

## Background
This method was introduced by Hirayama et al. in a 2017 paper published in *Scientific Reports* [S1]. It represents an advancement in volumetric displays by leveraging existing inkjet printing technology to create photoreactive luminescent volumes [S1]. The design algorithm for multi-view projection extends previous work by the same authors [13] [S1]. The technology is positioned as a novel information display device suitable for digital signage, media art, entertainment, and security [S1].

## Sources
[S1] Inkjet printing-based volumetric display projecting multiple full-colour 2D patterns., https://doi.org/10.1038/srep46511, fetched 2026-10-26
