# Multi-Resolution Analysis (MRA) conditions

Also called: MRA theory, Mallat-Meyer conditions

## Overview
Multi-Resolution Analysis (MRA) conditions are a set of mathematical axioms that define valid hierarchical representations for signals or functions, ensuring self-similarity, nested subspaces, and completeness across scales [S1]. What sets MRA apart from its siblings is that it provides the theoretical guarantee that allows for safe, coarse-to-fine integration of observations, enabling early stopping criteria and efficient memory usage in volumetric mapping frameworks like wavemap [S1].

## Shared principles
The MRA conditions define a sequence of closed subspaces $\{V_j\}_{j \in \mathbb{Z}}$ of the Lebesgue space $L^2(\mathbb{R})$, where $j$ is the scale index and $k$ is the translation index [S1]. The scale $2^j$ corresponds to the cell width at resolution level $j$ [S1]. The conditions are:

1.  **Self-similarity in space**: If a function $f(x)$ is in subspace $V_j$, then the shifted function $f(x - 2^j k)$ is also in $V_j$ for all integers $k$ [S1].
    $$ f(x) \in V_j \iff f(x - 2^j k) \in V_j \quad \text{for all } k \in \mathbb{Z} \quad (14) $$
2.  **Nested subspaces**: The subspace at a finer scale is a subset of the subspace at the next coarser scale [S1].
    $$ V_{j+1} \subset V_j \quad \text{for all } j \in \mathbb{Z} \quad (15) $$
3.  **Self-similarity in scale**: If $f(x)$ is in $V_j$, then the dilated function $f(x/2)$ is in $V_{j+1}$ [S1].
    $$ f(x) \in V_j \iff f(x/2) \in V_{j+1} \quad \text{for all } j \in \mathbb{Z} \quad (16) $$
4.  **Completeness at coarse scales**: The intersection of all subspaces as the scale becomes infinitely coarse contains only the zero element [S1].
    $$ \bigcap_{j=-\infty}^{\infty} V_j = \{0\} \quad (17) $$
5.  **Completeness at fine scales**: The closure of the union of all subspaces as the scale becomes infinitely fine is the entire space $L^2(\mathbb{R})$ [S1].
    $$ \lim_{j \to -\infty} V_j = \text{closure}\left(\bigcup_{j=-\infty}^{\infty} V_j\right) = L^2(\mathbb{R}) \quad (18) $$
6.  **Riesz basis**: The subspace $V_0$ admits a Riesz basis [S1]. In robotics applications, the most common Riesz basis consists of box functions arranged to span the cells of a regular grid [S1].

These conditions ensure that any function in $L^2(\mathbb{R})$ can be represented by refining the scale from coarse to fine, and that details can be extracted between consecutive resolutions [S1].

## Subtopics

### [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md)
Focuses on extracting the difference between consecutive resolutions ($W_j$) using the orthogonal complement structure derived from MRA.

### [Wavelet Tree Overview Map](5354-wavelet-tree-overview-map.md)
Provides a visualization technique for navigating the hierarchical structure defined by MRA conditions.

### [Wavelet Tree](5390-wavelet-tree.md)
Describes the data structure used to store the coefficients generated from the MRA decomposition.

### [Cardinal B-spline wavelets](11756-cardinal-b-spline-wavelets.md)
A specific choice of basis functions that satisfy the MRA conditions, offering smoothness properties distinct from box functions.

## Background
The MRA conditions were formalized by Mallat and Meyer [S1]. They generalize earlier multi-resolution representations such as Laplacian image pyramids introduced by Burt and Adelson [S1]. The theory is used as the foundation for hierarchical volumetric mapping frameworks, specifically wavemap, which leverages these conditions to integrate new observations in a coarse-to-fine manner [S1]. This approach allows for the use of complex sensor models and efficient memory usage in scene reconstruction tasks involving RGB-D and LiDAR data [S1].

## Sources
[S1] Efficient volumetric mapping of multi-scale environments ..., https://www.roboticsproceedings.org/rss19/p065.pdf, fetched 2026-10-26
