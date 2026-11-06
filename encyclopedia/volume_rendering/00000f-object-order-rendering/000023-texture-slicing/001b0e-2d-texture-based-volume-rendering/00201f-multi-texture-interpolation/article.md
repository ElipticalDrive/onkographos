# Multi-Texture Interpolation

Also called: Multi-Texture Blending, Multi-Texture Trilinear Interpolation

## Overview
Multi-Texture Interpolation is a technique for 2D texture-based volume rendering that eliminates visual artifacts caused by a fixed number of discrete slices [S1]. It enables real trilinear interpolation by computing intermediate slices on the fly using multi-texture hardware combiners [S1]. Unlike standard fixed-slice approaches that produce staircase or banding artifacts, this method retains the standard slice projection and enhances the interpolation step between adjacent slices using GPU combiners [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric data stored as a sequence of 2D texture images, where each image represents a fixed slice $S_i$ of the volume [S1].
*   **Output:** A rendered image with semi-transparent blending, where the interpolation between slices is continuous rather than discrete [S1].

**Data Structures:**
*   **Textures:** Each fixed slice $S_i$ is stored in a separate 2D texture unit; Texture 0 holds $S_i$ and Texture 1 holds $S_{i+1}$ [S1].
*   **Combiner Registers:** The GPU's multi-texture extension provides combiner stages. The setup uses a constant color register to store the interpolation factor $\alpha$ [S1]. Input variables for General Combiner 0 are defined as: $A$ (interpolation factor $\alpha$), $B$ (RGB components of Texture 0), $C$ (inverted $A$, i.e., $1-\alpha$), and $D$ (RGB components of Texture 1) [S1].

**Steps:**
1.  **Texture Binding:** Bind the two adjacent fixed slices, $S_i$ and $S_{i+1}$, to Texture Unit 0 and Texture Unit 1, respectively [S1].
2.  **Bilinear Interpolation:** The texture hardware automatically performs bilinear interpolation on the texels of each individual slice based on the fragment's texture coordinates $(s, t)$ [S1]. This yields two interpolated texels, one from each slice [S1].
3.  **Combiner Setup:** Configure General Combiner 0 to compute a component-wise weighted sum $AB + CD$ [S1]. Map $\alpha$ to input variable $A$, $(1-\alpha)$ to input variable $C$, the RGB output of Texture 0 to input variable $B$, and the RGB output of Texture 1 to input variable $D$ [S1].
4.  **Intermediate Slice Computation:** The combiner computes the intermediate slice $S_{i+}$ by blending the two bilinearly interpolated texels from the fixed slices [S1].
5.  **Alpha Blending:** The output of the first combiner stage (containing the interpolated RGB and Alpha) is passed directly to the final combiner stage for back-to-front alpha blending without further modification [S1].

**Formulas:**
The intermediate slice $S_{i+}$ is computed as a weighted sum of the adjacent fixed slices $S_i$ and $S_{i+1}$ [S1]:

$$ S_{i+} = (1 - \alpha) \times S_i + \alpha \times S_{i+1} $$

Where:
*   $S_{i+}$ is the interpolated slice value.
*   $S_i$ is the value of the previous fixed slice.
*   $S_{i+1}$ is the value of the next fixed slice.
*   $\alpha$ is the interpolation factor, representing the relative distance of the virtual slice between $S_i$ and $S_{i+1}$ [S1].

The combiner operation is defined as $AB + CD$ [S1].

**Parameters:**
*   **Interpolation Factor ($\alpha$):** Determines the position of the intermediate slice between $S_i$ and $S_{i+1}$ [S1].
*   **Opacity Scale Factor:** Opacity values must be adapted according to the new slice distance, approximated by a constant linear scale factor [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Conceptual setup for multi-texture interpolation combiner
def compute_intermediate_slice(alpha, tex0_rgb, tex1_rgb, tex0_a, tex1_a):
    # alpha: interpolation factor from constant color register
    # tex0_rgb, tex1_rgb: RGB components from Texture 0 and Texture 1 after hardware bilinear interpolation
    # tex0_a, tex1_a: Alpha components from Texture 0 and Texture 1 after hardware bilinear interpolation
    
    # General Combiner 0 computes component-wise weighted sum AB + CD
    # A = alpha, B = tex0_rgb, C = 1 - alpha, D = tex1_rgb
    interpolated_rgb = alpha * tex0_rgb + (1.0 - alpha) * tex1_rgb
    
    # Alpha portion interpolates similarly
    interpolated_alpha = alpha * tex0_a + (1.0 - alpha) * tex1_a
    
    # Output is passed directly to final combiner for back-to-front alpha blending
    return interpolated_rgb, interpolated_alpha

# Post-processing step for opacity adaptation
def adjust_opacity(interpolated_alpha, slice_distance, original_slice_distance):
    # Opacity values must be adapted according to the new slice distance
    # Approximated by a constant linear scale factor
    scale_factor = original_slice_distance / slice_distance
    return interpolated_alpha * scale_factor
```

**Edge Cases and Pitfalls:**
*   **Opacity Scaling:** If opacity values are not adapted to the new slice distance, visual artifacts may occur [S1]. A constant linear scale factor is used as an approximation [S1].
*   **Hardware Dependency:** The method requires specific multi-texture hardware combiners supporting the $AB+CD$ operation [S1].

## Subtopics

### [Multi-Texture Performance Enhancement](4543-multi-texture-performance-enhancement.md)
Maps $n$ consecutive slices onto a single polygon using $n$ texture units to reduce draw calls and frame buffer reads. Choose when rendering speed is critical and slice counts are fixed. Builds on Multi-Texture Interpolation by addressing performance bottlenecks.

### [Multi-Texture Pipeline Rendering](4945-multi-texture-pipeline-rendering.md)
Chains multiple 2D texture units to enhance accuracy and allow separate updates for different components (e.g., color, light map). Choose when component-level resolution control is needed. Builds on Multi-Texture Interpolation by utilizing hardware chaining.

## Operating characteristics
*   **Performance:** An intermediate slice is rendered at almost the same performance as a fixed single-textured slice because multi-texture interpolation and combination are performed within one clock cycle [S1].
*   **Memory:** No increase in memory size is required to insert multiple intermediate slices, as they are computed on the fly [S1].
*   **Image Quality:** Greatly enhances image quality by removing visual artifacts caused by the fixed number of slices [S1].
*   **Limitations:** Requires hardware support for multi-texture extensions and combiners; the opacity adjustment is a linear approximation that may not be perfectly accurate for all volume datasets [S1].

## Background
Multi-Texture Interpolation was introduced as an approach to enable real trilinear interpolation for 2D-texture based rendering [S1]. It addresses the limitation of fixed slice counts by computing intermediate slices on the fly, leveraging the multi-texture hardware combiners available in standard PC graphics hardware [S1].

## Sources
[S1] Interactive Volume Rendering on Standard PC Graphics Hardware, https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
