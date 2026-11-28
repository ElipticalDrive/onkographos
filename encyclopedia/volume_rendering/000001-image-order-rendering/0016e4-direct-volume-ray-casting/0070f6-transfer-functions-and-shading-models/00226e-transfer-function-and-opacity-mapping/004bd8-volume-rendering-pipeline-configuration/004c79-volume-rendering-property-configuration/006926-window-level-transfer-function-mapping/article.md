# Window/Level Transfer Function Mapping

Also called: Piecewise color/alpha mapping, W/L CT/MRI windowing, Window and Level Mapping

## Overview
Window/Level Transfer Function Mapping is a method for configuring volume rendering properties by mapping a selected range of scalar voxel intensities (the "window") to a range of displayed grayscale or color values (the "level") to improve contrast for specific tissue densities [S1, unofficial]. It sets apart from general transfer function editors by using a fixed 3-point piecewise linear model controlled by two geometric parameters (window width and center level) rather than arbitrary control points [S2, unofficial]. It builds on [Volume Rendering Property Configuration](13233-volume-rendering-property-configuration.md).

## How it works
**Inputs:**
- Scalar voxel intensity values (typically 16-bit signed or unsigned integers).
- Window width ($W$) and Window level ($L$) parameters.
- Opacity scalar ($\alpha_{max}$), typically 1.0.
- Color palette (default grayscale).
- Invert flag (boolean).

**Outputs:**
- RGBA transfer function definition (color and opacity values for any input intensity).
- Scalar opacity unit distance (set to minimum voxel spacing).

**Data Structures:**
The method constructs two piecewise linear functions:
1.  **Color Transfer Function:** Defined by 3 points $(x, r, g, b)$.
2.  **Opacity Transfer Function:** Defined by 3 points $(x, \alpha)$.

**Steps:**
1.  Compute the lower bound ($low$) and upper bound ($high$) of the window:
    $$ low = L - \frac{W}{2} $$
    $$ high = L + \frac{W}{2} $$
2.  Define the three control points for the transfer functions at intensities $low$, $L$, and $high$.
3.  Assign color values based on the palette. For grayscale, the color intensity is linearly interpolated between black (at $low$) and white (at $high$). If the invert flag is true, the palette is reversed (white at $low$, black at $high$) [S2, unofficial].
4.  Assign opacity values:
    - At $low$: $\alpha = 0$
    - At $L$: $\alpha = \alpha_{max} \times 0.15$
    - At $high$: $\alpha = \alpha_{max}$
5.  Set the scalar opacity unit distance to the minimum spacing value among the three axes to preserve physical scale [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_window_level_tf(intensity, window_width, level, opacity_max=1.0, invert=False, palette=None):
    """
    Computes RGBA value for a given intensity using window/level mapping.
    
    Args:
        intensity: float, voxel intensity value.
        window_width: float, W.
        level: float, L.
        opacity_max: float, maximum opacity (default 1.0).
        invert: bool, if True, reverse grayscale mapping.
        palette: list of RGB tuples, default is grayscale.
        
    Returns:
        tuple: (r, g, b, a)
    """
    # 1. Compute bounds
    low = level - window_width / 2.0
    high = level + window_width / 2.0
    
    # 2. Clamp intensity to window bounds for linear interpolation
    # Intensities outside [low, high] map to 0 opacity
    if intensity < low or intensity > high:
        return (0.0, 0.0, 0.0, 0.0)
    
    # 3. Linear interpolation factor t in [0, 1]
    t = (intensity - low) / (high - low)
    
    # 4. Determine color
    # Default grayscale palette: low=black, high=white
    if palette is None:
        if invert:
            # Inverted grayscale: low=white, high=black
            color_val = 1.0 - t
        else:
            # Standard grayscale: low=black, high=white
            color_val = t
        r = color_val
        g = color_val
        b = color_val
    else:
        # Interpolate within provided palette (simplified for 2-point palette)
        # Assuming palette has at least 2 entries: [color_low, color_high]
        c_low = palette[0]
        c_high = palette[1]
        r = c_low[0] + t * (c_high[0] - c_low[0])
        g = c_low[1] + t * (c_high[1] - c_low[1])
        b = c_low[2] + t * (c_high[2] - c_low[2])
        
    # 5. Determine opacity
    # Points: (low, 0), (level, opacity_max * 0.15), (high, opacity_max)
    if intensity <= level:
        # Interpolate between low and level
        # t_local goes from 0 to 0.5 as intensity goes from low to level
        t_local = (intensity - low) / (level - low) if level != low else 0.0
        alpha = 0.0 + t_local * (opacity_max * 0.15 - 0.0)
    else:
        # Interpolate between level and high
        # t_local goes from 0 to 1 as intensity goes from level to high
        t_local = (intensity - level) / (high - level) if high != level else 0.0
        alpha = (opacity_max * 0.15) + t_local * (opacity_max - opacity_max * 0.15)
        
    return (r, g, b, alpha)
```

**Parameters:**
- $W$ (Window Width): Controls the range of intensities mapped to the full display range. Typical values depend on tissue type (e.g., 300-400 for soft tissue, 1500-2000 for bone in CT).
- $L$ (Window Level): The center of the intensity range.
- $\alpha_{max}$: Maximum opacity, typically 1.0.

**Edge Cases:**
- If $W=0$, the window is undefined; clamp to a minimum positive value or return zero opacity.
- If $L$ is outside the data range, the window may map to empty space; ensure bounds calculation handles this gracefully.
- Precision issues may arise if $high \approx low$; use epsilon checks in division.

## Subtopics

### [VIPER Volume Rendering Configuration](15683-viper-volume-rendering-configuration.md)
A more complex configuration method allowing arbitrary control points and multiple transfer functions. Choose when fine-grained control over specific intensity ranges is needed beyond simple window/level.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel for transfer function evaluation.
- **Memory Complexity:** $O(1)$ for storing the 3-point transfer function definition.
- **Strengths:** Simple, intuitive for radiologists; computationally cheap; effective for highlighting specific tissue types in CT/MRI.
- **Limitations:** Limited flexibility compared to multi-point piecewise functions; cannot model complex opacity variations (e.g., multiple peaks); appearance depends heavily on source data quality [S1, unofficial].
- **When to use:** Standard clinical viewing of CT/MRI where quick contrast adjustment is required.
- **When not to use:** When complex material differentiation (e.g., separating fat, muscle, bone simultaneously with distinct colors) is needed; use general piecewise transfer functions instead.

## Background
Originating from standard radiological display practices, window/level mapping is adapted for volume rendering to provide a familiar interface for medical professionals. It is implemented in various open-source medical imaging viewers such as Sinus Viewer [S1, unofficial] and MRI3D [S2, unofficial]. The method simplifies the transfer function configuration to two parameters, aligning with the standard DICOM window/level attributes.

## Sources
[S1] sarfrazkhan66003/sinus-3d-viewer-pro, https://github.com/sarfrazkhan66003/sinus-3d-viewer-pro, fetched 2026-10-26 (unofficial)
[S2] kadmielp/MRI3D, https://github.com/kadmielp/MRI3D, fetched 2026-10-26 (unofficial)
