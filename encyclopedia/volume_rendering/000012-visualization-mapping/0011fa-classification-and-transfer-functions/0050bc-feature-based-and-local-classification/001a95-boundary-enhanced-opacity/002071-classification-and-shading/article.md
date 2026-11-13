# Classification and Shading

Also called: Color Ramp Shading, Gradient-based Opacity Modulation, Gradient-based Volume Shading, Intensity Window Shading, Shading and Gradients

## Overview
Classification and Shading maps raw volume intensity values to colors and opacities to produce a realistic visualization of volumetric data [S1, unofficial][S2]. It solves the problem of interpreting scalar fields by combining material classification (color) with lighting models (shading) to reveal surface orientation and depth [S1, unofficial]. Unlike [Boundary-Enhanced Opacity](795-boundary-enhanced-opacity.md), which modulates opacity based on gradient magnitude to emphasize boundaries, this method primarily uses gradient-based shading to enhance surface appearance while maintaining uniform or segment-specific opacity [S1, unofficial][S2].

## How it works
**Inputs:**
- Volume data array $V$ containing scalar intensity values $x(i, j, k)$ [S1, unofficial][S2].
- Optional segmentation index array $S$ containing integer segment indices [S2].
- Parameters: $IntensityMin$, $IntensityMax$ defining the intensity window [S2].
- Textures: $ColorRampTexture$ (2D texture mapping intensity to color) [S2].
- Data: $SegmentColors$ array mapping segment indices to RGB colors [S2].

**Outputs:**
- Final color $C$ and opacity $\alpha$ for each voxel or sample point [S1, unofficial][S2].

**Data Structures:**
- Volume Buffer: A structured buffer or array storing float intensity values [S2].
- Color Ramp Texture: A 2D texture where the U-coordinate corresponds to normalized intensity [S2].
- Segment Colors: An array of RGB vectors indexed by segment ID [S2].

**Steps:**

1.  **Gradient Estimation:**
    Estimate the gradient vector $\nabla x(i, j, k)$ at each voxel using central finite differences [S1, unofficial]:
    $$
    \nabla x(i, j, k) = \frac{1}{2} \begin{bmatrix} x(i+1, j, k) - x(i-1, j, k) \\ x(i, j+1, k) - x(i, j-1, k) \\ x(i, j, k+1) - x(i, j, k-1) \end{bmatrix}
    $$
    [S1, unofficial]

2.  **Opacity Calculation (Gradient-Based):**
    Calculate opacity $\alpha(i, j, k)$ using the gradient magnitude and a scalar function $g(x)$ [S1, unofficial]:
    $$
    \alpha(i, j, k) = \frac{|\nabla x(i, j, k)|}{|\nabla x(i, j, k)|_g} \cdot g(x(i, j, k))
    $$
    where $|\nabla x(i, j, k)|_g$ is a normalization factor (typically the maximum gradient magnitude in the volume or a global constant) and $g(x)$ is a scalar piecewise linear function defining material transparency [S1, unofficial].

3.  **Intensity Normalization:**
    For shading purposes, normalize the intensity value $x$ to the range $[0, 1]$ using the intensity window [S2]:
    $$
    unitIntensity = \text{saturate}\left( \frac{x - IntensityMin}{IntensityMax - IntensityMin} \right)
    $$
    where $\text{saturate}(v) = \max(0, \min(1, v))$ [S2].

4.  **Color Mapping:**
    Determine the base color $C_{base}$ based on the shading method [S2]:
    - **Simple Shading:** Set $C_{base} = (unitIntensity, unitIntensity, unitIntensity, 1.0)$ [S2].
    - **Color Ramp Shading:** Sample the 2D texture $ColorRampTexture$ at UV coordinates $(unitIntensity, 0)$ [S2]:
      $$
      C_{base} = \text{tex2d}(ColorRampTexture, (unitIntensity, 0))
      $$

