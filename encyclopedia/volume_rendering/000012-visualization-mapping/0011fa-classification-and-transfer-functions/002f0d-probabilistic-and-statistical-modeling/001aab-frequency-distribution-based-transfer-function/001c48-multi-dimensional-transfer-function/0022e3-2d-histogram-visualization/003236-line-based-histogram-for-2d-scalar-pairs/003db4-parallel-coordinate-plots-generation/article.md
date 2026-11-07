# Parallel Coordinate Plots Generation
Also called: PCP from Histograms, PC Plot Generation
## Overview
Parallel Coordinate Plots (PCP) Generation visualizes the joint distribution of two scalar attributes by mapping nonzero entries in a 2D joint histogram to geometric primitives (quads) on parallel axes. It differs from point-based scatter plots by using quads to represent histogram bin occupancy, providing a denser and more intuitive representation of material boundaries in volume rendering transfer function design. This method builds on [Line-based Histogram for 2D Scalar Pairs](6702-line-based-histogram-for-2d-scalar-pairs.md) by utilizing the underlying 2D histogram data structure.
## How it works
**Inputs and Outputs:**
*   **Input:** A 2D joint histogram texture `texcond` of size $W \times H$, where each pixel $P(i, j)$ stores the count $v(i, j)$ of samples falling into bin $(i, j)$ for attributes $x$ and $y$. An input color $C_0(i, j)$ associated with the bin.
*   **Output:** A set of colored quads stored in a transform feedback buffer, representing the PCP.

**Data Structures:**
*   `texcond`: A 2D texture array or single 2D texture containing the histogram counts.
*   Transform Feedback Buffer: GPU memory buffer storing the generated vertex/quad data.

**Steps:**
1.  Initialize the transform feedback buffer to record geometry.
2.  Loop through all pairs of conditional histograms (or the single joint histogram slice).
3.  For each histogram slice, draw a regular grid of vertices with dimensions matching the histogram texture size ($W \times H$).
4.  In the geometry shader, for each vertex corresponding to grid position $(i, j)$:
    *   Sample the histogram value $v(i, j)$ from `texcond`.
    *   If $v(i, j) > 0$, generate a quad. The quad starts at position $i$ on the PC axis for attribute $x$ and ends at position $j$ on the PC axis for attribute $y$.
    *   Compute the final color $C(i, j)$ using the formula below.
    *   Output the quad with the computed color.
5.  Store all generated quads in the transform feedback buffer.
6.  Render the quads directly from the transform feedback buffer to the screen without reading back to the CPU.

**Formula:**
The final color $C(i, j)$ is computed by modulating the input color $C_0(i, j)$ with the ratio of the natural logarithm of the data value to the natural logarithm of the total voxel number [S1, unofficial]:

$$C(i, j) = C_0(i, j) \frac{\log(v(i, j))}{\log(\Sigma v)}$$

Where:
*   $v(i, j)$ is the count in the histogram bin $(i, j)$.
*   $\Sigma v$ is the sum of all values in the histogram (total voxel count).
*   $\log$ denotes the natural logarithm.

**Parameters:**
*   Histogram resolution: Determined by the size of `texcond`.
*   Color modulation: Based on the log-ratio of bin count to total count.

**Edge Cases and Pitfalls:**
*   **Zero Values:** Pixels with $v(i, j) = 0$ are skipped and do not generate geometry.
*   **Dynamic Range:** The use of logarithms handles high dynamic range in data values [S1, unofficial].
*   **Precision:** Standard floating-point precision is assumed for color and histogram values.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GPU Geometry Shader logic
# Input: texcond (2D texture), C0 (input color texture or uniform)
# Output: Quads in transform feedback buffer

def generate_pcp(texcond, C0):
    # 1. Setup transform feedback buffer
    tf_buffer = allocate_transform_feedback_buffer()
    
    # 2. Compute total voxel count
    total_voxels = sum(texcond)
    
    # 3. Draw grid and generate quads
    # Draw a grid of vertices matching texcond dimensions
    draw_grid(width=texcond.width, height=texcond.height)
    
    # Geometry shader execution per vertex (i, j)
    for i in range(texcond.width):
        for j in range(texcond.height):
            v_ij = texcond[i, j]
            
            if v_ij > 0:
                # Compute color modulation
                # Note: log(0) is undefined, but v_ij > 0 check handles it
                # log(total_voxels) is assumed > 0
                log_v = log(v_ij)
                log_total = log(total_voxels)
                
                # Modulate input color
                # C0[i, j] is the base color for this bin
                C_final = C0[i, j] * (log_v / log_total)
                
                # Generate quad
                # Start at position i on axis x, end at position j on axis y
                quad = create_quad(start_x=i, end_y=j, color=C_final)
                
                # Store in transform feedback buffer
                tf_buffer.append(quad)
    
    # 4. Render from transform feedback buffer
    render_from_buffer(tf_buffer)
```
[G] The pseudocode assumes a simplified CPU-side loop for clarity, but the actual implementation uses a GPU geometry shader processing a full-screen or grid-draw call, as described in the source. The `draw_grid` call triggers the geometry shader for each vertex.

## Subtopics
## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of histogram pixels $W \times H$, as each pixel is processed once in the geometry shader.
*   **Memory Complexity:** Proportional to the number of nonzero histogram entries, as only quads for $v(i, j) > 0$ are generated and stored.
*   **Performance:** Highly parallel on the GPU due to the use of geometry shaders and transform feedback buffers. No CPU-GPU data transfer is required for the geometry generation step.
*   **Strengths:** Efficient generation of dense visualizations; handles high dynamic range via logarithmic color modulation; direct rendering from GPU memory.
*   **Limitations:** Requires GPU support for geometry shaders and transform feedback. The visual density may become overwhelming for very large histograms.
*   **When to use:** When designing transfer functions for volume rendering and needing to visualize the joint distribution of two scalar attributes efficiently.
*   **When not to use:** When interactive exploration requires rapid updates and the histogram size is extremely large, potentially causing GPU bottlenecks.

## Background
This method was proposed in [70] (cited in [S1, unofficial]) and is described in detail in Zhou's dissertation [S1, unofficial]. It is part of a broader framework for multivariate transfer function design in volume rendering, specifically within the context of probabilistic and statistical modeling of scalar data. The approach leverages GPU hardware acceleration to generate complex visualizations from statistical data (histograms) without CPU intervention.

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
