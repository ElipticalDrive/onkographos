# Centralized Conics Approach

Also called: Botsch et al. Splat Extent Approximation, Botsch et al.'s approximation, Zwicker et al. Centralized Conics Approach

## Overview
The Centralized Conics Approach is an efficient approximation method for computing the screen-space position and extents of a splat in point-based rendering [S1, unofficial]. It solves the problem of determining the bounding region of a projected Gaussian kernel by utilizing the inverse of a 2D-to-2D projective mapping from the splat's parameter space to screen space [S1, unofficial]. It sets itself apart from its siblings by prioritizing computational efficiency over geometric exactness, whereas methods like [Exact Half-Space Clipping](3395-exact-half-space-clipping.md) or [Mueller's Splatting Approach](5752-mueller-s-splatting-approach.md) aim for robustness or exactness at higher computational cost.

## How it works
The method approximates the screen-space footprint by inverting the projective transformation that maps the splat's local parameter space to the image plane [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** The splat's center position, the two vectors spanning its elliptical shape in local space, and the camera projection matrix.
*   **Output:** The screen-space center and the approximate extents (bounding box or ellipse) of the splat.

**Data Structures:**
The method relies on the 2D-to-2D projective mapping matrix $M$. This matrix is derived from the 3D-to-2D projection matrix by restricting it to the plane spanned by the splat's local basis vectors [S1, unofficial].

**Steps:**
1.  Construct the 2D-to-2D projective mapping matrix $M$ that maps coordinates from the splat's local parameter space (defined by its center and spanning vectors) to screen space [S1, unofficial].
2.  Compute the inverse of this mapping matrix, $M^{-1}$ [S1, unofficial].
3.  Apply $M^{-1}$ to the screen-space coordinates to determine the exact screen-space position and extents of the splat [S1, unofficial].

**Formulas:**
Let $M$ be the $3 \times 3$ projective mapping matrix. The screen-space position $\mathbf{p}_{screen}$ is related to the local parameter space position $\mathbf{p}_{local}$ by:
$$ \mathbf{p}_{screen} \sim M \mathbf{p}_{local} $$
The approach computes the extents by inverting this relationship:
$$ \mathbf{p}_{local} \sim M^{-1} \mathbf{p}_{screen} $$
Every symbol in $M$ and its inverse is defined by the camera projection and the splat's local geometry [S1, unofficial].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
The method is numerically unstable when the projection of the splat approaches a line [S1, unofficial]. In such cases, the condition number of the mapping matrix $M$ increases rapidly, leading to large roundoff errors in the inversion process [S1, unofficial]. This instability can cause the screen-space extents to be underestimated, resulting in rendering artifacts [S1, unofficial].

## Subtopics

### [Projective Splat Rasterization with Depth Optimization](2088-projective-splat-rasterization-with-depth-optimization.md)
A rasterization technique that uses the inverse of a projective mapping matrix to transform screen coordinates back to object space for kernel evaluation, while simultaneously computing window-space depth values via linear interpolation of homogeneous coordinates. It sets itself apart from standard perspective splatting by avoiding explicit matrix inversion through the use of adjoint matrices and optimizing depth computation by embedding the window-space depth formula directly into the homogeneous interpolation weights.

### [Perspective Accurate Splatting via Projective Conic Mapping](3485-perspective-accurate-splatting-via-projective-conic-mapping.md)
This method computes perspective-correct splat shapes for point-based rendering by approximating the projective mapping of Gaussian reconstruction kernels from their local tangent planes to image space using affine mappings derived from conic isocontours. Unlike previous methods that used affine approximations exact only at the kernel center, this approach ensures accuracy at the kernel cutoff boundary, preventing artifacts such as holes and incorrect elongation under extreme perspective projections. It solves the problem where previous projections integrated kernels along the z-axis rather than the oblique ray from the camera, causing particles to be too faint by $\cos(\theta)$ and insufficiently elongated radially.

## Operating characteristics
The Centralized Conics Approach is computationally efficient, making it suitable for real-time rendering on hardware such as NVIDIA GPUs [S1, unofficial]. However, its primary limitation is numerical instability under extreme perspective projections where the splat projects to a near-line [S1, unofficial]. This instability can lead to underestimation of extents and visual artifacts [S1, unofficial]. It is best used when performance is critical and extreme perspective distortions are rare or acceptable to approximate.

## Background
The method is attributed to Zwicker et al. and is also referred to as Botsch et al.'s approximation [S1, unofficial]. It was noted in the context of GPU-based raycasting of quadric surfaces by Sigg et al. as an efficient but potentially unstable solution compared to methods that avoid matrix inversion [S1, unofficial].

## Sources
[S1] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