5.  **Segmentation Modulation (Optional):**
    If segmentation indices are available, modulate the color by the segment-specific color [S2]:
    $$
    C_{final} = C_{base}.rgb \cdot SegmentColors[segment\_index] \cdot C_{base}.a
    $$
    [S2]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_and_shade(volume, i, j, k, segment_index, intensity_min, intensity_max, 
                       color_ramp_texture, segment_colors, grad_mag_g, g_func):
    """
    Classifies and shades a single voxel value x.
    
    Args:
        volume: 3D array of scalar intensity values.
        i, j, k: Voxel coordinates.
        segment_index: Integer index for segmentation (optional).
        intensity_min: Lower bound of the intensity window.
        intensity_max: Upper bound of the intensity window.
        color_ramp_texture: 2D texture for color mapping.
        segment_colors: Array of RGB colors for segments.
        grad_mag_g: Normalization factor for gradient magnitude.
        g_func: Scalar piecewise linear function for opacity.
        
    Returns:
        color: RGBA color vector.
        opacity: Float opacity value.
    """
    # Step 1: Gradient Estimation [S1, unofficial]
    dx = 0.5 * (volume[i+1, j, k] - volume[i-1, j, k])
    dy = 0.5 * (volume[i, j+1, k] - volume[i, j-1, k])
    dz = 0.5 * (volume[i, j, k+1] - volume[i, j, k-1])
    grad_mag = (dx**2 + dy**2 + dz**2)**0.5

    # Step 2: Opacity Calculation [S1, unofficial]
    alpha = (grad_mag / grad_mag_g) * g_func(volume[i, j, k])

    # Step 3: Intensity Normalization [S2]
    intensity = volume[i, j, k]
    unit_intensity = max(0.0, min(1.0, (intensity - intensity_min) / (intensity_max - intensity_min)))

    # Step 4: Color Mapping [S2]
    # Color ramp shading overrides simple shading
    color = color_ramp_texture.sample(unit_intensity, 0.0)

    # Step 5: Segmentation Modulation [S2]
    if segment_index is not None:
        seg_color = segment_colors[segment_index]
        color = [color[0] * seg_color[0] * color[3],
                 color[1] * seg_color[1] * color[3],
                 color[2] * seg_color[2] * color[3],
                 color[3]]

    return color, alpha
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### [Gradient-Based Boundary Classification](2571-gradient-based-boundary-classification.md)
Gradient-Based Boundary Classification determines whether a voxel lies on the boundary of a material segment by classifying it as "reflecting" (surface) or "ambient" (interior/exterior). It interprets the gradient magnitude in the context of the voxel's data value and the material's classification range, rather than relying on gradient magnitude in isolation. This approach departs from earlier methods [Lev90a, LL94] by using a linear approximation of the field function to estimate the distance to the material boundary, allowing for a probabilistic classification that handles noise and nonlinearity.

### Lighting Transfer Function for Boundary Enhancement
A method using lighting to selectively enhance boundary surfaces of interest while using uniform opacity to illustrate material thickness, independent of opacity modulation [S3]. Unlike traditional opacity-based methods where boundaries are visible only via opacity changes, this method keeps opacity uniform and uses lighting to make boundaries visible even at homogeneous scalar value transitions [S3]. Unlike uniform application of shading which adds confusion in homogeneous regions, this method applies lighting only at select density transitions [S3]. It requires approximately twice as many texture reads as traditional opacity-based transfer function volume rendering methods [S3]. On an Nvidia GeForce 6800, it achieves 7.3 fps for a 256x256x256 volume rendered to a 512x512 buffer, compared to 13.5 fps with uniform lighting [S3].

## Operating characteristics
- **Time Complexity:** The gradient-based opacity model requires computing gradients for all voxels, which is $O(N)$ where $N$ is the number of voxels, if precomputed [S1, unofficial]. During ray marching, shading adds computational cost per sample [S1, unofficial]. The Lighting Transfer Function method requires approximately twice as many texture reads as traditional methods [S3].
- **Memory Complexity:** Requires storage for the volume data, optional segmentation indices, and color ramp textures [S2].
- **Performance:** On an Nvidia GeForce 6800, rendering a 256x256x256 volume to a 512x512 buffer with sample spacing equal to the original volume yields 7.3 fps with Lighting Transfer Functions, compared to 13.5 fps with uniform lighting [S3].
- **Strengths:** Provides realistic surface appearance and depth cues through shading [S1, unofficial]. Segmentation allows for distinct material visualization [S2]. Lighting Transfer Functions enhance boundaries without losing thickness cues from uniform opacity [S3].
- **Limitations:** Lighting Transfer Functions have higher computational cost due to the increased texture read count [S3].

## Background
Classification and Shading is a fundamental technique in volume rendering that maps voxel values to color and opacity to emulate real-world viewing [S1, unofficial]. The gradient-based opacity model was introduced in early work on direct volume rendering, utilizing discrete differences for gradient estimation [S1, unofficial]. The concept of intensity windowing and color ramp shading is widely used in medical imaging and mixed reality applications for visualizing scalar fields like MRI data [S2]. Lighting Transfer Functions were developed to address the limitations of opacity-based boundary enhancement, providing a way to highlight surfaces while preserving depth cues from material thickness [S3].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
[S3] PDF Lighting Transfer Functions Using Gradient Aligned Sampling, https://www.cs.ucdavis.edu/~ma/papers/lightingtf.pdf, fetched 2026-10-29
