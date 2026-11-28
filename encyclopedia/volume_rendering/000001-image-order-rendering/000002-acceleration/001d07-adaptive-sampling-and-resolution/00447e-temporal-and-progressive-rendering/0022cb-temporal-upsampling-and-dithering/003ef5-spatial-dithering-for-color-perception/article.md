# Spatial Dithering for Color Perception

Also called: Half-toning / dithering methods, SLM Dithering

## Overview
Spatial dithering for color perception is a technique used in volumetric displays with single-bit-depth spatial light modulators (SLMs) to create the illusion of multi-bit color depth. It solves the problem of limited hardware color resolution by distributing quantization error spatially across the projection screen, allowing the human eye to perceive a wider color gamut than the physical device can natively produce. Unlike temporal methods that rely on frame sequencing, this approach operates within the spatial domain of a single frame or slice, often in conjunction with hardware-specific constraints like 1-bit MEMS mirrors.

## How it works
The method is implemented in systems utilizing a multi-SLM projection engine where each SLM has a 1-bit depth. The core process involves processing OpenGL pipeline data through proprietary rasterization and half-toning/dithering algorithms before projection.

**Inputs and Outputs:**
*   **Input:** Standard OpenGL rendering data (e.g., voxel colors, densities).
*   **Output:** 1-bit depth frames for each SLM channel (Red, Green, Blue).

**Data Structures:**
*   **SLM Frame Buffer:** A binary array corresponding to the SLM resolution (e.g., $768 \times 768$ pixels). Each element is a single bit representing the state of a micro-mirror (on/off).
*   **Color Map:** A mapping from voxel density/color values to target RGB intensities.

**Steps:**
1.  **Color Quantization:** Map the desired continuous or high-bit-depth color value for each voxel to a target intensity for each of the R, G, and B channels.
2.  **Dithering Application:** Apply a proprietary half-toning or dithering algorithm to the target intensity values. This algorithm distributes the quantization error to neighboring pixels or sub-pixels to preserve the average perceived brightness and color.
3.  **Binary Conversion:** Convert the dithered intensity values into binary states (0 or 1) for the SLM.
4.  **Projection:** Project the 1-bit images from the R, G, and B SLMs onto a rotating omnidirectional diffuser screen using a color-mixing prism.

**Parameters:**
*   **SLM Bit Depth:** 1-bit per SLM [S1, S2].
*   **Native Color Depth:** 3-bit per voxel (1-bit per channel) [S2].
*   **Perceived Color Depth:** Full color (via spatial dithering) [S1, S2].
*   **SLM Resolution:** $768 \times 768$ pixels [S1, S2].
*   **Update Rate:** Approximately 5 kHz [S1] or 168 µs per frame [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** The perceived color quality depends heavily on the dithering algorithm's ability to minimize visible artifacts (e.g., moiré patterns) while maintaining color accuracy.
*   **Hardware Constraints:** The method is tightly coupled with the specific hardware (e.g., Texas Instruments DLP technology) and optical setup (rotating screen, Lambertian diffuser) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatial_dithering_for_color_perception(voxel_colors, slm_width, slm_height):
    """
    Converts high-bit-depth voxel colors to 1-bit SLM frames using spatial dithering.
    
    Args:
        voxel_colors: List of (x, y, z, r, g, b) tuples representing voxel data.
        slm_width: Width of the SLM in pixels.
        slm_height: Height of the SLM in pixels.
        
    Returns:
        r_frame, g_frame, b_frame: 2D binary arrays for R, G, B SLMs.
    """
    # Initialize binary frames
    r_frame = [[0 for _ in range(slm_width)] for _ in range(slm_height)]
    g_frame = [[0 for _ in range(slm_width)] for _ in range(slm_height)]
    b_frame = [[0 for _ in range(slm_width)] for _ in range(slm_height)]
    
    # Note: The specific proprietary dithering algorithm is not disclosed in the sources.
    # The following steps represent the logical flow based on the description.
    
    for voxel in voxel_colors:
        x, y, z, r, g, b = voxel
        
        # 1. Map voxel color to target intensities for each channel
        # This step may involve density-to-color mapping and gamma correction
        target_r = map_to_intensity(r)
        target_g = map_to_intensity(g)
        target_b = map_to_intensity(b)
        
        # 2. Apply proprietary dithering/halftoning
        # The error is distributed spatially to create the perception of intermediate colors
        dithered_r = apply_dithering(target_r, x, y, r_frame)
        dithered_g = apply_dithering(target_g, x, y, g_frame)
        dithered_b = apply_dithering(target_b, x, y, b_frame)
        
        # 3. Convert to binary states for SLM
        r_frame[y][x] = 1 if dithered_r > 0.5 else 0
        g_frame[y][x] = 1 if dithered_g > 0.5 else 0
        b_frame[y][x] = 1 if dithered_b > 0.5 else 0
        
    return r_frame, g_frame, b_frame

def apply_dithering(target_intensity, x, y, frame):
    """
    Placeholder for the proprietary dithering algorithm.
    In a real implementation, this would handle error diffusion or ordered dithering.
    """
    # Not given in the sources. The specific algorithm is proprietary.
    return target_intensity
```

## Subtopics

### Perspecta Volumetric Dithering
A proprietary algorithm used in Perspecta volumetric displays that maps 3-bit color per voxel to full perceived color through dithering and quasi-halftoning on a rotating omnidirectional diffuser screen. It is distinguished by its integration with the Perspecta hardware platform and its use of a rotating screen to achieve volumetric rendering. It is chosen when using Perspecta's DLP-based microdisplay chips with 768x768 resolution and 168 µs update rates. The native color depth is 3-bit per voxel, but the perceived color depth is full color due to the dithering. [S2]

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The system operates in real-time, supporting OpenGL-based applications.
*   **Memory Complexity:** Not given in the sources. The SLM resolution is $768 \times 768$ pixels.
*   **Measured Performance:** The system can project approximately 200 slices of $768 \times 768$-pixel 2-D images to create a volume-filling 3-D scene at 5 kHz [S1].
*   **Scaling:** The method scales with the SLM resolution and update rate. Higher resolutions or frame rates may require more powerful processing for the dithering algorithm.
*   **Strengths:** Enables multi-bit color perception from 1-bit hardware, reducing hardware complexity and cost.
*   **Limitations:** The perceived color quality is dependent on the dithering algorithm and viewing conditions. The method is specific to certain hardware configurations (e.g., rotating screens, DLP technology).
*   **When to use:** When using volumetric displays with 1-bit SLMs and needing to render multi-bit color content.
*   **When not to use:** When high-fidelity, artifact-free color is required without the trade-offs of dithering, or when using hardware with native multi-bit depth.

## Background
The method was developed for volumetric displays using Texas Instruments Digital Mirror Device (DMD) technology. The first operational system was demonstrated in March 2001, capable of projecting 100 million voxels [S1]. The Perspecta system, described in 2005, further refined this approach with proprietary dithering algorithms [S2]. The technology is used in medical imaging (e.g., CT, MRI visualization), military imaging, and mathematical modeling.

## Sources
[S1] 100-million-voxel volumetric display, https://doi.org/10.1117/12.480930, fetched 2026-11-01
[S2] Volumetric 3D displays and application infrastructure, https://doi.org/10.1109/mc.2005.276, fetched 2026-11-01
