# Parallel Coordinate Plots Generation

Also called: PC Plot Generation

## Overview
Parallel Coordinate Plots (PCP) Generation is a visualization technique used to represent the joint distribution of scalar attribute pairs in volume rendering transfer functions. It solves the problem of over-plotting and high rendering costs associated with rendering individual data samples by mapping 2D histogram bins directly to geometric primitives (quads) [S1, unofficial]. Unlike point-based or line-based histogram visualizations, this method generates a quad for each nonzero bin in the joint histogram, effectively separating the number of rendered primitives from the number of raw data samples [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of conditional histogram textures (`texcond`), where each texture slice represents the joint histogram of a pair of scalar attributes (e.g., density and gradient magnitude). Each pixel $P(i, j)$ in the histogram texture contains a data value $v(i, j)$ representing the count of samples falling into bin $(i, j)$.
*   **Output:** A set of colored quads stored in a transform feedback buffer, ready for direct rendering.

**Data Structures:**
*   **Conditional Histogram Texture (`texcond`):** A 2D texture array or a series of 2D textures. Each slice corresponds to a specific pair of attributes. The resolution matches the binning of the histogram.
*   **Transform Feedback Buffer:** A GPU buffer object that captures the output vertices and attributes from the geometry shader stage without reading back to the CPU.

**Algorithm Steps:**
1.  **Setup:** Initialize a transform feedback buffer to record the resulting geometry.
2.  **Iteration:** Loop through all pairs of conditional histograms (or slices of the histogram texture array).
3.  **Rasterization:** For each histogram slice, draw a regular grid of vertices. The grid dimensions must match the resolution of the input conditional histogram texture slice.
4.  **Geometry Shader Execution:** For each vertex in the grid, the geometry shader samples the corresponding value $v(i, j)$ from the `texcond` texture at position $(i, j)$.
    *   If $v(i, j) > 0$, the shader generates a quad.
    *   The quad starts at position $i$ on the PC axis corresponding to the first attribute (x-axis) and ends at position $j$ on the PC axis corresponding to the second attribute (y-axis).
5.  **Color Computation:** Compute the final color $C(i, j)$ for the quad by modulating the input base color $C_0(i, j)$ with a ratio derived from the data value. The formula is:
    $$C(i, j) = C_0(i, j) \frac{\log(v(i, j))}{\log(\Sigma v)}$$
    Where:
    *   $v(i, j)$ is the data value (count) at histogram bin $(i, j)$.
    *   $\Sigma v$ is the total sum of all voxel counts (or total samples) in the histogram.
    *   $C_0(i, j)$ is the input color associated with the bin.
6.  **Storage:** Store all generated quads (vertices and colors) in the transform feedback buffer.
7.  **Rendering:** Render the geometry directly from the transform feedback buffer to the screen. No read-back to the CPU is required.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GPU-side Geometry Shader Logic
# Input: texcond (2D texture array), base_color_map (2D texture array)
# Output: Transform Feedback Buffer containing quads

def generate_pcp_quads(texcond, base_color_map):
    # 1. Setup transform feedback buffer
    tf_buffer = allocate_transform_feedback_buffer()
    
    # 2. Calculate total sum of all values for normalization
    # Note: In a real GPU implementation, this might be done via a reduction pass
    # or pre-computed. Here we assume a global variable or texture lookup.
    total_v = sum_all_values(texcond)
    
    # 3. Loop through all histogram slices (attribute pairs)
    for slice_index in range(num_slices):
        # 4. Draw a regular grid matching the histogram resolution
        # Let W, H be the width and height of the histogram slice
        draw_regular_grid(width=W, height=H)
        
        # 5. Geometry Shader processes each vertex
        # The vertex shader passes the grid coordinates (i, j) to the geometry shader
        # The geometry shader performs the following for each vertex:
        
        def geometry_shader_process(vertex_coords):
            i, j = vertex_coords
            
            # Sample the histogram value
            v_ij = texcond[slice_index].sample(i, j)
            
            # Check if the bin is nonzero
            if v_ij > 0:
                # Compute color modulation factor
                # Handle log(0) or log(1) edge cases if necessary, 
                # though v_ij > 0 ensures log(v_ij) is defined.
                # If total_v is 1, log(total_v) is 0, leading to division by zero.
                # [G] Guard against division by zero if total_v <= 1.
                if total_v <= 1:
                    ratio = 1.0
                else:
                    ratio = log(v_ij) / log(total_v)
                
                # Get base color
                c0 = base_color_map[slice_index].sample(i, j)
                
                # Compute final color
                c_final = c0 * ratio
                
                # Generate quad geometry
                # Start at position i on axis X, end at position j on axis Y
                # The exact vertex positions depend on the PCP coordinate system mapping
                # Typically, one axis is horizontal, the other vertical.
                # A quad is generated to represent the bin.
                quad_vertices = generate_quad_for_bin(i, j)
                
                # Attach color to vertices
                for vertex in quad_vertices:
                    vertex.color = c_final
                
                # Store in transform feedback buffer
                tf_buffer.store(quad_vertices)
```

**Parameters:**
*   **Histogram Resolution:** Determines the granularity of the PCP. Higher resolution yields more detailed distributions but increases the number of quads.
*   **Color Mapping ($C_0$):** The base color assigned to each bin, often derived from the transfer function or data attributes.

**Edge Cases and Pitfalls:**
*   **Zero Values:** Bins with $v(i, j) = 0$ are skipped, avoiding unnecessary geometry generation.
*   **Logarithm Domain:** The formula uses $\log(v(i, j))$. If $v(i, j) = 0$, the log is undefined, but these bins are filtered out. If $v(i, j) = 1$ and $\Sigma v = 1$, the ratio becomes $0/0$. [G] A guard clause is required to handle cases where $\Sigma v \le 1$ to prevent division by zero.
*   **Over-plotting:** While this method reduces over-plotting compared to rendering individual samples, dense histograms can still result in many overlapping quads.

## Subtopics

### PCP from 2D Histograms
This variant, proposed by Novotny and Hauser [70], explicitly separates the number of polylines to render from the number of data samples by generating the PCP from 2D histograms of adjacent variable pairs. It also utilizes histogram analysis to identify outliers. This approach is chosen when the dataset is large and direct sample rendering is prohibitively expensive, or when outlier detection is integrated into the visualization pipeline.

## Operating characteristics
*   **Time Complexity:** The generation cost is proportional to the number of nonzero bins in the joint histograms, not the number of data samples. This makes it efficient for large datasets. The GPU parallelism allows for rapid generation of quads.
*   **Memory Complexity:** Requires storage for the histogram textures and the transform feedback buffer. The buffer size depends on the number of nonzero bins.
*   **Performance:** Rendering cost is significantly lower than rendering individual polylines for each sample. The use of transform feedback avoids CPU-GPU synchronization overhead.
*   **Strengths:** Efficient handling of large datasets; separates rendering cost from data size; integrates outlier detection; leverages GPU parallelism.
*   **Limitations:** Resolution is limited by the histogram binning; may lose fine-grained detail if bins are too coarse; over-plotting can still occur in dense regions.
*   **When to use:** When visualizing high-dimensional transfer functions for large volume datasets where direct sample rendering is too slow.
*   **When not to use:** When exact sample-level detail is required, or when the dataset is small enough that direct rendering is feasible.

## Background
Parallel Coordinate Plots were originally introduced by Inselberg for multivariate data visualization. In the context of volume rendering transfer functions, the specific method of generating PCPs from 2D histograms was proposed by Novotny and Hauser [70] to address the over-plotting issue inherent in rendering individual data samples. This approach was further discussed and implemented in the context of multivariate transfer function design by Zhou et al. [S1, unofficial]. The technique leverages GPU geometry shaders and transform feedback to efficiently generate the geometric representation of the histogram bins.

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
