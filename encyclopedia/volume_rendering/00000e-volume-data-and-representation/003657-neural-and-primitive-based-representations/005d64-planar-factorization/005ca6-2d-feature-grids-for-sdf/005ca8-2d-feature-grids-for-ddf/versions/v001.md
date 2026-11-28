# 2D Feature Grids for DDF
Also called: DDF Grid Factorization, Factorized 6D Grid
## Overview
A memory-efficient neural scene representation that factorizes a 6D feature grid into 15 separate 2D grids to represent Directional Distance Functions (DDF) [S1]. It solves the $O(N^6)$ memory cost of a full 6D grid by assuming independence between feature distributions in orthogonal planes, enabling fast inverse rendering by replacing iterative sphere tracing with a single forward pass [S1]. It differs from its parent [2D Feature Grids for SDF](17216-2d-feature-grids-for-sdf.md) by operating on the 6D input space $(p, r) \in \mathbb{R}^6$ (point on unit sphere + direction) rather than the 3D space $\mathbb{R}^3$, allowing explicit encoding of directional information to resolve surface geometry along specific viewing directions without spatial ambiguity [S1].

## How it works
**Inputs and Outputs:**
- Input: A 6D tuple $(p, r)$ where $p = (p_x, p_y, p_z) \in S^2$ is a point on the unit sphere and $r = (r_x, r_y, r_z) \in S^2$ is a direction vector [S1].
- Output: A feature vector $f^{pd}(p, r) \in \mathbb{R}^{15 \times K}$, where $K$ is the feature dimension [S1].

**Data Structures:**
- 15 feature grids, denoted $M_{p_xp_y}, \dots, M_{r_zr_x}$ [S1].
- Layout: $\mathbb{R}^{15 \times N \times N \times K}$ [S1].
- Each grid stores $K$-dimensional features for a specific 2D projection of the 6D space [S1].
- Resolution $N$ is not given in the sources.

**Steps:**
1. Factorize the 6D input tuple $(p_x, p_y, p_z, r_x, r_y, r_z)$ into all possible 2D coordinate pairs. There are $\binom{6}{2} = 15$ such pairs [S1]:
   $(p_x, p_y), (p_y, p_z), (p_z, p_x), (r_x, r_y), (r_y, r_z), (r_z, r_x), (p_x, r_x), (p_y, r_y), (p_z, r_z), (p_x, r_y), (p_y, r_z), (p_z, r_x), (p_x, r_z), (p_y, r_x), (p_z, r_y)$ [S1].
2. For each of the 15 pairs, sample the corresponding 2D feature grid using bilinear interpolation to obtain a $K$-dimensional vector [S1].
   - Let $m_{p_xp_y} \sim M_{p_xp_y}$ denote the interpolated feature from grid $M_{p_xp_y}$ at coordinates $(p_x, p_y)$ [S1].
   - Bilinear interpolation samples 4 neighboring features and interpolates between them [S1].
3. Concatenate the 15 interpolated $K$-dimensional vectors to form the output [S1]:
   $$f^{pd}(p, r) = (m_{p_xp_y}, \dots, m_{r_zr_x}) \in \mathbb{R}^{15 \times K}$$ [S1].

**Parameters:**
- Number of 2D grids: 15 [S1].
- Feature dimension $K$: Not given in the sources.
- Grid resolution $N$: Not given in the sources.

**Edge Cases and Pitfalls:**
- The method assumes the function is linear in the high-dimensional feature space [S1].
- Without factorization, memory scales as $O(N^6)$, which is computationally inefficient [S1].
- The grids are learned per shape class, not per object [S1].

## Subtopics

### [DDF Model (fd)](17229-ddf-model-fd.md)
The DDF model is a neural scene representation that predicts the directional distance to an object's surface and the probability of a ray hitting that surface for points on the unit sphere. It replaces iterative sphere tracing with a single forward pass, accelerating inverse rendering algorithms by reducing network evaluations per iteration. It differs from standard SDF models by explicitly encoding directional information via factorized 2D feature grids, allowing it to resolve surface geometry along specific viewing directions without spatial ambiguity. Link: [DDF Model (fd)](17229-ddf-model-fd.md).

### DiF-Grid-SL
A minor variant using $N=2$ factors: Factor 1 consists of 3D Grids and Factor 2 consists of 3D Grids [S2, unofficial]. It employs a Sawtooth coordinate transformation and uses a Single Level structure [S2, unofficial].

## Operating characteristics
- **Memory Complexity:** Scales quadratically $O(N^2)$ with grid resolution per grid, but total memory is proportional to the number of grids (15) times $N^2 K$ [S1]. This avoids the $O(N^6)$ cost of a full 6D grid [S1].
- **Time Complexity:** Requires 15 bilinear interpolations per query [S1].
- **Strengths:** Reduces rendering times for representing complex shapes [S1]. Enables fast inverse rendering by avoiding iterative sphere tracing [S1].
- **Limitations:** Assumes independence between feature distributions in orthogonal planes [S1]. Learned per shape class, not per object [S1].
- **When to use:** When representing directional distance functions for inverse rendering with limited memory [S1].
- **When not to use:** When per-object adaptation is required without retraining [S1].

## Background
Originated in the FIRe paper (Fast Inverse Rendering Using Directional and Signed Distance Functions) [S1]. The method builds on the observation that high-dimensional features stored in high-resolution grids reduce rendering times [S1]. It extends the 2D feature grid concept from SDFs (3D input) to DDFs (6D input) by factorizing the 6D space into 2D projections [S1].

## Sources
[S1] FIRe: Fast Inverse Rendering Using Directional and Signed Distance Functions, https://www.youtube.com/watch?v=Qj0ymDlRYZs, fetched 2026-10-30
[S2] Factor Fields: A Unified Framework for Neural Fields and Beyond, https://doi.org/10.48550/arxiv.2302.01226, fetched 2026-10-26 (unofficial)
