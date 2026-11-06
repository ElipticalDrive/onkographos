# 3D Table Computation on GPU

Also called: 3D Texture Lookup Volume Rendering, GPU-based 3D Texture Volume Rendering

## Overview
This method computes a 3D pre-integration table for volume rendering by rasterizing 2D slices on the GPU, where each pixel represents a specific triplet of scalar values $(s_f, s_m, s_b)$ along a ray [S1]. It solves the problem of efficiently evaluating the volume rendering integral for transfer functions defined on 8-bit scalar domains by pre-computing the integration results into a texture [S1]. It is distinguished from its parent method, [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md), by utilizing GPU rasterization and pre-integration to accelerate the evaluation of the volume rendering equation, rather than performing per-ray sampling in software or on the CPU [S1].

## How it works
**Inputs and Outputs:**
- Input: A transfer function mapping scalar values to color and opacity, defined on an 8-bit domain [S1].
- Output: A 3D texture (pre-integration table) of size $256 \times 256 \times 256$, where each voxel $(i, j, k)$ stores the accumulated color and opacity for a ray segment traversing scalar values $i, j, k$ [S1].

**Data Structures:**
- **Pre-integration Table:** A 3D texture with dimensions $256 \times 256 \times 256$ [S1]. Each texel stores a 4-component floating-point vector $(R, G, B, \alpha)$ representing the accumulated color and opacity [S1].
- **Viewport:** $256 \times 256$ pixels [S1].
- **Projection:** Orthogonal projection matrix [S1].

**Algorithm Steps:**
1. **Setup:** Configure an orthogonal projection matrix and a viewport of size $256 \times 256$ [S1]. Initialize the 3D texture for the pre-integration table [S1].
2. **Slice Rendering Loop:** For each depth index $k$ from 0 to 255 (representing the back sample $s_b$):
   a. Set a uniform variable in the fragment shader to the current back sample value $s_b = k$ [S1].
   b. Render a quadrilateral parallel to the projection plane, filling the viewport [S1].
   c. **Fragment Shader Execution:** For each pixel $(x, y)$ in the viewport:
      i. Map screen coordinates to scalar values: $s_f = x$ (front sample), $s_m = y$ (middle sample) [S1].
      ii. Determine the coefficients $A, B, C$ for the second-order polynomial approximation according to equation (2) in the source [S1].
      iii. Initialize accumulated color $C_{acc} = (0, 0, 0)$ and accumulated opacity $\alpha_{acc} = 0$ [G].
      iv. Iterate a fixed number of steps $N$ over the interval $[0, 1]$ [S1]. For each step $t$:
          1. Compute the scalar value $s$ at the current step using Horner's rule for numerical accuracy and performance: $s = (A \cdot t + B) \cdot t + C$ [S1].
          2. Fetch the color $c(s)$ and opacity $\alpha(s)$ from the transfer function for scalar $s$ [S1].
          3. Composite the result using a back-to-front algorithm with the previous accumulated color and opacity [S1].
      v. Write the final $(C_{acc}, \alpha_{acc})$ to the frame buffer at pixel $(x, y)$ [S1].
   d. Perform a copy-to-texture operation to store the rendered 2D slice into the $k$-th slice of the 3D pre-integration table [S1].

**Parameters:**
- **Number of Iterations ($N$):** User-defined [S1]. Determines the quality of the pre-integration [S1].
- **Scalar Domain:** 8-bit (0-255) [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Accumulation is performed in floating-point registers to avoid precision loss during compositing [S1].
- **Polynomial Approximation:** The method uses a second-order polynomial approximation for the transfer function integration [S1].
- **Boundary Conditions:** The method assumes scalar values map directly to integer indices $0-255$ [S1].

## Subtopics

### [3D Table Animation](2531-3d-table-animation.md)
3D Table Animation is a method for controlling the movement and shape of volumetric gases by storing control values in a 3D table surrounding the gas volume in world space. It solves the problem of directing gas flow by interpolating values from lattice points to determine sample-specific control parameters. It is distinguished from its parent method, [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md), by its specific application to gas animation via functional flow fields and vector fields, rather than general volume shadowing or density computation.

## Operating characteristics
- **Time Complexity:** $O(256^3 \cdot N)$, where $N$ is the number of integration steps per ray segment [G]. The method requires rendering 256 slices, each with 256x256 pixels, and performing $N$ iterations per pixel [S1].
- **Memory Complexity:** $O(256^3)$ for the pre-integration table, storing 4 floats per voxel [G].
- **Performance:** The method leverages GPU parallelism for the rasterization and compositing steps [G]. The fixed number of iterations and polynomial evaluation allow for efficient shader execution [S1].
- **Strengths:** Pre-computation allows for fast rendering during the final volume rendering pass, as the integration results are looked up from the texture [G].
- **Limitations:** The method is specific to 8-bit scalar domains and second-order polynomial approximations [S1]. It requires significant GPU memory for the 3D texture [G]. The quality depends on the number of integration steps $N$ [S1].

## Background
This method is described in the paper "Second Order Pre-Integrated Volume Rendering" [S1]. It builds on the concept of pre-integrated volume rendering, adapting it for GPU implementation using rasterization techniques [S1]. The approach was presented at PacificVis 2008 [S1].

## Sources
[S1] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
