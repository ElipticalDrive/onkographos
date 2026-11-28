# Coarse Alignment with Optical Flow
Also called: Monocular Depth Scale Alignment via Optical Flow

## Overview
Coarse Alignment with Optical Flow is a preprocessing step in 3D Gaussian Splatting pipelines that aligns relative monocular depth estimates across multiple views by optimizing per-view scale and offset parameters. It solves the problem of multi-view inconsistency caused by the relative nature of monocular depth, which leads to significant misalignments of initialized 3D Gaussians representing the same surfaces [S1]. Unlike Multi-View Stereo (MVS) methods, which can produce artifacts with sparse images and large baselines, this method leverages optical flow correspondences to enforce geometric coherence without requiring dense point clouds [S1].

## How it works
**Inputs:**
- A set of $N$ input images.
- Monocular depth maps $D^m_i$ for each view $i$, estimated by a monocular depth estimation network (e.g., Yang et al. [60] as cited in [S1]).
- Camera poses for each view.
- Optical flow field $M_{i \rightarrow j}$ representing pixel correspondences from view $i$ to view $j$.

**Outputs:**
- Optimized scale parameters $\mathbf{s}^*$ and offset parameters $\mathbf{o}^*$ for each view.
- Aligned initial depth maps $D^{\text{init}}_i$.

**Data Structures:**
- $\mathbf{s}$: Vector of scale factors, one per view.
- $\mathbf{o}$: Vector of offset factors, one per view.
- $D^m_i[\mathbf{p}]$: Depth value at pixel $\mathbf{p}$ in view $i$.
- $M_{i \rightarrow j}$: Binary or confidence-weighted mask indicating valid flow correspondences from view $i$ to view $j$.
- $g(d, \mathbf{p})$: Function projecting a depth value $d$ at pixel $\mathbf{p}$ in view $i$ to the corresponding pixel coordinate $\mathbf{q}$ in view $j$ using the known camera geometry (intrinsics and extrinsics).

**Algorithm:**
1. **Compute Optical Flow:** Calculate optical flow fields $M_{i \rightarrow j}$ for all relevant pairs of views $(i, j)$. This establishes pixel-wise correspondences between images [S1].
2. **Define Objective:** Minimize the L1 difference between the projected depth of view $i$ and the depth of view $j$ at corresponding pixels, weighted by the flow mask. The objective function is:
   $$ \mathbf{s}^*, \mathbf{o}^* = \arg \min_{\mathbf{s}, \mathbf{o}} \sum_{(i,j)}\sum_{\mathbf{p}} \left\| M_{i \rightarrow j} \odot \left( g(s_i \cdot D^m_i[\mathbf{p}] + o_i, \mathbf{p}) - g(s_j \cdot D^m_j[\mathbf{q}] + o_j, \mathbf{q}) \right) \right\|_1 $$
   where $\mathbf{q}$ is the pixel coordinate in view $j$ corresponding to $\mathbf{p}$ in view $i$ via the optical flow [S1].
3. **Optimize:** Solve for $\mathbf{s}$ and $\mathbf{o}$ using gradient-based optimization. The optimization adjusts the scale and offset of each monocular depth map to minimize the reprojection error between views [S1].
4. **Apply Alignment:** Compute the final aligned depth for each view $i$ using the optimized parameters:
   $$ D^{\text{init}}_i = s_i \cdot D^m_i + o_i $$
   This aligned depth is then used to initialize the 3D Gaussians [S1].

**Parameters:**
- Optimization method: Gradient descent (specifics not given in sources) [S1].
- Loss function: L1 norm [S1].

**Edge Cases and Pitfalls:**
- **Flow Accuracy:** The method assumes the optical flow is accurate. Areas with inaccurate flow are masked out by $M_{i \rightarrow j}$, leaving their depth unaltered, which may result in residual misalignments in those regions [S1].
- **Sparse Views:** While MVS struggles with sparse views and large baselines, this method relies on monocular depth which may have larger errors in textureless regions, though it is noted to complement decoder-based approaches better than MVS in such cases [S1].

## Subtopics

### SUDS Baseline Configuration
This variant refers to the experimental setup used to evaluate the SUDS method on the City-1M benchmark. It involves computing optical flow for all sequences using RAFT, aligning auxiliary inputs like depth with the main CoherentGS method, and deactivating the DINO feature distillation branch used for semantic reconstruction in SUDS [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Avoids artifacts associated with MVS in sparse view scenarios with large baselines [S1]. Provides a coherent representation that supports regularized optimization [S1].
- **Limitations:** Relies on the quality of monocular depth estimation and optical flow. Misalignments may persist in areas where flow is inaccurate [S1].

**When to Use:**
Use when initializing 3D Gaussians from monocular depth estimates to ensure multi-view consistency, especially when MVS is prone to artifacts due to sparse input views [S1].

## Background
Coarse Alignment with Optical Flow was introduced in the CoherentGS paper (2024) as a solution to the multi-view inconsistency problem in 3D Gaussian Splatting initialization [S1]. It builds on the observation that monocular depth estimates, while high-quality, are relative and inconsistent across views, leading to misaligned Gaussians [S1]. The method contrasts with MVS-based initialization approaches like MVSNeRF, which can suffer from artifacts in sparse settings [S1].

## Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
[S2] Multi-Level Neural Scene Graphs for Dynamic Urban Environments, https://doi.org/10.1109/cvpr52733.2024.01996, fetched 2026-10-29
