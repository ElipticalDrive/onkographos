# Spatial Subdivision and Hierarchical Sampling


## Overview
Spatial Subdivision and Hierarchical Sampling accelerates volume rendering and neural radiance field training by dynamically adjusting sampling density based on local visual complexity or error metrics [G]. It solves the inefficiency of uniform sampling by concentrating computational resources (rays or network evaluations) in regions with high gradients or large reconstruction errors while skipping or interpolating smooth areas [G]. Unlike [Adaptive Screen Sampling](5838-adaptive-screen-sampling.md) which relies on simple color variance heuristics, or [Resolution Adaptation](3063-resolution-adaptation.md) which adjusts grid resolution, this family explicitly uses hierarchical data structures (quadtrees) or iterative error bounds to define the sampling regions [G].

## Shared principles
Builds on the shared principles of [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md). All methods in this family partition the image plane or volume using hierarchical data structures (typically quadtrees) to compute a local metric (gradient magnitude, rendering error, or color variance) that modulates sampling density or resolution [G]. The adaptation follows a control loop: compute a scalar metric $M$ representing local complexity or error [G]; adjust a parameter $P$ (step size $\Delta t$, resolution $R$, or sampling probability $w$) based on $M$ [G]; execute the volume integral or loss computation with the adjusted parameter [G]. Common adaptation laws include step size adaptation $\Delta t \propto 1/M$, resolution control via threshold-based switching, and probability weighting $w_i \propto M_i$ for importance sampling [G]. All methods aim to maintain visual fidelity while reducing the total number of volume queries or network evaluations [G].

## Subtopics

### [Adaptive Refinement for Volume Rendering](1892-adaptive-refinement-for-volume-rendering.md)
Uses iterative convergence with either Finite Element Method (FEM) error bounds for rigorous quality or Level-of-Detail (LOD) feedback for interactive frame rates. It balances quality and performance by subdividing image regions based on error estimates.

### [Adaptive Screen Sampling](5838-adaptive-screen-sampling.md)
A heuristic image-order technique that adjusts ray counts per pixel based on local color variance. It is simpler than FEM approaches but may over-refine linear regions or fail in non-homogeneous scenes.

### [Adaptive Ray Shooting & Quadtree Subdivision](8158-adaptive-ray-shooting-quadtree-subdivision.md)
A training-time acceleration method for neural radiance fields. It subdivides training views into quadtrees based on rendering error to reduce ray redundancy, working with various radiance field architectures.

## Background
The concept of adaptive sampling in volume rendering originated with Levoy's work on hierarchical data structures and error estimation [G]. The specific application of quadtree subdivision for adaptive ray shooting in the context of neural radiance fields was introduced to address the computational cost of training multi-view radiance fields [G].

## Sources
[S1] Adaptive Ray Shooting and Quadtree Subdivision for Training Radiance Fields, URL, date fetched
[S2] Adaptive Refinement for Volume Rendering, URL, date fetched
