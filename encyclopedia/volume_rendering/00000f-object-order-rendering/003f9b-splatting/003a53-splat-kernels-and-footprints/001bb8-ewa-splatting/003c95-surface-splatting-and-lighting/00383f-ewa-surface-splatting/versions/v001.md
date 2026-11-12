# EWA Surface Splatting

Also called: EWA Volume Splatting, Elliptical Weighted Average splatting, Zwicker EWA Volume Splatting

## Overview
EWA Surface Splatting is a high-quality point rendering algorithm that renders point samples by superimposing band-limited reconstruction kernels, called surface splats, in the image domain [S1]. It efficiently suppresses aliasing artifacts and guarantees hole-free reconstructions even for moderate sampling densities and under arbitrary transformations [S1]. Unlike simple forward-projection of pure point-samples which causes holes if sampling density is insufficient, EWA splatting uses splats with spatial extent to ensure hole-free results [S1]. It builds on [Surface Splatting and Lighting](9275-surface-splatting-and-lighting.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A set of point samples. Each sample consists of a position $\mathbf{c}$, a normal vector $\mathbf{n}$, and two tangent axes $\mathbf{u}, \mathbf{v}$ with corresponding radii $r_u, r_v$. These define an elliptical disk in object space [S1].
*   **Output:** A rasterized image with accumulated color, depth, and normal attributes [S1].

**Data Structures:**
Each splat is defined by:
*   Center position $\mathbf{c} \in \mathbb{R}^3$.
*   Normal $\mathbf{n} \in \mathbb{R}^3$.
*   Tangent basis vectors $\mathbf{u}, \mathbf{v} \in \mathbb{R}^3$, orthogonal to $\mathbf{n}$ and each other.
*   Radii $r_u, r_v \in \mathbb{R}$ defining the extent along $\mathbf{u}$ and $\mathbf{v}$.

**Steps:**
1.  **Object-Space Splat Definition:** For each point sample, construct a quadrilateral in object space. This quadrilateral is texture-mapped with a 2D Gaussian function. The dimensions of the quadrilateral correspond to the tangent axes $\mathbf{u}$ and $\mathbf{v}$ and their radii [S2, unofficial].
2.  **Deformation and Projection:** Deform the object-space quadrilateral to account for the viewing transformation and perspective projection. The goal is to result in the correct screen-space EWA splat footprint after projection [S2, unofficial].
3.  **Screen-Space Rasterization:** Rasterize the deformed quadrilateral. The texture coordinates map to the Gaussian kernel values, which are used to weight the contribution of the splat's attributes (color, normal, depth) to the framebuffer [S2, unofficial].
4.  **Accumulation:** Accumulate the weighted contributions of all overlapping splats into the frame buffer. This process reconstructs the continuous surface signal [G].

**Formulas:**
The screen-space footprint is analytically defined as a single 2D Gaussian footprint that integrates an elliptical 3D Gaussian reconstruction kernel and a 2D Gaussian low-pass filter [S2, unofficial]. The specific analytical formula for the screen-space covariance matrix is not provided in the sources [S2, unofficial].

**Parameters:**
*   Gaussian kernel width: Not given in the sources.
*   Tangent radii $r_u, r_v$: Determined by the sampling density and desired reconstruction quality [S1].

**Edge Cases and Pitfalls:**
*   **Insufficient Sampling Density:** Simple forward-projection of points causes holes. EWA splatting mitigates this by ensuring mutual overlap of surface splats in object space [S1].
*   **Arbitrary Transformations:** The method guarantees hole-free results even under arbitrary transformations, provided the splat radii are appropriately scaled [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method can be implemented efficiently on graphics hardware [S1]. Perspective accurate variants allow efficient projection and deformation on modern GPUs [S2, unofficial].
*   **Strengths:** High-quality rendering, hole-free reconstruction, suppression of aliasing artifacts [S1].
*   **Limitations:** Requires pre-computed tangent axes and radii for each point sample [S1].
*   **When to use:** High-quality direct visualization of point-sampled surfaces [S1].
*   **When not to use:** When sampling density is extremely low and cannot be compensated by splat radii, or when computational resources are severely limited and hardware acceleration is unavailable.

## Background
EWA splatting was introduced by Zwicker et al. in 2001 [S1]. Perspective accurate variants were introduced starting with Zwicker et al. 2004 and Botsch et al. 2004 [S1]. The method was designed for high-quality direct visualization of point-sampled surfaces [S1]. Ren et al. [Ren et al. 2002] derived an object space formulation of the EWA surface splats and described its efficient implementation on graphics hardware [S2, unofficial].

## Sources
[S1] A hardware architecture for surface splatting, https://doi.org/10.1145/1275808.1276490, fetched 2026-10-28
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
