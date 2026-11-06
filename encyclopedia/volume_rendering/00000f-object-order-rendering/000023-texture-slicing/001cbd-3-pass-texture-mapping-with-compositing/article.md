# 3-Pass Texture Mapping with Compositing

Also called: 3-pass version of 2-pass texture mapping with compositing

## Overview
This method accelerates the geometric transformation of volume data by pre-computing scalar values into intermediate 3-D datasets, thereby simplifying the image generation problem to the geometric transformation of two values per voxel (color and opacity) followed by compositing [S1, unofficial]. It distinguishes itself from standard 2-pass texture mapping by re-sampling separately in three orthogonal directions to reduce computational expense, allowing for rapid interactive manipulation of volume data [S1, unofficial].

## How it works
The method relies on the shared principles of [Texture slicing](25-texture-slicing.md), specifically using proxy geometry and hardware-supported blending [S1, unofficial][S3, book]. The core innovation is the decomposition of the transformation and compositing process into three distinct passes to optimize performance.

**Inputs and Outputs**
*   **Input**: A scalar volume dataset.
*   **Intermediate Data**: Two pre-computed 3-D datasets containing color and opacity values for each voxel [S1, unofficial].
*   **Output**: A 2-D composite image.

**Data Structures**
The volume data is represented as textures in graphics hardware. For this method, the scalar volume data is mapped to RGBA values via classification functions and stored in a texture map [S1, unofficial]. Specifically, the method utilizes intermediate 3-D datasets to store the pre-computed colors and opacities [S1, unofficial].

**Steps**
1.  **Pre-computation**: Pre-compute the colors and opacities for the volume data and store them in intermediate 3-D datasets [S1, unofficial]. This step simplifies the subsequent rendering passes by reducing the per-voxel computation to geometric transformations of these two values.
2.  **First Pass (Resampling X)**: Re-sample the intermediate 3-D datasets separately in the first orthogonal direction (e.g., X-axis). This involves geometrically transforming the voxel values [S1, unofficial].
3.  **Second Pass (Resampling Y)**: Re-sample the results from the first pass in the second orthogonal direction (e.g., Y-axis) [S1, unofficial].
4.  **Third Pass (Resampling Z and Compositing)**: Re-sample the results from the second pass in the third orthogonal direction (e.g., Z-axis) and composite the results to form the final image [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Omitting the Third Pass**: If a suitably interpolated sample array is available, the third pass may be omitted entirely. In this case, the transformed 2-D slices are composited together to form the image [S1, unofficial]. This optimization reduces computational expense further but requires specific interpolation properties in the sample array.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation phase
# Input: volume_data (3D array of scalars)
# Output: color_texture (3D texture), opacity_texture (3D texture)

def pre_compute_volume(volume_data):
    color_texture = create_3d_texture(volume_data.dimensions)
    opacity_texture = create_3d_texture(volume_data.dimensions)
    
    for each voxel (x, y, z) in volume_data:
        scalar = volume_data[x, y, z]
        # Apply classification function to get color and opacity
        rgba = classify(scalar)
        color_texture[x, y, z] = rgba.color
        opacity_texture[x, y, z] = rgba.opacity
        
    return color_texture, opacity_texture

# Rendering phase
# Input: color_texture, opacity_texture, view_transformation_matrix
# Output: final_image

def render_3_pass(color_texture, opacity_texture, view_matrix):
    # Pass 1: Resample along first orthogonal axis (e.g., X)
    # This transforms the 3D data into a 2D representation along the other axes
    intermediate_2d_x = resample_axis(color_texture, axis='X', transform=view_matrix)
    intermediate_opacity_x = resample_axis(opacity_texture, axis='X', transform=view_matrix)
    
    # Pass 2: Resample along second orthogonal axis (e.g., Y)
    intermediate_2d_y = resample_axis(intermediate_2d_x, axis='Y', transform=view_matrix)
    intermediate_opacity_y = resample_axis(intermediate_opacity_x, axis='Y', transform=view_matrix)
    
    # Pass 3: Resample along third orthogonal axis (e.g., Z) and composite
    # This step composites the results to form the final image
    final_image = composite_axis(intermediate_2d_y, intermediate_opacity_y, axis='Z', transform=view_matrix)
    
    return final_image

# Optimization: Omit third pass if suitably interpolated sample array is available
def render_2_pass_optimized(transformed_2d_slices):
    # Composite transformed 2-D slices together to form an image
    final_image = composite_slices(transformed_2d_slices)
    return final_image
```

## Subtopics

### Second-pass Surface Voxel Texture Mapping
This variant performs texture mapping during a second pass instead of outputting a typical 32-bit ARGB color [S2, unofficial]. The GPU is given the voxel position and a pointer to its surface structure [S2, unofficial]. The shader raytraces to a cube which it already knows it will hit, determining the face of the cube with a few simple cases [S2, unofficial]. The GPU then performs texture mapping with filters like bilinear and mip mapping [S2, unofficial]. The pixel layout is 64-bit: 12 bits for x, 12 bits for y, 12 bits for z, and a 28-bit index to the leaf node holding surface voxel information [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The method reduces computational expense by re-sampling separately in each of three orthogonal directions [S1, unofficial]. This decomposition allows for faster geometric transformations compared to full 3D resampling in a single pass.
*   **Memory Complexity**: Requires storage for intermediate 3-D datasets for both color and opacity [S1, unofficial].
*   **Strengths**: Enables rapid interactive manipulation of volume data [S1, unofficial]. It is an object-space approach, efficient for modern graphics processors, making it the choice for interactivity [S3, book].
*   **Limitations**: Requires pre-computation of colors and opacities, which adds an initial processing step [S1, unofficial]. The omission of the third pass is conditional on the availability of a suitably interpolated sample array [S1, unofficial].

## Background
The technique was proposed as a promising method for speeding up geometric transformations of volume data [S1, unofficial]. It builds on the concept of 2-pass texture mapping with compositing [S1, unofficial]. The author acknowledges a survey of architectures for rendering voxel data by Kaufman [S1, unofficial]. The method suggests that hardware implementations might be feasible and that a suitably interconnected 2-D array of processors with sufficient backing storage might be capable of producing visualizations of volume datasets in real-time or near real-time [S1, unofficial].

## Sources
[S1] DTIC ADA193649: Rendering of Surfaces from Volume Data, https://archive.org/details/DTIC_ADA193649, fetched 2026-10-26 (unofficial)
[S2] gpdaniels/pnd3d, https://github.com/gpdaniels/pnd3d, fetched 2026-10-27 (unofficial)
