# Spatially Varying Gaussian Splatting (SVGS)

## Overview
SVGS is a method that utilizes spatially varying colors and opacity within a single Gaussian primitive to improve representation ability, addressing the non-compactness of vanilla Gaussian Splatting where primitives have only a single view-dependent color and opacity [S1]. It sets itself apart from baseline Gaussian Splatting methods by defining color and opacity as spatially varying functions of the intersection point $p$ on the Gaussian primitive, rather than constant values per primitive [S1]. This allows different rays intersecting the same Gaussian primitive to have different colors if they intersect at different locations, enabling a single primitive to fit complex textures that would otherwise require many primitives in baseline methods [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin and direction; a set of Gaussian primitives (specifically 2D Gaussian surfels by default) [S1].
*   **Output:** The final color $c$ and opacity $\alpha$ for the ray intersection [S1].

**Data Structures:**
The method operates on 2D Gaussian surfels [S1]. Each surfel maintains parameters for its geometry (position, orientation, scale) and the parameters for the chosen spatially varying function (e.g., bilinear weights, kernel centers, or MLP weights) [S1]. The intersection point $p$ is defined in the local 2D coordinate system of the Gaussian surfel, with the Gaussian origin as $(0,0)$ and the ellipse axes as coordinate axes [S1].

**Formulas:**
The final color $c$ and opacity $\alpha$ are computed as [S1]:
$$c(p, d) = SH(d) + F_c(p)$$
$$\alpha(p) = F_\alpha(p)$$
where:
*   $p$ is the intersection point between the ray and the Gaussian primitive [S1].
*   $d$ is the viewing direction [S1].
*   $SH(d)$ is the view-dependent spherical harmonic color based on $d$ [S1].
*   $F_c(p)$ is the spatially varying color function based on $p$ [S1].
*   $F_\alpha(p)$ is the spatially varying opacity function based on $p$ [S1].

**Parameters:**
*   **Primitive Type:** 2D Gaussian surfels are used as the base primitive representation [S1].
*   **Activation:** The final color and opacity are normalized to a valid range via a sigmoid activation during training and rendering, allowing intermediate $F_c$ and $F_\alpha$ values to be negative [S1].

**Steps:**
1.  **Compute Intersection Point:** Determine the intersection point $p$ between the ray and the Gaussian primitive. For 2DGS-based SVGS, $p$ is defined in the local 2D coordinate system of the Gaussian surfel, with the Gaussian origin as $(0,0)$ and ellipse axes as coordinate axes [S1].
2.  **Evaluate Spatially Varying Functions:** Compute $F_c(p)$ and $F_\alpha(p)$ using one of the implemented spatially varying functions (Bilinear Interpolation, Movable Kernels, or Tiny MLPs) [S1].
3.  **Compute View-Dependent Color:** Evaluate the spherical harmonic color $SH(d)$ based on the viewing direction $d$ [S1].
4.  **Combine Color:** Calculate the final color $c(p, d)$ by adding the view-dependent color and the spatially varying color component: $c(p, d) = SH(d) + F_c(p)$ [S1].
5.  **Compute Opacity:** Set the opacity $\alpha(p)$ to the result of the spatially varying opacity function: $\alpha(p) = F_\alpha(p)$ [S1].
6.  **Normalize:** Apply sigmoid activation to $c(p, d)$ and $\alpha(p)$ to ensure they are within valid ranges [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def svgs_render(ray_origin, ray_direction, gaussian_primitive):
    # 1. Compute intersection point p in local 2D coordinates
    # Intersection calculation is inherited from 2D Gaussian Splatting [G]
    p = gaussian_primitive.compute_local_intersection(ray_origin, ray_direction)
    
    # 2. Evaluate spatially varying functions Fc(p) and Falpha(p)
    # Implementation depends on the chosen variant (Bilinear, Movable Kernels, or MLP) [G]
    Fc = gaussian_primitive.spatially_varying_color(p)
    Falpha = gaussian_primitive.spatially_varying_opacity(p)
    
    # 3. Compute view-dependent spherical harmonic color SH(d)
    # Standard spherical harmonic evaluation inherited from parent method [G]
    SH_color = gaussian_primitive.spherical_harmonics(ray_direction)
    
    # 4. Combine color
    raw_color = SH_color + Fc
    
    # 5. Set opacity
    raw_opacity = Falpha
    
    # 6. Normalize via sigmoid activation
    final_color = 1.0 / (1.0 + math.exp(-raw_color))
    final_opacity = 1.0 / (1.0 + math.exp(-raw_opacity))
    
    return final_color, final_opacity
```

**Edge Cases and Pitfalls:**
*   **Negative Values:** $F_c$ and $F_\alpha$ may take negative values. This is permissible because they contribute additively to the original spherical harmonic color representation, and the final values are normalized via sigmoid activation [S1].
*   **Primitive Removal:** The optimization framework automatically removes Gaussians whose opacity falls below a critical threshold, which helps manage the parameter count [S1].
*   **Intersection Calculation:** The calculation of the intersection point $p$ assumes the use of 2D Gaussian Splatting surfels. Extending to 3D Gaussians requires regarding them as ellipsoids and calculating intersections with the ellipsoids, which may have different computational costs [S1].

## Subtopics

### Bilinear Interpolation
A spatially varying function implementation that uses bilinear interpolation to map local coordinates to color and opacity [S1]. It is the most compact variant in terms of parameters [S1]. It is chosen when computational efficiency and low parameter count are prioritized, though it may have lower representational power compared to other variants [S1].

### Movable Kernels
SVGS with Movable Kernels is a spatially varying function implementation within the SVGS framework that defines $k$ movable exponential kernels located on a Gaussian surfel, weighted by their distance to the intersection point [S1]. It sets itself apart from its sibling [Bilinear Interpolation](8729-spatially-varying-gaussian-splatting-svgs.md#bilinear-interpolation) by replacing fixed quadrant weights with learnable, gradient-driven kernel centers, and from [Tiny MLPs](8729-spatially-varying-gaussian-splatting-svgs.md#tiny-mlps) by using a compact parametric form rather than a neural network [S1]. It achieves the best novel view synthesis performance among the three variants [S1].

### Tiny MLPs (SVGS Variant)
A spatially varying function implementation using a small multilayer perceptron (MLP) on each Gaussian surfel to map local coordinates to color and opacity [S1]. The MLP accepts a two-dimensional input $p = (u, v)$ and outputs the color and opacity at that location, using a three-layer network with sigmoid activation [S1]. The Tiny MLP approach has significantly more parameters than Bilinear Interpolation or Movable Kernels, as shown in Fig. 5, but performs well when Gaussian primitive numbers are limited [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Depends on the chosen spatially varying function. Bilinear Interpolation and Movable Kernels are more compact than Tiny MLPs, which have significantly more parameters [S1].
*   **Measured Performance:** SVGS outperforms the baseline 2DGS in novel view synthesis across all three spatially varying function designs [S1]. SVGS with movable kernels surpasses all other Gaussian Splatting-based methods on the Blender dataset [S1].
*   **Scaling:** SVGS demonstrates compactness by using a limited number of Gaussian primitives and limited training times to achieve superior rendering quality [S1].
*   **Strengths:** Enhances representational power of Gaussian primitives, allowing them to better capture complex textures and geometric variations. Particularly effective in scenes combining complex textures with relatively simple geometry [S1].
*   **Limitations:** The Tiny MLP variant has a high parameter count. The method relies on the optimization framework to remove Gaussians with low opacity, which may require careful tuning [S1].
*   **When to Use:** When a single Gaussian primitive needs to represent complex textures that would otherwise require many primitives in vanilla Gaussian Splatting [S1].
*   **When Not to Use:** When computational resources are extremely limited and the scene geometry is simple enough that vanilla Gaussian Splatting suffices [S1].

## Background
SVGS was introduced in the paper "SVGS: Enhancing Gaussian Splatting Using Primitives With Spatially Varying Colors" [S1]. It addresses the limitation of current Gaussian primitives having only a single view-dependent color and opacity, which results in a non-compact representation [S1]. The method was developed to improve novel-view synthesis and geometric reconstruction, particularly for scenes with complex textures on relatively flat geometries [S1]. It builds on the foundation of 2D Gaussian Splatting [3] and 3D Gaussian Splatting [2] [S1].

## Sources
[S1] SVGS: Enhancing Gaussian Splatting Using Primitives With Spatially Varying Colors., https://doi.org/10.1109/tvcg.2026.3690745, fetched 2026-10-26
