# Performance Optimization Techniques

Also called: Computation Reordering, Mode Sorting, Adaptive Stepping, Temporal Reprojection, Bilateral Upsampling, Quarter-Resolution Raymarching

## Overview
Performance Optimization Techniques reduce the computational cost of volume rendering and large-scale visualization by minimizing the number of samples evaluated, reordering operations to improve memory coherence, and leveraging temporal or spatial redundancy. Unlike [Acceleration](18-acceleration.md) methods which primarily cull empty space or terminate rays early based on opacity, these techniques focus on hardware utilization, memory bandwidth efficiency, and frame reconstruction. They address the high cost of raymarching volumetric textures and out-of-core dataset visualization by trading off some visual fidelity (e.g., resolution, strict ordering) for significant performance gains.

## How it works
The methods are categorized into spatial/temporal reconstruction, adaptive sampling, and data access optimization.

### Spatial and Temporal Reconstruction
These techniques reduce the number of rays marched per frame by rendering at a lower resolution or reusing previous frames.

**Inputs:** Current camera state, previous frame's color/depth buffers, current frame's low-resolution render.
**Outputs:** Full-resolution image.

1.  **Quarter-Resolution Raymarching [S1, unofficial]:**
    *   Execute the raymarching compute shader at half ($1/2$) or quarter ($1/4$) of the display resolution.
    *   Store the resulting cloud color and depth buffers.
2.  **Bilateral Upsampling [S1, unofficial]:**
    *   Upscale the low-resolution color and depth to full resolution.
    *   Use a depth-aware bilateral filter. The filter weights pixels based on color similarity and depth difference to prevent edge bleeding against terrain or other geometry.
3.  **Temporal Reprojection [S1, unofficial]:**
    *   **Jittering:** Apply a 2D blue noise pattern to jitter ray step offsets or pixel sampling positions per frame.
    *   **Reprojection:** Use camera motion vectors to map pixels from the previous frame's history to the current frame's position.
    *   **Blending:** Blend the reprojected history with the current low-resolution render to reconstruct a high-quality full-resolution image.

### Adaptive Stepping
This technique varies the ray step size based on local scene content to skip empty regions.

**Inputs:** Ray origin, ray direction, volume density field.
**Outputs:** Integrated color and opacity.

1.  **Initialize:** Set step size $s_{base}$ (fine step).
2.  **Loop:** For each sample along the ray:
    *   Query density $\rho$ at current position.
    *   **If $\rho > 0$:**
        *   Use fine step size $s_{base}$.
        *   Perform standard front-to-back compositing.
    *   **If $\rho = 0$ (Empty Air):**
        *   Double the step size: $s_{new} = 2 \cdot s_{current}$.
        *   Skip the density evaluation for intermediate points (or use a coarse check if available).
    *   **Check Termination:** If accumulated opacity $\alpha_{acc} \geq 1$ or ray exits volume, break.
    *   Update position: $pos = pos + s_{current}$.

### Data Access Optimization (Out-of-Core)
These techniques optimize memory hierarchy for datasets larger than RAM.

**Inputs:** Octree structure, disk storage, memory buffer.
**Outputs:** Rendered image.

1.  **Computation Reordering [S2, unofficial]:**
    *   Identify visible octree nodes.
    *   Separate nodes into "In-Memory" and "On-Disk" sets.
    *   **Rasterize In-Memory Nodes First:** Process all nodes currently resident in RAM.
    *   **Fetch and Rasterize On-Disk Nodes:** While in-memory nodes are being processed, fetch on-disk nodes. Rasterize them as they arrive.
    *   **Z-Buffer Sorting:** The final image is correct because the Z-buffer algorithm sorts primitives at the pixel level, making the execution order of independent nodes irrelevant.
2.  **Mode Sorting [S2, unofficial]:**
    *   Group primitives by rendering mode (e.g., polygon vs. line).
    *   Reorder traversal to render all primitives of one mode before switching to another.
    *   This minimizes context switches and state changes in the graphics pipeline.

**Pitfalls:**
*   **Computation Reordering:** Requires that operations are independent. If the final result depends on execution order (e.g., certain transparency blending orders without proper sorting), rescheduling produces incorrect results [S2, unofficial].
*   **Bilateral Upsampling:** Requires accurate depth information; incorrect depth can cause artifacts at object boundaries [S1, unofficial].
*   **Adaptive Stepping:** Doubling step size in empty space must be bounded to prevent missing thin features if the "empty" check is coarse [S1, unofficial].

## Subtopics

### Computation Reordering
A technique that reschedules independent operations to exploit memory coherence. It prioritizes rasterizing nodes currently in memory over those on disk. It is used in out-of-core visualization with octrees to improve interactivity, unlike photorealistic ray tracing systems that use regular grids [S2, unofficial].

### Mode Sorting
A technique that reorders primitive traversal to group identical rendering modes (e.g., all polygons, then all lines). It saves time by avoiding frequent graphics state switches (context switches) [S2, unofficial].

### Adaptive Stepping
A ray marching strategy that doubles the step size when traversing empty air and reverts to fine steps upon encountering non-zero density. It reduces sample count in transparent regions [S1, unofficial].

### Temporal Reprojection
A frame reconstruction technique that uses motion vectors and blue noise jittering to blend previous frames with current low-resolution renders. It maintains high frame rates by reducing the number of rays marched per frame [S1, unofficial].

### Bilateral Upsampling
A depth-aware upscaling filter used to upscale low-resolution raymarched results. It prevents edge bleeding by weighting pixels based on depth similarity, crucial for rendering clouds against terrain [S1, unofficial].

### Quarter-Resolution Raymarching
A performance strategy where the compute shader executes at half or quarter display resolution. It stores color and depth at lower resolution before upscaling, significantly reducing GPU load [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Quarter-Resolution reduces raymarching work by 4x to 16x.
    *   Adaptive Stepping reduces sample count proportionally to the volume of empty space traversed.
    *   Computation Reordering reduces I/O wait time by overlapping disk fetches with computation.
*   **Memory Complexity:**
    *   Temporal Reprojection requires storing history buffers (color, depth, motion vectors).
    *   Computation Reordering requires an octree structure and a buffer to track node residency status.
*   **Strengths:** High frame rates on consumer hardware; effective for large datasets exceeding RAM; reduces aliasing via temporal accumulation.
*   **Limitations:** Introduces temporal flicker if reprojection fails (e.g., fast camera motion); depth-aware upscaling can blur thin details if depth precision is low; out-of-core reordering adds complexity to the renderer architecture.
*   **When to use:** Real-time volumetric rendering (clouds, smoke); visualization of terabyte-scale datasets on workstations with limited RAM.
*   **When not to use:** Offline photorealistic rendering where strict sample ordering is required; scenes with rapid, unpredictable camera motion where reprojection artifacts are unacceptable.

## Background
Performance optimization in volume rendering has evolved from simple culling to complex memory-aware and temporal techniques. Early work focused on empty-space skipping [S2, unofficial]. The concept of computation reordering was notably applied by Pharr et al. in ray tracing systems for datasets larger than memory, focusing on photorealism with regular grids [S2, unofficial]. In contrast, interactive visualization systems adopted octrees and reordering for interactivity [S2, unofficial]. Modern real-time techniques, such as those for volumetric clouds, combine low-resolution raymarching with bilateral filtering and temporal reprojection to achieve high frame rates on OpenGL-compatible hardware [S1, unofficial].

## Sources
[S1] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
