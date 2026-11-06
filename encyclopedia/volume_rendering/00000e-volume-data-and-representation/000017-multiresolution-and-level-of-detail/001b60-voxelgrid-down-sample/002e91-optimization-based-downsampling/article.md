# Optimization-based Downsampling
Also called: SGGX-based volume downsampling, Zhao, Wu et al. downsampling

## Overview
Optimization-based downsampling is a method for reducing the resolution of microflake volumes that preserves appearance by iteratively optimizing scattering parameters to match a high-resolution reference [S1]. It solves the problem of geometric aliasing and shadowing artifacts in aggregated microgeometry, distinguishing itself from naive averaging or simple SGGX approximation by explicitly accounting for shadowing effects through image-space error minimization [S1].

## How it works
The method generates low-resolution microflake volumes by optimizing scattering parameters such that the rendered appearance of the lower-resolution level of detail (LoD) matches the reference high-resolution volume [S1].

**Inputs and Outputs:**
*   **Input:** High-resolution microflake volume data (typically defined by Scalable Generalized Gaussian eXtension (SGGX) distributions or similar microflake parameters) [S1].
*   **Output:** Low-resolution microflake volume data with optimized scattering parameters [S1].

**Data Structures:**
The source does not specify the exact bit-packing or field layout of the microflake parameters. It assumes the existence of a parameter set representing scattering properties for each voxel [S1].

**Algorithm Steps:**
1.  **Clustering:** To make the optimization tractable, low-resolution voxels are assigned to a few clusters that share scattering parameters [S1]. This reduces the dimensionality of the optimization problem.
2.  **Initialization:** Initialize the scattering parameters for the clusters. The source notes that linear downsampling is not used for density parameters in related constrained approaches, but does not specify the initial values for the Zhao et al. method [S1].
3.  **Rendering:** Perform large-scale training rendering of the current low-resolution volume approximation [S1].
4.  **Error Calculation:** Compute image-space errors between the rendered low-resolution volume and the reference high-resolution volume [S1]. The error assumes that each pixel value depends mostly on the first non-empty voxels seen directly [S1].
5.  **Optimization:** Iteratively optimize the scattering parameters of the clusters to minimize the image-space error [S1].
6.  **Convergence:** Stop the optimization when the appearance of the LoDs matches the reference volume [S1].

**Formulas:**
The source does not provide explicit mathematical formulas for the scattering parameters, the clustering assignment, or the image-space error metric [S1].

**Parameters:**
*   **Cluster Count:** The number of clusters is not specified, but must be small enough to make optimization tractable [S1].
*   **Convergence Criteria:** "Appearance matches reference" is the stopping condition, but no quantitative threshold is given [S1].

**Edge Cases and Pitfalls:**
*   **Intractability:** The enormous number of parameters makes the problem intractable without clustering [S1].
*   **Spatial Variation Artifacts:** Voxel clustering leads to artifacts when input datasets have spatially varying scattering parameters [S1].
*   **Semi-Transparency:** The optimization using image-space errors assumes direct visibility of the first non-empty voxels. It is unclear how the algorithm adapts to semi-transparent datasets or datasets with both transparent and very dense voxels [S1].

## Subtopics

### [SGGX-based Downsampling](6179-sggx-based-downsampling.md)
SGGX-based Downsampling is a method for reducing the resolution of microflakes normal distributions in volume data by approximating a set of fine-scale Scalable Generalized Gaussian eXtension (SGGX) distributions with a coarser set. It solves the problem of geometric aliasing and loss of anisotropic detail when aggregating microgeometry, distinguishing itself from naive averaging by preserving the directional coherence of surface normals. This approach serves as the foundational geometric simplification step for higher-level optimization-based downsampling [HDCD15].

## Operating characteristics
*   **Time Complexity:** The method is computationally expensive due to iterative large-scale training rendering and optimization [S1].
*   **Memory Complexity:** Requires storing high-resolution reference data and low-resolution cluster parameters [S1].
*   **Performance:** Produces accurate LoDs for challenging datasets including complex multi-fiber fabrics [S1].
*   **Strengths:** Accounts for shadowing effects; preserves appearance at all scales [S1].
*   **Limitations:** Intractable without clustering; artifacts with spatially varying parameters; unclear applicability to semi-transparent or mixed-density datasets [S1].
*   **When to use:** When accurate appearance preservation with shadowing is required for microflake volumes and computational cost is acceptable [S1].
*   **When not to use:** For semi-transparent datasets or datasets with mixed transparent/dense voxels where the direct-visibility assumption fails [S1].

## Background
The method was proposed by Zhao, Wu et al. [ZWDR16] as an extension of Heitz et al.'s work on downsampling microflakes normal distributions [HDCD15]. It addresses the limitations of simple RGBA downsampling [KB08] and anisotropic voxel downsampling for real-time global illumination [CNS* 11] by specifically targeting microflake volumes with shadowing effects [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
