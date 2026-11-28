# Proximal Algorithm

Also called: Proximal Methods

## Overview
Proximal algorithms are a family of optimization methods for non-smooth, constrained, distributed, or large-scale problems [S1]. They decompose objectives into modular sub-problems solved via proximal operators, enabling guaranteed convergence for convex composite objectives [S1].

## Shared principles
The core mechanism decomposes a global optimization problem into simpler sub-problems solved via a proximal operator [S1]. The proximal operator of a function $H(\cdot)$ is defined as [S1]:

$$
\text{prox}_{\lambda H}(u) = \arg \min_v H(v) + \frac{1}{2\lambda} \|v - u\|^2
$$

where $v, u \in \mathbb{R}^n$ and $\lambda \in \mathbb{R}$ is a regularization parameter [S1]. The parameter $\lambda$ controls the relevance of the proximity term $\|v - u\|^2$ [S1]. A smaller $\lambda$ prioritizes minimizing $H(v)$, while a larger $\lambda$ keeps the solution closer to the reference point $u$ [S1]. This operator gets closer to the minimum of $H(\cdot)$ while staying close to $u$ [S1]. Many functions have closed-form proximal operators, allowing efficient construction for complex problems [S1]. The general problem structure is [S1]:

$$
\min_{v,z} F(v) + G(z) \quad \text{subject to} \quad Kv = z
$$

where $F$ and $G$ are convex functions and $K$ is a linear operator [S1]. The algorithm solves this by iteratively applying proximal operators for $F$ and $G$ [S1].

## Subtopics

### [FISTA (Fast Iterative Shrinkage-Thresholding Algorithm)](18227-fista-fast-iterative-shrinkage-thresholding-algorithm.md)
An iterative optimization algorithm that accelerates convergence by combining gradient descent with Nesterov’s momentum. It handles non-smooth regularization (e.g., L1 norms) via proximal operators, making it suitable for tomographic reconstruction requiring sparsity or bounded support. It achieves an $O(1/k^2)$ convergence rate for convex objectives compared to $O(1/k)$ for standard gradient descent. See [FISTA](18227-fista-fast-iterative-shrinkage-thresholding-algorithm.md).

### Linearized Alternating Direction Method of Multipliers (LADMM)
A proximal algorithm variant that linearizes constraints to solve $\min F(v) + G(z)$ subject to $Kv=z$ [S1]. It is chosen when proximal operators for individual components are available but direct minimization is difficult due to coupling [S1]. See [LADMM](10111-ladmm.md).

## Background
The framework originates from convex analysis and optimization theory [S1]. It is applied in cryo-electron tomography (Cryo-ET) reconstruction to solve non-linear optimization problems with guaranteed convergence for convex objectives [S1]. Its modular nature suits large-scale problems where objectives split into data fidelity and regularization terms [S1]. The specific formulation and Cryo-ET application are detailed in [S1].

## Sources
[S1] GPU Accelerated 3D Tomographic Reconstruction and Visualization From Noisy Electron Microscopy Tilt-Series., https://doi.org/10.1109/tvcg.2022.3230445, fetched 2026-10-28
