# Spatial Subdivision and Hierarchical Sampling

Also called: Adaptive Ray Shooting & Quadtree Subdivision, Adaptive Refinement for Volume Rendering, Adaptive Screen Sampling

## Overview
Spatial Subdivision and Hierarchical Sampling accelerates volume rendering and neural radiance field training by dynamically adjusting sampling density based on local visual complexity or error metrics. It solves the inefficiency of uniform sampling by concentrating computational resources (rays or network evaluations) in regions with high gradients or large reconstruction errors while skipping or interpolating smooth areas. Unlike [Adaptive Screen Sampling](5838-adaptive-screen-sampling.md) which relies on simple color variance heuristics, or [Resolution Adaptation](3063-resolution-adaptation.md) which adjusts grid resolution, this family explicitly uses hierarchical data structures (quadtrees) or iterative error bounds to define the sampling regions.

## Shared principles
The core mechanism shared by all children is the deviation from a fixed, uniform sampling strategy. Instead of casting a constant number of rays or samples per pixel/ray, the method computes a metric (such as gradient magnitude, rendering error, or temporal difference) and uses it to modulate the sampling rate or resolution.

Mathematically, this often involves a control loop:
1.  **Metric Evaluation:** Compute a scalar value $M$ representing local complexity or error.
2.  **Adaptation:** Adjust a parameter $P$ (step size $\Delta t$, resolution $R$, or sampling probability $w$) based on $M$.
3.  **Rendering/Training:** Execute the volume integral or loss computation with the adjusted parameter.

Common adaptation laws include:
*   **Step Size Adaptation:** $\Delta t \propto \frac{1}{M}$ or $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, \frac{k}{M}))$ [S6, S8, S9, unofficial].
*   **Resolution Control:** $R_{new} = R_{old} + \alpha \cdot (t_{target} - t_{actual})$ or threshold-based switching [S3063, unofficial].
*   **Probability Weighting:** $w_i \propto M_i$ for importance sampling [S4696, unofficial].

All methods in this family aim to maintain visual fidelity while reducing the total number of volume queries or network evaluations [S1][S2, unofficial].

## Subtopics

### [Adaptive Refinement for Volume Rendering](1892-adaptive-refinement-for-volume-rendering.md)
Uses iterative convergence with either Finite Element Method (FEM) error bounds for rigorous quality or Level-of-Detail (LOD) feedback for interactive frame rates. It balances quality and performance by subdividing image regions based on error estimates.

### [Adaptive Screen Sampling](5838-adaptive-screen-sampling.md)
A heuristic image-order technique that adjusts ray counts per pixel based on local color variance. It is simpler than FEM approaches but may over-refine linear regions or fail in non-homogeneous scenes.

### [Adaptive Ray Shooting & Quadtree Subdivision](8158-adaptive-ray-shooting-quadtree-subdivision.md)
A training-time acceleration method for neural radiance fields. It subdivides training views into quadtrees based on rendering error to reduce ray redundancy, working with various radiance field architectures.

## Background
The concept of adaptive sampling in volume rendering originated with Levoy's work on hierarchical data structures and error estimation [S2, unofficial]. The specific application of quadtree subdivision for adaptive ray shooting in the context of neural radiance fields was introduced to address the computational cost of training multi-view radiance fields [S1].

## Sources
[S1] Adaptive Ray Shooting and Quadtree Subdivision for Training Radiance Fields, URL, date fetched
[S2] Adaptive Refinement for Volume Rendering, URL, date fetched
