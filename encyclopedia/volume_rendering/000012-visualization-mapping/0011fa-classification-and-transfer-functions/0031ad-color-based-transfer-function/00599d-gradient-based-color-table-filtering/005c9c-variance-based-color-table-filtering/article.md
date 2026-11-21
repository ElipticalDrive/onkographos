# Variance-Based Color Table Filtering

Also called: Variance-Based Interval Estimation

## Overview
Variance-Based Color Table Filtering is a technique for accurately filtering color table-based textures by estimating the noise value range $\delta$ from the loss in noise variance due to filtering [S2]. It addresses the error introduced when noise values and gradients are evaluated on unfiltered noise, which is a common source of inaccuracy in procedural noise rendering [S1]. Unlike methods that rely on affine arithmetic or shader simplification, this approach leverages the availability of an analytical expression for noise variance to determine the appropriate filtering interval [S2].

## How it works
The method relies on the premise that an analytical expression for the noise variance is available [S2]. By comparing the variance of the unfiltered noise with the variance after filtering, the method estimates the range of noise values $\delta$ that contributes to the filtered result [S1]. This estimated range is then used to adjust the color table lookup or filtering process to ensure accuracy [S2].

**Inputs:**
- Procedural noise function $N(x)$ for which an analytical variance expression $\text{Var}(N)$ is available [S2].
- Filtering kernel or sampling configuration [S1].

**Outputs:**
- Estimated noise value range $\delta$ [S2].
- Filtered color table values or texture samples [S1].

**Data Structures:**
- Not specified in the sources. The method primarily requires the ability to evaluate the noise function and its analytical variance [S2].

**Steps:**
1. Evaluate the analytical variance of the unfiltered noise function, denoted as $\text{Var}_{unfiltered}$ [S2].
2. Determine the variance of the noise after applying the desired filtering operation, denoted as $\text{Var}_{filtered}$ [S2].
3. Calculate the loss in noise variance due to filtering: $\Delta \text{Var} = \text{Var}_{unfiltered} - \text{Var}_{filtered}$ [S2].
4. Estimate the noise value range $\delta$ based on the calculated variance loss $\Delta \text{Var}$ [S1]. The exact mapping from $\Delta \text{Var}$ to $\delta$ depends on the specific analytical variance expression available for the noise function [S2].
5. Use the estimated range $\delta$ to adjust the color table filtering or sampling process to achieve accurate results [S1].

**Parameters:**
- $\delta$: The noise value range, estimated from the loss in noise variance [S2].

**Edge Cases and Pitfalls:**
- The method requires an analytical expression of the noise variance to be available [S2]. If such an expression is not known or cannot be derived, this method cannot be applied [S2].
- Precision issues may arise if the variance loss $\Delta \text{Var}$ is very small, making the estimation of $\delta$ sensitive to numerical errors [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Provides accurate filtering by directly accounting for the variance loss due to filtering [S2].
- **Limitations:** Requires an analytical expression for the noise variance, which may not be available for all procedural noise functions [S2].
- **When to use:** When accurate filtering of color table-based textures is required and an analytical variance expression for the noise function is available [S2].
- **When not to use:** When the noise function does not have a known analytical variance expression [S2].

## Background
The method was introduced by Lagae et al. [LLDD09a] as a way to achieve accurate filtering for color table-based textures [S2]. It is discussed in the context of analyzing procedural noise functions and addressing errors caused by evaluating noise values and gradients on unfiltered noise [S1]. The approach is contrasted with other methods such as affine arithmetic [HSS98] and automatic shader simplification [OKS03] [S1].

## Sources
[S1] A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S2] State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
