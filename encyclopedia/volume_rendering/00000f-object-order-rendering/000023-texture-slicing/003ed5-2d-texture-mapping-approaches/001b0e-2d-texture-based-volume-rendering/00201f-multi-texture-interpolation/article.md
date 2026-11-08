# Multi-Texture Interpolation

Also called: Multi-Texture Blending, Multi-Texture Trilinear Interpolation

## Overview
Multi-Texture Interpolation is a technique for 2D texture-based volume rendering that eliminates visual artifacts caused by a fixed number of pre-rendered slices by computing intermediate slices on the fly [S1]. It builds on [2D Texture-Based Volume Rendering](916-2d-texture-based-volume-rendering.md) by leveraging multi-texture hardware combiners to perform real trilinear interpolation, distinguishing itself from siblings like [Novel Sampling Scheme for Slice Based Volume Rendering](953-novel-sampling-scheme-for-slice-based-volume-rendering.md) by relying on fixed hardware combiners rather than custom sampling logic [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric data stored as a sequence of 2D texture images (slices) $S_i$.
*   **Output:** A rendered image with semi-transparent blending, where intermediate slices are interpolated between fixed slices.

**Data Structures:**
*   **Textures:** Each volume slice is stored in a separate 2D texture unit. Texture 0 holds slice $S_i$, and Texture 1 holds slice $S_{i+1}$ [S1].
*   **Combiner Registers:**
    *   **Constant Color Register:** Stores the interpolation factor $\alpha$ [S1].
    *   **Input Registers (A, B, C, D):** Configured to map texture components and the interpolation factor to the combiner's arithmetic unit [S1].

**Numbered Steps:**
1.  **Texture Assignment:** Assign the fixed slice $S_i$ to Texture 0 and the adjacent fixed slice $S_{i+1}$ to Texture 1 using the multi-texture extension [S1].
2.  **Combiner Setup:** Configure General Combiner 0 to compute a component-wise weighted sum $AB + CD$ [S1].
    *   Map the interpolation factor $\alpha$ (from the constant color register) to input variable $A$ [S1].
    *   Map the inverted interpolation factor $(1 - \alpha)$ to input variable $C$ (derived by inverting $A$) [S1].
    *   Map the RGB components of Texture 0 to input variable $B$ [S1].
    *   Map the RGB components of Texture 1 to input variable $D$ [S1].
    *   Apply the same mapping logic to the Alpha components [S1].
3.  **Interpolation Calculation:** The hardware computes the intermediate slice $S_{i+}$ for both RGB and Alpha channels in a single clock cycle [S1].
4.  **Blending:** The output of the first combiner stage is used directly for back-to-front alpha blending without modification by the final combiner stage [S1].

**Formulas:**
The intermediate slice $S_{i+}$ is computed as a weighted sum of adjacent fixed slices $S_i$ and $S_{i+1}$ [S1]:

$$ S_{i+} = (1 - \alpha) \times S_i + \alpha \times S_{i+1} $$

Where:
*   $S_{i+}$ is the interpolated slice.
*   $S_i$ is the RGB/Alpha data of the slice at Texture 0.
*   $S_{i+1}$ is the RGB/Alpha data of the slice at Texture 1.
*   $\alpha$ is the interpolation factor, stored in the constant color register [S1].

**Parameters:**
*   **Opacity Scaling:** Opacity values must be adapted according to the new slice distance. This is approximated by a constant linear scale factor, similar to the 3D-texture based approach [S1].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Opacity Adaptation:** Failure to adapt opacity values according to the new slice distance will result in incorrect visual density [S1].
*   **Precision:** The method relies on hardware combiners performing the operation in one clock cycle; precision is limited by the hardware's combiner implementation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Setup textures
texture_0 = load_slice(S_i)
texture_1 = load_slice(S_i_plus_1)

# Set interpolation factor alpha in constant color register
alpha = calculate_interpolation_factor(viewing_ray_position, slice_distance)

# Configure General Combiner 0
# Input A: alpha (from constant color register)
# Input B: RGB of texture_0
# Input C: 1 - alpha (inverted A)
# Input D: RGB of texture_1
# Operation: AB + CD

# The hardware computes:
# RGB_interpolated = alpha * texture_0.RGB + (1 - alpha) * texture_1.RGB
# Alpha_interpolated = alpha * texture_0.Alpha + (1 - alpha) * texture_1.Alpha

# Scale opacity based on slice distance
opacity_scale = constant_linear_scale_factor
Alpha_interpolated = Alpha_interpolated * opacity_scale

# Output is used for back-to-front alpha blending
render_blend(RGB_interpolated, Alpha_interpolated)
```

## Subtopics

### [Multi-Texture Performance Enhancement](4543-multi-texture-performance-enhancement.md)
Accelerates rendering by mapping $n$ consecutive slices onto a single polygon using $n$ multi-texture units, reducing draw calls and frame buffer reads by a factor of $1/n$.

### [Multi-Texture Pipeline Rendering](4945-multi-texture-pipeline-rendering.md)
Uses multiple 2D texture units in a pipeline to allow separate updates for different components (e.g., color, light map) at different resolutions.

## Operating characteristics
*   **Time Complexity:** An intermediate slice is rendered at almost the same performance as a fixed single-textured slice because multi-texture interpolation and combination are performed within one clock cycle of the graphics CPU [S1].
*   **Memory Complexity:** Multiple intermediate slices can be inserted without any increase in memory size, as they are computed on the fly from the fixed slices [S1].
*   **Strengths:** Removes visual artifacts caused by the fixed number of slices; enhances image quality by enabling real trilinear interpolation [S1].
*   **Limitations:** Requires hardware support for multi-texture extensions and general combiners [S1].
*   **When to use:** When high image quality is required and the hardware supports multi-texturing, but 3D textures are not available or insufficient [S1].
*   **When not to use:** When hardware does not support multi-texture combiners or when memory bandwidth is not the primary bottleneck [S1].

## Background
Multi-Texture Interpolation was introduced to enhance the image quality of 2D-texture-based volume rendering by removing visual artifacts caused by a fixed number of slices [S1]. It enables real trilinear interpolation by computing intermediate slices on the fly, with the third interpolation step performed within the rasterization hardware using multi-textures [S1]. The method was developed for standard PC graphics hardware to provide an alternative to 3D-texture based approaches [S1].

## Sources
[S1] Interactive Volume Rendering on Standard PC Graphics Hardware, https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
