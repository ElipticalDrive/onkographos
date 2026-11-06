# Reconstruction and Compositing Techniques
## Overview
This family encompasses methods for evaluating continuous fields along rays (via polynomial composition) and resampling discrete volume data (via spatial convolution) to support direct volume ray casting [G]. It is distinguished from sibling methods like [Pre-Integration Lookup](2854-pre-integration-lookup.md) by focusing on the algebraic or spatial mechanics of data evaluation rather than lookup-table optimization or surface-based compositing [G].
## Shared principles
The methods in this family share the goal of mapping 3D data to 1D ray values or resampling grids, but they differ fundamentally in domain and mechanism [G]. [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md) operates in the algebraic domain, composing trivariate polynomials with linear ray parameterizations to produce univariate polynomials [G]. [Feed-Backward Reconstruction](2433-feed-backward-reconstruction.md) operates in the spatial domain, using convolution kernels to resample discrete signals [G]. No single mathematical formulation or data structure is shared across both; they are grouped here by their role in the reconstruction and compositing pipeline of volume rendering [G].
## Subtopics

### [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md)
Evaluates the composition of a trivariate polynomial field with a linear ray parameterization. It solves the problem of efficient numerical evaluation in direct volume ray casting by using convolution operators to reduce computation and register usage compared to naive substitution [G]. Unlike [Feed-Backward Reconstruction](2433-feed-backward-reconstruction.md), which focuses on reconstruction filters, this method specifically optimizes the algebraic evaluation of the polynomial composition itself. Choose this when the volume data is represented as trivariate polynomials and high-order continuity is required [G].

### [Feed-Backward Reconstruction](2433-feed-backward-reconstruction.md)
A spatial-domain convolution method that computes each output sample by gathering weighted contributions from neighboring input samples. It solves the problem of resampling discrete volume data onto a finer or different grid by centering the reconstruction kernel at the output location and summing the input values weighted by the kernel function. It is distinguished from its sibling [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md) by its explicit spatial gathering loop structure, which accesses each output sample once and each input sample multiple times, contrasting with feed-forward approaches that distribute input energy to multiple outputs. Choose this for standard resampling of discrete voxel grids [G].

### Source-Attenuation Reprojection
A technique by Vannier et al. that assigns a source strength and an attenuation coefficient (opacity) to each voxel [S1, unofficial]. It traces rays through the volume array until they hit a surface, then assigns an intensity inversely proportional to the distance traveled along the ray [S1, unofficial]. A surface is defined as a significant change in opacity value, making this approach threshold-based and susceptible to associated artifacts [S1, unofficial].

## Background
The concept of source-attenuation reprojection originates from work by Vannier et al. in the context of stereotactic neurosurgery planning [S1, unofficial]. These techniques are used in direct volume visualization pipelines where accurate field evaluation or resampling is critical for image quality [G].
## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
