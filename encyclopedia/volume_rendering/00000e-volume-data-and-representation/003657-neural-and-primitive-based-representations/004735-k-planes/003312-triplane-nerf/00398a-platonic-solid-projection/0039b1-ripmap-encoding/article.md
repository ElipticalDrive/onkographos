# Ripmap Encoding

Also called: anisotropic Mipmap

## Overview
Ripmap Encoding is a featurization method for anisotropic 2D areas projected onto the faces of a Platonic solid, enabling precise and efficient anisotropic area-sampling in neural rendering [S1]. It distinguishes itself from isotropic area-sampling methods like Tri-MipRF by using anisotropic kernels to characterize ellipsoidal footprints, preventing ambiguities and blurriness that arise when different viewing cones project to the same isotropic area [S1]. It builds on [Platonic Solid Projection](8518-platonic-solid-projection.md) by providing the hierarchical data structure required to resolve the anisotropy introduced by the projection.

## How it works
**Inputs:**
- Base level feature grid $R_{0,0}$: A 2D grid with shape $H \times W \times C$, where $H$ is height, $W$ is width, and $C$ is the number of channels [S1].
- Query parameters for a 2D Gaussian: Mean position $(p_x, p_y)$, covariance matrix $\Sigma_{proj}$, hyper-parameter $w$, and scene radius $r$ [S1].

**Outputs:**
- Feature vector $f$ obtained via tetra-linear interpolation from the Ripmap Encoding [S1].

**Data Structures:**
The Ripmap Encoding $R$ is a hierarchical structure containing $L \times L$ levels [S1].
- **Base Level ($R_{0,0}$):** The only learnable level, stored as a dense 2D feature grid $F$ of shape $H \times W \times C$ [S1].
- **Derived Levels ($R_{i,j}$):** Constructed from the base level using anisotropic average pooling. Indices $i$ and $j$ represent the level indices in the $x$ and $y$ directions, respectively [S1].

**Construction Steps:**
1. Initialize the base level $R_{0,0}$ as the learnable feature grid $F$ [S1].
2. Construct higher levels using anisotropic average pooling operations defined by the indices $i$ and $j$ [S1]:
   - If $j \neq 0$: $R_{i,j} = \text{Avg}_{2\times1}(R_{i, j-1})$ [S1].
   - If $i \neq 0$ and $j = 0$: $R_{i,j} = \text{Avg}_{1\times2}(R_{i-1, j})$ [S1].
   - Otherwise (if $i=0$ and $j=0$): The value is the base level $F$ [S1].
   - Note: The source defines the recurrence for $i \neq 0, j \neq 0$ implicitly by the base cases, but explicitly states "otherwise undefined" for the base case logic in the provided text snippet, implying the grid is filled row/column by row/column based on the anisotropic rules [S1]. Specifically, moving in the $y$-direction (index $j$) uses $2\times1$ pooling (averaging 2 pixels in $x$, 1 in $y$), and moving in the $x$-direction (index $i$) uses $1\times2$ pooling (averaging 1 pixel in $x$, 2 in $y$) [S1].

**Querying and Interpolation:**
To featurize an anisotropic 2D Gaussian, the method queries the Ripmap Encoding using tetra-linear interpolation [S1].

1. **Calculate Query Levels:**
   Determine the level indices $(l_x, l_y)$ based on the standard deviations of the projected 2D Gaussian $\Sigma_{proj}$ [S1]:
   $$ l_d = \log_2 \left( \frac{w \sigma_d}{r} \right), \quad d \in \{x, y\} $$
   where:
   - $\sigma_x, \sigma_y = \text{diag}(\Sigma_{proj})$ are the standard deviations along the $x$ and $y$ axes [S1].
   - $w$ is a hyper-parameter adjusting the probability mass coverage of the Gaussian footprint [S1].
   - $r$ is the radius of the bounding sphere for the reconstructed scene [S1].
   - $l_x, l_y$ may be fractional, requiring interpolation between levels [S1].

2. **Tetra-linear Interpolation:**
   Query the feature vector $f$ at position $(p_x, p_y)$ and level $(l_x, l_y)$ [S1]:
   $$ f = R(p_x, p_y, l_x, l_y) $$
   This involves interpolating spatially within the grid and interpolating between the discrete levels $R_{i,j}$ and $R_{i+1,j}$ (and similarly for $y$) to handle fractional level indices [S1]. The formal mathematical expression for tetra-linear interpolation is provided in the source's supplementary material but is not detailed in the main text [S1].

**Parameters:**
- $w$: Hyper-parameter to adjust how much probability mass of the Gaussian footprint is covered [S1].
- $r$: Radius of the bounding sphere for the reconstructed scene [S1].

**Edge Cases and Pitfalls:**
- The level indices $l_x, l_y$ must be clamped to the valid range $[0, L-1]$ to avoid out-of-bounds access, though the source does not specify the clamping strategy [G].
- The anisotropic pooling direction is fixed: $y$-level increments use horizontal averaging ($2\times1$), and $x$-level increments use vertical averaging ($1\times2$) [S1]. This assumes the coordinate system aligns with the pooling axes.

## Operating characteristics
- **Memory Complexity:** The encoding reduces memory consumption from $O(n^3)$ to $O(n^2)$ by factorizing 3D space into 2D planes [S1]. The Ripmap itself stores $L \times L$ levels, but since higher levels are derived from the base level, only the base level $R_{0,0}$ needs to be stored in memory, keeping the memory footprint equivalent to a single $H \times W \times C$ grid [S1].
- **Performance:** The method achieves state-of-the-art rendering quality on public benchmarks and real-world datasets, with sharper details on repetitive patterns compared to isotropic methods [S1].
- **Strengths:** Precisely characterizes anisotropic ellipsoidal footprints, preventing blurriness and ambiguities present in isotropic Mipmap approaches like Tri-MipRF [S1].
- **Limitations:** Requires the construction of the hierarchical structure (or on-the-fly derivation) and tetra-linear interpolation, which may add computational overhead compared to simple bilinear sampling [S1].

## Background
Ripmap Encoding was introduced in the paper "Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids" [S1]. It is inspired by the Ripmap technique by McReynolds et al. (1998), which dynamically adjusts the aspect ratio of pre-filtering kernels based on the angle of incidence [S1]. The method is part of the Rip-NeRF framework, which combines Platonic Solid Projection with this encoding to handle anisotropic sampling in neural radiance fields [S1].

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
