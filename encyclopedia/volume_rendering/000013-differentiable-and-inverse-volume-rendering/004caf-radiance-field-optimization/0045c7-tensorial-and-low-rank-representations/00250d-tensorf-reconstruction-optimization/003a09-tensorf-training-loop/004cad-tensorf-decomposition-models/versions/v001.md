# TensoRF Decomposition Models

Also called: CP model, HY model, HY-S model, Rank-Residual Decomposition Models (CP, HY-S)

## Overview
This family defines the factorization structures for the radiance field tensor in TensoRF, balancing storage efficiency against rendering quality by decomposing the 3D field into lower-rank components. Unlike the standard Vector-Matrix (VM) decomposition which enforces a fixed 1:1 ratio of vector to matrix ranks, these models allow arbitrary adjustment of this ratio, providing greater flexibility in model size control [S1].

## Shared principles
The core mechanism is **Rank-Residual Decomposition**. The radiance field is represented as a sum of groups, where each group contributes a specific rank to the overall factorization. This allows the model to start with a base configuration and incrementally add complexity (residuals) to improve quality without retraining from scratch [S1].

The decomposition separates the field into **Density** ($\sigma$) and **Color** ($c$) components. Each component is factorized into a set of basis vectors or matrices along the spatial axes. The total rank for a component is the sum of ranks contributed by the base group and any subsequent residual groups.

Key parameters include:
*   **Vector-based ranks**: Represented by 1D basis vectors.
*   **Matrix-based ranks**: Represented by 2D basis matrices.
*   **Groups ($M$)**: The number of additive components in the decomposition.
*   **Ratio**: The proportion of vector-based ranks to matrix-based ranks.

## Subtopics

### CP Model
The CP (Canonical Polyadic) model uses only vector-based rank components for both density and color. It minimizes storage footprint but requires higher total ranks to achieve comparable rendering quality, leading to increased GPU memory usage and slower querying speeds [S1].

*   **Configuration**:
    *   Density: 96 vector ranks.
    *   Color: 384 vector ranks.
    *   Groups: $M=4$. Each group contains 96 color ranks.
*   **When to choose**: When storage is the primary constraint and query speed is less critical.

### HY Model
The HY (Hybrid) model incorporates both vector- and matrix-based components. Matrix components consume more storage per rank but efficiently reduce the total number of ranks required for high quality, thereby lowering GPU memory usage compared to the CP model [S1].

*   **Configuration**:
    *   Density: 64 vector components, 16 matrix components.
    *   Color: 256 vector components, 64 matrix components.
    *   Groups: $M=4$. Each group contains 64 vector components and 16 matrix components for color.
*   **When to choose**: When a balance between storage, memory usage, and rendering quality is needed.

### HY-S Model
The HY-S (Hybrid-Scalable) model extends the HY structure to allow adjustable model size over a larger range. It starts with a base of vector components and gradually appends matrix components in subsequent groups [S1].

*   **Configuration**:
    *   Base: Density and color initially use 96 vector components.
    *   Residuals: Four additional groups for color append matrix components with ranks $\{4, 16, 32, 64\}$ gradually.
    *   Total Groups: $M=5$ (the first group has only 96 vector components; the subsequent four add matrix components).
*   **When to choose**: When flexible scaling of model capacity is required without retraining.

### Comparison with VM Decomposition
The primary distinction between these models and the standard TensoRF VM decomposition is the rank ratio flexibility. VM decomposition requires a fixed 1:1 ratio of vector- to matrix-based ranks, limiting control over model size [S1]. The HY family allows arbitrary adjustment of this ratio, enabling more precise tuning of the trade-off between compression and quality [S1].

## Background
The Rank-Residual Decomposition models were introduced in "Compressible-Composable NeRF Via Rank-Residual Decomposition" [S1]. The work demonstrates that these methods reach near-optimal quality without needing retraining for optimal models, unlike baseline sort-and-truncate strategies [S1]. Training is typically performed for 30,000 iterations using the Adam optimizer with an initial learning rate of 0.02 for factorized matrices and 0.001 for singular values [S1].

## Sources
[S1] Compressible-Composable NeRF Via Rank-Residual Decomposition, https://doi.org/10.52202/068431-1076, fetched 2026-10-26
