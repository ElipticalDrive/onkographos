# Centralized Conics Approach
Also called: Botsch et al. Splat Extent Approximation, Botsch et al.'s approximation, Zwicker et al. Centralized Conics Approach

## Overview
The Centralized Conics Approach is an efficient approximation method for computing the screen-space position and extents of a splat in point-based rendering [S1, unofficial]. It approximates the projected Gaussian kernel's footprint by utilizing the inverse of a 2D-to-2D projective mapping from the splat's parameter space to screen space [S1, unofficial]. Unlike methods that prioritize geometric exactness or robustness under extreme perspective, this approach trades precision for computational speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The splat's center position, the two vectors spanning its elliptical shape in local space, and the camera projection matrix [S1, unofficial].
*   **Output:** The screen-space center and the approximate extents of the splat [S1, unofficial].

**Data Structures:**
The method relies on a 2D-to-2D projective mapping matrix $M$ [S1, unofficial].

**Steps:**
1. Construct the 2D-to-2D projective mapping matrix $M$ that maps coordinates from the splat's local parameter space to screen space [S1, unofficial].
2. Compute the inverse of this mapping matrix, $M^{-1}$ [S1, unofficial].
3. Apply $M^{-1}$ to screen-space coordinates to approximate the splat's position and extents [S1, unofficial].

**Formulas:**
Let $M$ be the projective mapping matrix. The screen-space position $\mathbf{p}_{screen}$ relates to the local parameter space position $\mathbf{p}_{local}$ by $\mathbf{p}_{screen} \sim M \mathbf{p}_{local}$ [S1, unofficial]. The approach computes the extents by inverting this relationship: $\mathbf{p}_{local} \sim M^{-1} \mathbf{p}_{screen}$ [S1, unofficial].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
The method is numerically unstable when the projection of the splat approaches a line [S1, unofficial]. In such cases, the condition number of the mapping matrix increases rapidly, leading to large roundoff errors during inversion [S1, unofficial]. This instability can cause the screen-space extents to be underestimated, resulting in rendering artifacts [S1, unofficial].

## Subtopics

### [Projective Splat Rasterization with Depth Optimization](2088-projective-splat-rasterization-with-depth-optimization.md)
A rasterization technique that uses the inverse of a projective mapping matrix to transform screen coordinates back to object space for kernel evaluation, while simultaneously computing window-space depth values via linear interpolation of homogeneous coordinates. It sets itself apart from standard perspective splatting by avoiding explicit matrix inversion through the use of adjoint matrices and optimizing depth computation by embedding the window-space depth formula directly into the homogeneous interpolation weights.

### [Perspective Accurate Splatting via Projective Conic Mapping](3485-perspective-accurate-splatting-via-projective-conic-mapping.md)
This method computes perspective-correct splat shapes for point-based rendering by approximating the projective mapping of Gaussian reconstruction kernels from their local tangent planes to image space using affine mappings derived from conic isocontours. Unlike previous methods that used affine approximations exact only at the kernel center, this approach ensures accuracy at the kernel cutoff boundary, preventing artifacts such as holes and incorrect elongation under extreme perspective projections. It solves the problem where previous projections integrated kernels along the z-axis rather than the oblique ray from the camera, causing particles to be too faint by $\cos(\theta)$ and insufficiently elongated radially.

## Operating characteristics
The Centralized Conics Approach is computationally efficient, making it suitable for real-time rendering on hardware such as NVIDIA GPUs [S1, unofficial]. Its primary limitation is numerical instability under extreme perspective projections where the splat projects to a near-line [S1, unofficial]. This instability can lead to underestimation of extents and visual artifacts [S1, unofficial]. It is best used when performance is critical and extreme perspective distortions are rare or acceptable to approximate [S1, unofficial].

## Background
The method is attributed to Zwicker et al. for the centralized conics approach, while the efficient approximation formulas are credited to Botsch et al. [S1, unofficial]. It was noted in the context of GPU-based raycasting of quadric surfaces by Sigg et al. as an efficient but potentially unstable solution compared to methods that avoid matrix inversion [S1, unofficial].

## Sources
[S1] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
