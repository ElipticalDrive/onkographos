# Variance-Based Color Table Filtering

Also called: Variance-Based Interval Estimation

## Overview
Variance-Based Color Table Filtering is a technique for accurately filtering color table-based textures by estimating the noise value range $\delta$ from the loss in noise variance due to filtering [S3]. It addresses the error introduced when noise values and gradients are evaluated on unfiltered noise [S1]. Unlike methods that rely on affine arithmetic or shader simplification, this approach leverages the availability of an analytical expression for noise variance to determine the appropriate filtering interval [S3].

## How it works
The method relies on the premise that an analytical expression for the noise variance is available [S3]. By estimating the noise value range $\delta$ from the loss in noise variance caused by the filtering operation, the method determines the appropriate filtering interval [S1].

**Inputs:**
- Procedural noise function $N(x)$ for which an analytical variance expression is available [S3].
- Filtering kernel or sampling configuration [S1].

**Outputs:**
- Estimated noise value range $\delta$ [S3].
- Filtered color table values or texture samples [S1].

**Data Structures:**
- Not given in the sources. The method primarily requires the ability to evaluate the noise function and its analytical variance [S3].

**Steps:**
1. Obtain the analytical expression for the noise variance of the procedural noise function [S3].
2. Determine the loss in noise variance caused by the filtering operation [S3].
3. Estimate the noise value range $\delta$ from this variance loss [S3].
4. Apply the estimated range $\delta$ to achieve accurate filtering for color table-based textures [S3].

**Formulas:**
- Not given in the sources. The exact mathematical mapping from the variance loss to $\delta$ is not provided [S3].

**Parameters:**
- $\delta$: The noise value range, estimated from the loss in noise variance [S3].

**Edge Cases and Pitfalls:**
- The method requires an analytical expression of the noise variance to be available [S3]. If such an expression is not known or cannot be derived, this method cannot be applied [S3].
- Precision issues may arise if the variance loss is very small, making the estimation of $\delta$ sensitive to numerical errors [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1].
- **Memory Complexity:** Not given in the sources [S1].
- **Measured Performance:** Not given in the sources [S1].
- **Strengths:** Provides accurate filtering by directly accounting for the variance loss due to filtering [S3].
- **Limitations:** Requires an analytical expression for the noise variance, which may not be available for all procedural noise functions [S3].
- **When to use:** When accurate filtering of color table-based textures is required and an analytical variance expression for the noise function is available [S3].
- **When not to use:** When the noise function does not have a known analytical variance expression [S3].

## Background
The method was introduced by Lagae et al. [LLDD09a] as a way to achieve accurate filtering for color table-based textures [S3]. It is discussed in the context of analyzing procedural noise functions and addressing errors caused by evaluating noise values and gradients on unfiltered noise [S1]. The approach is contrasted with other methods such as affine arithmetic [HSS98] and automatic shader simplification [OKS03] [S1].

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S3] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
