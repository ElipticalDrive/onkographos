# Splitting Mechanics and Density Preservation

Also called: Gaussian Splitting, Density-Preserving Split

## Overview
This article defines the geometric and parametric mechanics for splitting a parent 3D Gaussian into two children and the strategies for preserving integrated density during this operation. It solves the problem of introducing high-frequency detail by replacing a single large primitive with two smaller, adjacent primitives without causing artifacts from opacity or density mismatches . It is distinguished from cloning (exact duplication) and naive splitting (which may cause loss perturbations) by explicitly adjusting child parameters to match the parent's integrated density .

## Shared principles
All splitting variants in this family share the following mechanism:
1.  **Axis Selection:** The split occurs along the principal axis with the largest scale value $\sigma_{max}$ of the parent Gaussian [Parent].
2.  **Mean Shift:** The means of the two children are shifted along this axis by a distance $\Delta = \alpha \cdot \sigma_{max}$, where $\alpha$ is a constant [Parent].
3.  **Inheritance:** Rotation and Spherical Harmonic (SH) coefficients are copied from the parent to both children [Parent].
4.  **Density Preservation:** The scales and opacities of the children are adapted such that the sum of their densities approximates the parent's density, preventing artifacts .

## Subtopics

### [3DGS Adaptive Density Control](7847-3dgs-adaptive-density-control.md)
Uses standard split-and-clone. Splits "large" Gaussians by shifting means and resetting opacity/scale, clones "small" ones.

### [Density-preserving Gaussian splitting](9290-density-preserving-gaussian-splitting.md)
Focuses on preserving integrated density during the split operation using an adaptation function that accounts for opacity variations. Discards cloning.

### [Adaptive Density Control (Split & Clone)](11878-adaptive-density-control-split-clone.md)
Dual-mode approach. Splits over-reconstructed Gaussians and clones under-reconstructed ones. Uses standard density preservation approximations.

## Background
Originating from the 3D Gaussian Splatting framework , splitting mechanics were introduced to densify sparse initializations. The density-preserving variant was later proposed to mitigate loss perturbations and cluster formation caused by naive splitting .

## Sources

[S1] Density-preserving Gaussian splitting notes, URL not provided, date fetched not provided.
[S2] 3DGS Adaptive Density Control notes, URL not provided, date fetched not provided. (unofficial)
