# Distribution Families for Differentiable Rendering

Also called: Differentiable Sigmoids, Soft CDF/PDF Families

## Overview
This family provides a collection of smooth, differentiable cumulative distribution functions (CDFs) and probability density functions (PDFs) used as transfer functions in differentiable volume rendering [S1]. It replaces non-differentiable step functions with continuous, parameterized curves to allow gradients to flow through opacity assignment during backpropagation [G]. What sets this family apart from its siblings is the explicit enumeration of diverse mathematical families (e.g., Cauchy, Gumbel, Gamma) and their variants (reversed, square-root) to provide a unified library of basis functions for transfer function design, rather than deriving a single statistical model from data [G].

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

## Subtopics

### Standard CDF Families
Basic distributions defined by their cumulative distribution functions, such as Logistic, Cauchy, and Gumbel-Max [S1]. These serve as foundational smooth approximations to the Heaviside step function [G]. [Link](814-frequency-distribution-based-transfer-function.md)

### Reversed and Square-Root Variants
A reversed CDF is defined as $F_{\text{Rev.}}(x) = 1 - F(-x)$, which equals $F(x)$ for symmetric distributions [S1]. The square-root distribution $F_{\text{sq}}$ is defined in terms of $F$ as in Equation (5) of the source paper [S1]. These variants allow for different transition shapes and gradient profiles [G]. [Link](1812-gaussian-sum-transfer-function.md)

### Parameterized Families
Distributions that include shape parameters, such as the Gamma distribution with shape parameter $p > 0$ [S1]. These allow for tuning the skewness and width of the transition region beyond simple scaling [G]. [Link](5458-clustering-based-gray-gradient-transfer-function.md)

## Background
The specific set of distribution families and their mathematical definitions (including the Cauchy, Reciprocal, Heaviside, Gumbel-Max, Uniform, Cubic Hermite, Levy, Wigner Semicircle, and Gamma distributions) were formalized in the context of the GenDR (Generalized Differentiable Renderer) framework [S1]. The approach extends differentiable rendering techniques to a comprehensive library of statistical distributions for transfer function design [G].

## Sources
[S1] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
