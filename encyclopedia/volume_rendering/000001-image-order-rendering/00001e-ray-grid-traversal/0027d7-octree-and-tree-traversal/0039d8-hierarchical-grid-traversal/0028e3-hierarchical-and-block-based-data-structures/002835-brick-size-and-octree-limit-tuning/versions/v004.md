# Brick Size and Octree Limit Tuning

Also called: Hardware-specific Bricking, Performance Optimization via Brick Size and Octree Limit Tuning, Sampling Direction Optimization

## Overview
Brick Size and Octree Limit Tuning is a method for optimizing GPU-based volume rendering by empirically selecting optimal parameters for spatial subdivision and hardware texture storage [S1, unofficial]. It addresses the trade-off between rasterization overhead, triangle throughput, and cache efficiency inherent in hierarchical grid traversal [S1, unofficial]. It differs from sibling methods by focusing specifically on the hardware-aware selection of brick dimensions and octree depth limits to mitigate architecture-specific bottlenecks, such as 3D texture sampling penalties [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume dataset dimensions and density, graphics hardware specifications (texture memory size, 3D texture sampling behavior), and viewport resolution with viewing direction [S1, unofficial].
*   **Outputs:** Optimal brick side length in voxels and optimal octree limit in voxels [S1, unofficial].

**Data Structures:**
*   **Bricks:** 3D texture objects containing voxel data [S1, unofficial]. Layout depends on hardware; typically stored as stacks of 2D slices on certain architectures (e.g., ATi FireGL X1) [S1, unofficial].
*   **Octree:** Internal nodes store aggregate information (e.g., occupancy flags) derived from children [S1, unofficial]. Leaf nodes correspond to bricks or smaller cubes down to the octree limit [G].

**Numbered Steps:**
1.  **Analyze Hardware Constraints:** Determine if the GPU treats 3D textures as linear 3D arrays or stacks of 2D slices [S1, unofficial]. Identify the optimal texture size (power of 2) and available texture memory [S1, unofficial].
2.  **Determine Brick Size ($B$):** Select $B$ such that $B^3$ fits within available texture memory [S1, unofficial]. Balance cache efficiency (larger $B$) against interpolation overlap overhead (smaller $B$) [S1, unofficial]. If the hardware suffers from z-direction sampling penalties, alternate texture orientations for consecutive bricks to mitigate cache trashing when the viewing direction is not aligned with the x-y plane [S1, unofficial].
3.  **Determine Octree Limit ($L$):** Select $L$ to balance the number of triangles (rasterization load) against the overhead of traversing smaller nodes (CPU/triangle throughput) [S1, unofficial]. Optimize $B$ and $L$ orthogonally by varying one variable while holding the other constant [S1, unofficial].
4.  **Evaluate Performance:** Render the volume with the selected $B$ and $L$ [S1, unofficial]. Measure frame rate [S1, unofficial].
5.  **Iterate:** Adjust $B$ and $L$ based on performance metrics [S1, unofficial]. For hardware with sampling direction sensitivity, test performance for both xy-aligned and non-xy-aligned viewing directions [S1, unofficial].

**Formulas:**
*   **Early Ray Termination (ERT) Compositing:**
    $$C_{i+1} = (1 - A_i) \cdot \alpha_i \cdot c_i + C_i$$
    $$A_{i+1} = (1 - A_i) \cdot \alpha_i + A_i$$
    Where $C$ is accumulated color, $A$ is accumulated opacity, $\alpha_i$ is sample opacity, and $c_i$ is sample color [S1, unofficial].
*   **Acceleration Factor:**
    $$\text{Acceleration} = \frac{\text{Frame Rate}_{\text{optimized}}}{\text{Frame Rate}_{\text{non-optimized}}}$$
    [S1, unofficial]

**Parameters:**
*   **Brick Size ($B$):** Side length of the cubic bricks [S1, unofficial]. Optimal size depends on available texture memory, optimal texture size (power of 2), nature of the data set, overlap overhead, and hardware constraints [S1, unofficial]. It is orthogonal to the octree limit [S1, unofficial].
*   **Octree Limit ($L$):** The smallest octree cube allowed [S1, unofficial]. Balances rasterization load and triangle throughput; smaller limits cause CPU overhead and high triangle counts, while larger limits cause rasterization overhead [S1, unofficial].
*   **Texture Orientation:** For hardware treating 3D textures as 2D slice stacks, the orientation of the brick's texture coordinates relative to the viewing direction [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sampling Direction Penalty:** On ATi FireGL X1, traversing 3D textures in the z-direction is significantly slower than x or y [S1, unofficial]. If the viewing direction differs from the texture z-axis by more than 45°, z-traversal is inevitable, causing performance drops [S1, unofficial].
*   **Overlap Overhead:** Smaller bricks increase the overhead due to interpolation overlap [S1, unofficial].
*   **CPU Overhead:** Very small octree limits lead to high triangle counts and CPU overhead [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Octree build time is bounded by CPU performance [S1, unofficial]. For a $512^3$ volume (256MB), build time was 6.5ms on a Xeon 1.7GHz and 3.5ms on a Xeon 3.0GHz [S1, unofficial].
*   **Memory Complexity:** Depends on brick size and available texture memory [S1, unofficial]. Bricks must fit in texture memory [S1, unofficial].
*   **Performance:** 
    *   nVidia QuadroFX 3400: 73.5 fps (optimized) vs 9.6 fps (non-optimized), 7.66x acceleration [S1, unofficial].
    *   ATi FireGL X1 (xy aligned): 83.3 fps (optimized) vs 0.23 fps (non-optimized), 362x acceleration [S1, unofficial].
    *   ATi FireGL X1 (non xy aligned): 27.4 fps (optimized) vs 0.23 fps (non-optimized), 119x acceleration [S1, unofficial].
    *   3DLabs Wildcat 7110: 21.3 fps (optimized) vs 0.38 fps (non-optimized), 56.1x acceleration [S1, unofficial].
    *   Early Ray Termination Impact: On QuadroFX 3400 with a $512^3$ volume, frame rate increased from 2.2 fps (no opt) to 5.2 fps (bricking+octree) to 16.1 fps (with ERT) [S1, unofficial].
*   **Strengths:** Significant acceleration on hardware-specific bottlenecks; orthogonal optimization of brick size and octree limit allows fine-tuning for specific datasets and hardware [S1, unofficial].
*   **Limitations:** Performance is highly dependent on hardware architecture (e.g., 3D texture handling); requires empirical tuning for each GPU [S1, unofficial].

## Background
The method was presented by Danny Ruijters in "Optimizing GPU Volume Rendering" [S1, unofficial]. It addresses bottlenecks in the graphics pipeline for GPU-based volume rendering, specifically focusing on space-skipping and early ray termination tailored to hardware constraints [S1, unofficial].

## Subtopics

### Hardware-specific Bricking
Adapts brick storage and orientation to specific GPU architectures, particularly those with 3D texture sampling penalties. Chosen when rendering on hardware that treats 3D textures as 2D slice stacks. [Hardware-specific Bricking](2577-hardware-specific-bricking.md)

### Performance Optimization via Brick Size and Octree Limit Tuning
Empirical tuning of brick and octree parameters for maximum frame rate. Chosen when balancing rasterization load, triangle throughput, and cache efficiency is required for a specific GPU and dataset. [Performance Optimization via Brick Size and Octree Limit Tuning](4133-performance-optimization-via-brick-size-and-octree-limit-tuning.md)

### Sampling Direction Optimization
Techniques to mitigate performance drops due to non-aligned sampling directions in 3D textures. Chosen when viewing angles cause inevitable z-direction traversal on hardware with slice-stack texture memory. [Sampling Direction Optimization](1554-sampling-direction-optimization.md)

## Sources
[S1] Optimizing GPU Volume Rendering, Danny Ruijters, https://dannyruijters.nl/docs/OptimizingGPUVolumeRendering_final.pdf, fetched 2026-10-26 (unofficial)
