# Congradient Shading

Also called: Gradient-Based Shading Lookup, Gradient-based Table-driven Shading, Table-driven congradient shading, real-time congradient shading

## Overview
Congradient shading is a real-time volumetric shading acceleration technique that defines the surface orientation of a voxel as one of a finite set of local gradients, enabling a fast table-driven mechanism [S2, book]. It solves the performance bottleneck of per-voxel normal calculation and lighting integration by integrating shading into the projection pipeline, generating "congradient values" (concatenated color and gradient data) that serve as direct indices to a precomputed illumination look-up table [S2, book]. Unlike sibling methods that may perform shading in a separate post-processing stage or require complex normal quantization, congradient shading operates as a pipeline with the viewing process, accruing no additional time for shading because the congradient calculation time is much less than the projection time [S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset with precomputed normals, opacities, and an octree structure [S1]. The light source location can be arbitrarily changed [S2, book].
*   **Output:** A shaded image where each pixel's color is determined by the congradient value derived from the voxel's local gradient and color [S2, book].

**Data Structures:**
1.  **Triple Shift Register:** A hardware structure holding the depth and color values of the voxel being processed, the whole scan-line before it, and the whole scan-line after it. This makes the voxel's two horizontal and vertical neighbors readily available for gradient computation [S3].
2.  **Congradient Value:** A data unit holding concatenated values of color and local gradient [S2, book].
3.  **Look-Up Table (LUT):** A table containing color as a function of local voxel gradient [S1]. The indices are the congradient values, and the values are the shaded colors based on light source parameters [S2, book].

**Steps:**
1.  **Preprocessing:** Compute normals, opacities, and the octree as a preprocessing step [S1]. Define the surface orientation of a voxel as one of a finite set of local gradients [S2, book].
2.  **Projection and Congradient Generation:** The shading unit is an integral part of the projection process. Each projected pixel is pipelined through memory access, the projection unit, and the shading unit [S2, book].
3.  **Gradient Computation:** Using the triple shift register, compute the local gradient for the current voxel. Two approaches are implemented:
    *   **Unidirectional Shading:** The gradient is computed only from the depth of the horizontal neighbors [S2, book].
    *   **Bidirectional Shading:** Both horizontal and vertical neighbors are employed [S2, book].
4.  **Concatenation:** Generate a congradient value by concatenating the voxel's color and the computed local gradient [S2, book]. The production time of a congradient value is interleaved with the projection time of the next pixel [S2, book].
5.  **Lookup:** The display stage uses standard hardware where the congradient values are used as indices to the look-up table [S2, book].
6.  **Shading:** The look-up table translates the congradient value to its shaded color based upon the light source parameters [S2, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing phase
normals = compute_normals(volume)
opacities = compute_opacities(volume)
octree = build_octree(volume)

# Hardware/Architecture specific structures
# triple_shift_register holds: [scanline_before, current_scanline, scanline_after]
# Each scanline is an array of (depth, color) tuples

def process_pixel(x, y, z, light_params):
    # 1. Access neighbors via triple shift register
    # Assuming current voxel is at index i in current_scanline
    # left_neighbor = current_scanline[i-1]
    # right_neighbor = current_scanline[i+1]
    # top_neighbor = scanline_before[i]
    # bottom_neighbor = scanline_after[i]
    
    # 2. Compute local gradient based on approach
    # For unidirectional:
    gradient = compute_gradient_horizontal(left_neighbor, right_neighbor)
    
    # For bidirectional:
    # gradient = compute_gradient_bidirectional(left_neighbor, right_neighbor, top_neighbor, bottom_neighbor)
    
    # 3. Get current voxel color
    current_voxel = current_scanline[i]
    color = current_voxel.color
    
    # 4. Generate congradient value (concatenate color and gradient)
    # Implementation detail: exact bit-packing depends on hardware, 
    # but logically it combines color and gradient index
    congradient_value = concatenate(color, gradient)
    
    # 5. Lookup shaded color
    shaded_color = look_up_table[congradient_value]
    
    return shaded_color

# The projection loop interleaves this with memory access
# No additional time is accrued for shading [S2, book]
```

**Parameters:**
*   **Light Source Location:** Can be arbitrarily changed, providing motion parallax [S2, book].
*   **Gradient Set:** A predefined set of surface gradients stored in the look-up table [S3].

**Edge Cases and Pitfalls:**
*   **Hardware Realization:** The architecture is designed to be simple and suitable for hardware realization [S2, book].
*   **Precision:** The method relies on a finite set of local gradients, which may limit the accuracy of surface orientation representation compared to continuous normal fields [S2, book].

## Subtopics

### Precomputed Illumination Table
A technique to accelerate rendering by precomputing illumination for each possible encoded gradient direction index, storing results in a table independent of color [S4, book]. It computes illumination once for each possible encoded index and stores results in an array where each scalar value may have separate entries for ambient, diffuse, and specular illumination [S4, book]. Positional light sources cannot be supported accurately because they result in different light vectors for data samples with the same gradient due to different positions in the volume; thus, positional lights are approximated as infinite lights, and a single view direction is used for specular highlights to prioritize fast rendering over accuracy [S4, book].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** No additional time is accrued for shading because the congradient calculation time for each pixel is much less than its projection time [S2, book]. The total time for an arbitrary projection is $O(n^2 \log n)$, allowing real-time arbitrary projection [S2, book].
*   **Memory:** Requires a look-up table for congradient values and a triple shift register for neighbor access [S2, book][S3].

**Measured Performance:**
*   Implemented on the Cube architecture, a VLSI-based volume visualization system [S3].
*   Tested on workstations such as Sun-S, Sun-4, Silicon Graphics IRIS, and HP Turbo SRX with resolutions of $128^3$, $256^3$, and $512^3$ [S2, book].

**Scaling:**
*   The modular nature of the Cube architecture is well suited for VLSI implementation [S3].

**Strengths and Limitations:**
*   **Strengths:** Faster than alternative techniques, simpler for hardware realization, supports arbitrary light source movement for motion parallax [S2, book].
*   **Limitations:** Accuracy of specular highlights is limited to orthographic viewing directions where the view vector does not vary based on sample position [S4, book]. Positional lights are approximated as infinite lights [S4, book].

**When to Use:**
*   When real-time performance is critical and hardware implementation is feasible [S2, book][S3].
*   When motion parallax is desired with arbitrary light source movement [S2, book].

**When Not to Use:**
*   When high accuracy for positional lights or perspective specular highlights is required [S4, book].

## Background
Congradient shading was developed for the Cube architecture, emphasizing real-time performance [S2, book]. It capitalizes on the advantages of both gradient shading and contextual shading by defining the surface orientation of a voxel as one of a finite set of local gradients [S2, book]. The technique was developed as part of the Cube project, which aimed to build a full-scale VLSI-based volume visualization system [S3]. The first implementation, Cube-1, used a $16^3$ resolution and verified basic concepts of skewed memory, parallel access, and parallel processing of an orthographic beam [S3].

## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
[S2] Viewing and Rendering Processor for a Volume Visualization System, https://doi.org/10.2312/eggh/eggh89/171-178, fetched 2026-10-26 (book)
[S3] Building a Full Scale VLSI-Based Volume Visualization System, https://doi.org/10.2312/eggh/eggh90/109-115, fetched 2026-10-26
[S4] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
