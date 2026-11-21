# TensoRF Decomposition Models
Also called: CP model, HY model, HY-S model, Rank-Residual Decomposition Models (CP, HY-S)

## Overview
This family defines the tensor decomposition strategies used to factorize the radiance field density and color fields in TensoRF, balancing storage efficiency against rendering quality [S1]. It differs from sibling methods like [NeuManifold](9771-neumanifold.md) or [StyleTRF](11815-styletrf.md) by explicitly structuring the low-rank approximation into discrete rank groups (rank-residual learning) and offering specific variants (CP, HY, HY-S) to control the ratio of vector- to matrix-based components [S1].

## Shared principles
The core mechanism is **Rank-Residual Decomposition**, where the radiance field is approximated as a sum of $M$ groups of components. Each group $m$ contains a specific number of rank components for density and color. The decomposition supports two types of components:
1.  **Vector-based (CP):** Uses only 1D vectors for each spatial dimension.
2.  **Matrix-based (HY):** Uses 1D vectors for one dimension and 2D matrices for the other two, allowing for higher local correlation capture with fewer total ranks [S1].

The total field value at a point is the sum of contributions from all groups. The model size is controlled by the number of groups $M$ and the rank distribution within each group. The decomposition is optimized using an Adam optimizer with different learning rates for factorized matrices and singular values [S1].

## Subtopics

### [Factorized Radiance Field with Fixed View-Dependent Components](17971-factorized-radiance-field-with-fixed-view-dependent-componen.md)
Reconstructs a radiance field by factorizing geometry and appearance into $n$ components (one per input view) using Vector-Matrix (VM) decomposition and a global appearance dictionary. It differs from the standard TensoRF families by binding components to specific views to enable robust point cloud fusion.

### CP Model
Uses only vector-based rank components. It requires higher ranks (96 for density, 384 for color) to achieve quality, leading to higher GPU memory usage and slower querying, but offers lower storage [S1].

### HY Model
Uses both vector- and matrix-based components. It reduces the required number of ranks (64 vector/16 matrix for density; 256 vector/64 matrix for color), thereby reducing GPU memory usage at the cost of slightly higher storage [S1].

### HY-S Model
A scalable variant of HY that allows adjustable model size. It starts with 96 vector components for both density and color, then gradually appends matrix components (4, 16, 32, 64) across 5 groups, enabling flexible trade-offs between quality and resource usage [S1].

## Background
The Rank-Residual Decomposition models were introduced in "Compressible-Composable NeRF Via Rank-Residual Decomposition" [S1]. The work addresses the limitations of the standard TensoRF VM decomposition, which requires a fixed 1:1 ratio of vector- to matrix-based ranks, limiting flexibility in model size control [S1]. The HY decomposition allows arbitrary adjustment of this ratio, and the HY-S model provides a scalable range of model sizes without requiring retraining for optimal configurations [S1].

## Sources
[S1] Compressible-Composable NeRF Via Rank-Residual Decomposition, https://doi.org/10.52202/068431-1076, fetched 2026-10-26
