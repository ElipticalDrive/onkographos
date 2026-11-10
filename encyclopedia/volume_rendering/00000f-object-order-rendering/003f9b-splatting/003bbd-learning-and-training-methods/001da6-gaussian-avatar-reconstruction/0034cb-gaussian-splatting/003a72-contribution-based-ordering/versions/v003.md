# Contribution-based ordering

Also called: Contribution score, Contribution-based progressive rendering, Euclidean distance from center, Gcontr, PRoGS Contribution Metric

## Overview
Contribution-based ordering is a post-training method for 3D Gaussian Splatting (3DGS) that sorts primitives by their visual importance to enable progressive rendering [S1]. It solves the problem of inefficient data streaming by ensuring high-impact splats are transmitted and rendered first [S1]. Unlike [EAGLES](1386-gaussian-in-the-wild-gs-w.md) which uses this metric during training to prune splats, this method operates on the fully trained model to select impactful splats for independent streaming [S1].

## How it works
**Inputs:** A trained set of $M$ 3D Gaussians $\{G_i\}_{i=1}^M$, each defined by position $\mathbf{x}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients for color $c_i$ [S1]. A set of training viewpoints $V$ [S1].
**Outputs:** A sorted list of Gaussian IDs based on their contribution score, or a spatially refined ordering using an octree [S1].

**Data Structures:**
1.  **Contribution Map:** A hash map or array indexed by Gaussian ID, storing the scalar contribution score $B_{id}$ [S1].
2.  **Octree (Optional):** A spatial subdivision structure (typically depth 3) used to refine global ordering [S1]. Each leaf node stores a local ranking of Gaussians within its bounds [S1].

**Steps:**
1.  **Render Training Views:**
    For each viewpoint $v \in V$, render the scene using standard 3DGS volume splatting [Gaussian Splatting](946-gaussian-splatting.md) [S1].
2.  **Calculate Per-Pixel Contribution:**
    For each pixel $p$ in each view, identify the $N$ overlapping Gaussians [S1]. The color $C$ is computed as:
    $$ C = \sum_{i=1}^{N} T_i \alpha_i c_i $$
    where $T_i$ is the transmittance:
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    The contribution of the $i$-th Gaussian to pixel $p$ is defined as the weight factor:
    $$ w_{i,p} = T_i \alpha_i $$
    Store the top $K$ contributing Gaussians per pixel [S1].
3.  **Aggregate Global Contribution Score:**
    Sum the contributions of each Gaussian across all pixels $P$ and all views $V$:
    $$ B_{id} = \sum_{v \in V} \sum_{p \in P} (T_{id} \alpha_{id}) $$
    where $T_{id}$ and $\alpha_{id}$ are the transmittance and opacity of Gaussian $id$ at pixel $p$ in view $v$ [S1].
4.  **Refine with Octree (Optional):**
    To prevent bias towards densely captured areas, insert all Gaussians into an octree [S1].
    *   For each leaf node, sort the contained Gaussians by their local contribution [S1].
    *   This ensures spatial distribution of high-contribution splats, preventing sparsely captured regions from being unfairly under-reconstructed [S1].
5.  **Alternative Metric: Euclidean Distance:**
    An alternative metric used in some implementations (e.g., three.js) orders Gaussians by Euclidean distance from the scene center $(0,0,0)$ [S1]:
    $$ G_{contr} = \sqrt{(x_2 - x_1)^2 + (y_2 - y_1)^2 + (z_2 - z_1)^2} $$
    where $(x_1, y_1, z_1)$ is the camera position (or center) and $(x_2, y_2, z_2)$ is the Gaussian center [S1]. The center is taken as $(0,0,0)$ [S1].

**Parameters:**
*   $K$: The number of highest contributing Gaussians per pixel used to decide on the ordering [S1]. Typical value: 20 [S1].
*   Octree depth: Controls spatial subdivision granularity [S1]. Typical value: 3 [S1].

**Pitfalls:**
*   **Size Bias:** Large splats covering multiple pixels inherently receive higher scores than small splats, as each pixel's contribution is capped at 1 [S1]. Small splats must have high opacity or cover many pixels to compete [S1].
*   **Density Bias:** Without octree refinement, Gaussians in densely captured areas dominate the score, leading to poor reconstruction of sparse regions [S1].
*   **Frustum Culling Margin:** When enabling frustum culling, boundaries should be increased by 30% to avoid clipping visible splats at chunk boundaries [S1].

## Subtopics

### [Contribution-based compression integration](8785-contribution-based-compression-integration.md)
Integrates this ordering into compression pipelines to stream high-contribution primitives first, prioritizing semantic importance over spatial/Morton order.

## Operating characteristics
*   **Complexity:** Requires a full forward pass over all training views to compute contributions [S1].
*   **Performance:** Significantly improves perceptual quality at low bitrates compared to distance-based or random ordering [S1].
*   **Strengths:** Maximizes visual fidelity for the first few chunks sent; effective for progressive web streaming [S1].
*   **Limitations:** Computationally expensive pre-processing step; large splats are favored, potentially delaying fine details [S1].
*   **When to use:** Optimal for bandwidth-constrained progressive delivery where early perceptual quality is prioritized [S1].
*   **When not to use:** Unsuitable for training-time pruning or scenes where uniform spatial coverage is required without octree refinement [S1].

## Background
Originating in the PRoGS (Progressive Rendering of Gaussian Splats) paper [S1], this method was developed to address the inefficiency of transmitting full 3DGS models for real-time web viewing [S1]. It distinguishes itself from training-time pruning methods like EAGLES by operating post-training to optimize data transmission order [S1].

## Sources
[S1] PRoGS: Progressive Rendering of Gaussian Splats, https://doi.org/10.1109/wacv61041.2025.00308, fetched 2026-10-26
