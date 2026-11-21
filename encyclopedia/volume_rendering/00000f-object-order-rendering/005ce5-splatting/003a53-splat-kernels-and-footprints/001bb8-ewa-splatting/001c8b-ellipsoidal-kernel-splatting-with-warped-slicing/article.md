# Ellipsoidal Kernel Splatting with Warped Slicing

Also called: Efficient ellipsoid slicing on the GPU, Ellipsoid slicing, Warping slicing planes

## Overview
Ellipsoidal Kernel Splatting with Warped Slicing is a volume rendering technique that represents point samples as 3D ellipsoids and rasterizes them by slicing the ellipsoid into 2D polygons [S1]. It solves the problem of efficiently computing the intersection of general ellipsoids with screen-aligned slicing planes on GPU hardware, where direct analytical slicing requires expensive square roots and conditional statements [S1]. The method warps the slicing planes from screen space into the coordinate system of a unit sphere, allowing the use of hard-wired GPU texture mapping and blending on radially symmetric kernel slices [S1]. It differs from its parent [EWA Splatting](1069-ewa-splatting.md) by explicitly slicing the kernel into discrete polygons for incremental rasterization rather than rasterizing the entire projected footprint, and it avoids the pre-integrated slab approach of earlier image-aligned splatting by using point-sampled slices [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of point samples, each defined by position $(X, Y, Z)$, scaling factors $(S_x, S_y, S_z)$, and a rotation matrix $R$ [S1]. The viewing transformation $V = S_V R_V$ (zoom and rotation) is also required [S1].
*   **Output:** A set of 2D polygonal slices rasterized to screen space, which are then composited using front-to-back alpha blending [S1].

**Data Structures:**
Each ellipsoid is stored using 13 floating-point values in the GPU memory [S1]:
1.  $X, Y, Z$: Position of the ellipsoid center.
2.  $S_x, S_y, S_z$: Scaling factors along the local axes.
3.  $R_1, R_2, R_3, R_4, R_5, R_6$: Six elements representing the $3 \times 3$ rotation matrix $R$. The third row is derived from the cross-product of the first two rows to ensure orthogonality [S1].

**Method Steps:**

1.  **Construct the Ellipsoid Quadric:**
    Model the 3D elliptical Gaussian kernel as a general ellipsoid (quadric surface) [S1]. The implicit equation is:
    $$ (x, y, z, 1)^T \cdot Q \cdot (x, y, z, 1) = 0 $$
    The quadric matrix $Q$ is composed by scaling and rotating a unit sphere [S1]. Let $I'$ be the quadric matrix of a unit sphere:
    $$ I' = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & 0 \\ 0 & 0 & 0 & -1 \end{bmatrix} $$
    The transformation matrix $M$ maps the unit sphere to the ellipsoid in screen space [S1]. It is constructed as:
    $$ M = R_V \cdot T \cdot R \cdot S $$
    Where:
    *   $R_V$ is the view rotation.
    *   $T$ is the translation matrix:
        $$ T = \begin{bmatrix} 1 & 0 & 0 & X \\ 0 & 1 & 0 & Y \\ 0 & 0 & 1 & Z \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
    *   $R$ is the rotation matrix:
        $$ R = \begin{bmatrix} R_{11} & R_{12} & R_{13} & 0 \\ R_{21} & R_{22} & R_{23} & 0 \\ R_{31} & R_{32} & R_{33} & 0 \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
    *   $S$ is the scaling matrix:
        $$ S = \begin{bmatrix} S_x & 0 & 0 & 0 \\ 0 & S_y & 0 & 0 \\ 0 & 0 & S_z & 0 \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
    
    The quadric matrix $Q$ is calculated as:
    $$ Q = (M^{-1})^T \cdot I' \cdot M^{-1} $$
    Since $R$ is orthogonal ($R^{-1} = R^T$) and $S$ is diagonal ($S^{-1} = S^T$), this simplifies to:
    $$ Q = R \cdot S^{-1} \cdot I' \cdot S^{-1} \cdot R^T $$
    [S1]

2.  **Transform to Unit Sphere Space:**
    To exploit radial symmetry, transform the ellipsoid into a unit sphere [S1]. Multiply the view-transformed ellipsoid quadric $Q$ by the inverse transformation $M$:
    $$ Q' = M^T \cdot Q \cdot M = I' $$
    This confirms that in the transformed space, the kernel is a unit sphere [S1].

3.  **Compute Slicing Planes in Unit Sphere Space:**
    The slicing planes are defined in screen space as a set of parallel planes perpendicular to the viewing direction [S1]. To find their representation in unit sphere space, apply the inverse of the transformation matrix $M'$ to the coordinate system vectors of the screen-aligned slicing planes [S1].
    
    First, compute the effective transformation matrix $M'$ used for the warping:
    $$ M' = R_V \cdot T^{-1} \cdot R_V^{-1} \cdot M $$
    [S1].
    
    The vectors $u, v, n$ in unit sphere space are the first three columns of $M'^{-1}$:
    $$ [u, v, n] = M'^{-1} $$
    [S1]

4.  **Re-orthogonalize Vectors:**
    Because the scaling matrix $S$ is not orthonormal, the vectors $u, v, n$ derived from $M'^{-1}$ are generally not orthogonal [S1]. Re-orthogonalize them to obtain $u', v', n'$ [S1].
    *   $n'$ is the normalized direction of the axis passing through the centers of all slicing planes and the sphere center [S1].
    *   $u'$ and $v'$ define the orientation of the slicing planes [S1].

5.  **Define Incremental Vectors:**
    Compute four sets of vectors for the incremental rasterization procedure:
    *   $n_{us}$: Vector connecting centers of adjacent slices in unit sphere space.
    *   $n_{ss}$: Vector connecting centers of adjacent slices in screen space.
    *   $u_{ss}, v_{ss}$: Vectors defining the slice texture polygon orientation in screen space.
    *   $u_{us}, v_{us}$: Vectors defining the slice texture orientation in unit sphere space.
    
    Transform the unit sphere space vectors to screen space by multiplying by $M$:
    $$ [u_{ss}, v_{ss}, n_{ss}] = M \cdot [u_{us}, v_{us}, n_{us}] $$
    [S1]

6.  **Rasterize Slices:**
    Use the vectors $u', v', n'$ to extract consecutive kernel slices from a pre-computed 1D texture storing the Gaussian scaling function [S1]. The distance from the center of the kernel determines the scaling factor for the footprint texture [S1]. Rasterize these 2D polygons to the screen using the screen-space orientation vectors $u_{ss}, v_{ss}$ and position increments $n_{ss}$ [S1].

**Parameters:**
*   **Slice Spacing:** The distance between adjacent slicing planes. Not given in the sources.
*   **Texture Resolution:** Resolution of the 1D Gaussian scaling texture. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Pre-integrated Slabs:** Storing pre-integrated slab integrals leads to errors because the warping undoes the orthogonality of the rays [S1]. The integrals for warped rays would be looked up incorrectly [S1]. Therefore, only a cross-sectional texture of the radially symmetric RBF (Gaussian) is stored, which is then scaled by the orthogonal kernel value [S1].
*   **Staircase Artifacts:** Since the method extracts kernel slices (point-sampling of the slice function) rather than pre-filtering slabs, staircase artifacts may occur [S1]. Pre-integrated rendering can be added using interpolated densities of front and back slices as indices to overcome this [S1].
*   **GPU Hardware:** Direct analytical slicing of the general ellipsoid requires square roots and conditional statements, which are not well accelerated on GPU hardware [S1]. The warping method avoids these by using vector/matrix operations and texture mapping [S1].

## Subtopics

### [GPU-Accelerated Image-Aligned Splatting Framework](1293-gpu-accelerated-image-aligned-splatting-framework.md)
An object-order rendering approach for scattered data points using front-to-back compositing, spatial decomposition, and GPU-based early z-culling for occlusion culling and splat elimination. It differs from its parent [Ellipsoidal Kernel Splatting with Warped Slicing](1290-ellipsoidal-kernel-splatting-with-warped-slicing.md) by replacing the warped slicing planes with a direct screen-aligned slicing pass that outputs vertex attributes for GPU rasterization, enabling efficient occlusion culling via early Z-tests.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Each ellipsoid requires 13 floats for storage [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Avoids expensive square roots and conditional statements required by direct analytical ellipsoid slicing; exploits hard-wired GPU texture mapping and blending [S1].
*   **Limitations:** Point-sampling of slices may introduce staircase artifacts compared to pre-integrated slab methods; requires careful handling of pre-integration due to warping-induced non-orthogonality [S1].
*   **When to use:** When rendering ellipsoidal kernels on GPU hardware where performance is critical and direct analytical slicing is too slow [S1].

## Background
The method was introduced in the paper "GPU-Accelerated Volume Splatting With Elliptical RBFs" [S1]. It builds on techniques from [ZPvB*01] (EWA splatting), [JWH*04], [WBH*05], [GM04], and [MCQ04] [S1]. The authors previously described techniques in [NM05] that promoted the use of built-in hardware facilities such as early z-culling to eliminate kernels projecting onto opaque image areas [S1]. This work extends the standard algorithm by generalizing the basic primitive of the spherical kernel to a general ellipsoid [S1].

## Sources
[S1] GPU-Accelerated Volume Splatting With Elliptical RBFs, https://doi.org/10.2312/vissym/eurovis06/013-020, fetched 2026-10-26
