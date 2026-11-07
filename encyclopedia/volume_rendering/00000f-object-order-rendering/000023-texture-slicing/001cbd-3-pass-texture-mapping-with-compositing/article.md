# 3-Pass Texture Mapping with Compositing

Also called: 3-pass version of 2-pass texture mapping with compositing

## Overview
This method accelerates the geometric transformation of volume data by pre-computing scalar values into intermediate 3-D datasets, thereby simplifying the image generation problem to the geometric transformation of two values per voxel (color and opacity) followed by compositing [S1, unofficial]. It distinguishes itself from standard 2-pass texture mapping by re-sampling separately in three orthogonal directions to reduce computational expense, allowing for rapid interactive manipulation of volume data [S1, unofficial].

## How it works
Builds on [Texture slicing](25-texture-slicing.md) [G].

**Inputs and Outputs**
*   **Input**: A scalar volume dataset.
*   **Intermediate Data**: Two pre-computed 3-D datasets containing color and opacity values for each voxel [S1, unofficial].
*   **Output**: A 2-D composite image.

**Data Structures**
The scalar volume data is mapped to RGBA values via classification functions and stored in intermediate 3-D datasets [S1, unofficial].

**Steps**
1.  **Pre-computation**: Pre-compute the colors and opacities for the volume data and store them in intermediate 3-D datasets [S1, unofficial]. This simplifies the image generation problem to one of geometrically transforming two values per voxel and compositing the results [S1, unofficial].
2.  **First Pass (Resampling X)**: Re-sample the intermediate 3-D datasets separately in the first orthogonal direction [S1, unofficial].
3.  **Second Pass (Resampling Y)**: Re-sample the results from the first pass in the second orthogonal direction [S1, unofficial].
4.  **Third Pass (Resampling Z and Compositing)**: Re-sample the results from the second pass in the third orthogonal direction and composite the results to form the final image [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Omitting the Third Pass**: If a suitably interpolated sample array is available, the third pass may be omitted entirely [S1, unofficial]. In this case, the transformed 2-D slices are composited together to form the image [S1, unofficial]. This optimization reduces computational expense further but requires specific interpolation properties in the sample array [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pre_compute(volume_data):
    # Create intermediate 3-D datasets for color and opacity [S1, unofficial]
    color_data = [None] * len(volume_data)
    opacity_data = [None] * len(volume_data)
    for i, voxel in enumerate(volume_data):
        # Classification step is not given in sources
        color_data[i], opacity_data[i] = None, None
    return color_data, opacity_data

def render_3_pass(color_data, opacity_data, transform):
    # Pass 1: Resample along first orthogonal direction [S1, unofficial]
    # Implementation of resampling along axis 0 is not given in sources
    intermediate_1 = None
    # Pass 2: Resample along second orthogonal direction [S1, unofficial]
    # Implementation of resampling along axis 1 is not given in sources
    intermediate_2 = None
    # Pass 3: Resample along third orthogonal direction and composite [S1, unofficial]
    # Implementation of resampling along axis 2 and compositing is not given in sources
    final_image = None
    return final_image

def render_2_pass_optimized(transformed_2d_slices):
    # If a suitably interpolated sample array is available, omit third pass [S1, unofficial]
    # Composite transformed 2-D slices together to form an image
    # Implementation of compositing 2-D slices is not given in sources
    final_image = None
    return final_image
```

## Subtopics

### Second-pass Surface Voxel Texture Mapping
This variant performs texture mapping during a second pass instead of outputting a typical 32-bit ARGB color [S2, unofficial]. The GPU is given the voxel position and a pointer to its surface structure [S2, unofficial]. The shader raytraces to a cube which it already knows it will hit, determining the face of the cube with a few simple cases [S2, unofficial]. The GPU then performs texture mapping with filters like bilinear and mip mapping [S2, unofficial]. The pixel layout is 64-bit: 12 bits for x, 12 bits for y, 12 bits for z, and a 28-bit index to the leaf node holding surface voxel information [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The method reduces computational expense by re-sampling separately in each of three orthogonal directions [S1, unofficial]. This decomposition allows for faster geometric transformations compared to full 3D resampling in a single pass [S1, unofficial].
*   **Memory Complexity**: Requires storage for intermediate 3-D datasets for both color and opacity [S1, unofficial].
*   **Strengths**: Enables rapid interactive manipulation of volume data [S1, unofficial].
*   **Limitations**: Requires pre-computation of colors and opacities, which adds an initial processing step [S1, unofficial]. The omission of the third pass is conditional on the availability of a suitably interpolated sample array [S1, unofficial].

## Background
The technique was proposed as a promising method for speeding up geometric transformations of volume data [S1, unofficial]. It builds on the concept of 2-pass texture mapping with compositing [S1, unofficial]. The author acknowledges a survey of architectures for rendering voxel data by Kaufman [S1, unofficial]. The method suggests that hardware implementations might be feasible and that a suitably interconnected 2-D array of processors with sufficient backing storage might be capable of producing visualizations of volume datasets in real-time or near real-time [S1, unofficial].

## Sources
[S1] DTIC ADA193649: Rendering of Surfaces from Volume Data, https://archive.org/details/DTIC_ADA193649, fetched 2026-10-26 (unofficial)
[S2] gpdaniels/pnd3d, https://github.com/gpdaniels/pnd3d, fetched 2026-10-27 (unofficial)
