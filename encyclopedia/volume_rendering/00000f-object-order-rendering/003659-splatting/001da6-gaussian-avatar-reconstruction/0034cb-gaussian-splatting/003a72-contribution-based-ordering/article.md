# Contribution-based ordering

Also called: Contribution score, Contribution-based progressive rendering, Euclidean distance from center, Gcontr, PRoGS Contribution Metric

## Overview
Contribution-based ordering is a post-training method for 3D Gaussian Splatting (3DGS) that sorts primitives by their visual importance to enable progressive rendering [S1]. It solves the problem of inefficient data streaming by ensuring high-impact splats are transmitted and rendered first, improving the Time to First Paint (TFP) [S1]. Unlike [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md) which defines the primitive shape, or [EAGLES](1386-gaussian-in-the-wild-gs-w.md) which uses similar metrics for pruning during training, this method operates on the fully trained model to select chunks for independent streaming [S1].

## How it works
**Inputs:** A trained set of $M$ 3D Gaussians $\{G_i\}_{i=1}^M$, each defined by position $\mathbf{x}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients for color $c_i$. A set of training viewpoints $V$.
**Outputs:** A sorted list of Gaussian IDs based on their contribution score, or a spatially refined ordering using an octree.

**Data Structures:**
1.  **Contribution Map:** A hash map or array indexed by Gaussian ID, storing the scalar contribution score $B_{id}$.
2.  **Octree (Optional):** A spatial subdivision structure (typically depth 3) used to refine global ordering [S1]. Each leaf node stores a local ranking of Gaussians within its bounds.

**Steps:**

1.  **Render Training Views:**
    For each viewpoint $v \in V$, render the scene using standard 3DGS volume splatting [Gaussian Splatting](946-gaussian-splatting.md).

2.  **Calculate Per-Pixel Contribution:**
    For each pixel $p$ in each view, identify the $N$ overlapping Gaussians. The color $C$ is computed as:
    $$ C = \sum_{i=1}^{N} T_i \alpha_i c_i $$
    where $T_i$ is the transmittance:
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    The contribution of the $i$-th Gaussian to pixel $p$ is defined as the weight factor:
    $$ w_{i,p} = T_i \alpha_i $$
    Store the top $K$ contributing Gaussians per pixel. The typical value for $K$ is 20 [S1].

3.  **Aggregate Global Contribution Score:**
    Sum the contributions of each Gaussian across all pixels $P$ and all views $V$:
    $$ B_{id} = \sum_{v \in V} \sum_{p \in P} (T_{id} \alpha_{id}) $$
    where $T_{id}$ and $\alpha_{id}$ are the transmittance and opacity of Gaussian $id$ at pixel $p$ in view $v$ [S1].

4.  **Refine with Octree (Optional):**
    To prevent bias towards densely captured areas, insert all Gaussians into an octree [S1].
    *   For each leaf node, sort the contained Gaussians by their local contribution.
    *   This ensures spatial distribution of high-contribution splats, improving foreground quality in sparsely captured regions [S1].

5.  **Alternative Metric: Euclidean Distance:**
    As a simpler baseline, order Gaussians by their Euclidean distance from the scene center $(0,0,0)$ [S1]:
    $$ G_{contr} = \sqrt{(x_2 - 0)^2 + (y_2 - 0)^2 + (z_2 - 0)^2} $$
    where $(x_2, y_2, z_2)$ is the Gaussian center [S1]. This is often used in web-based viewers (e.g., three.js) [S1].

**Pitfalls:**
*   **Size Bias:** Large splats covering multiple pixels inherently receive higher scores than small splats, as each pixel's contribution is capped at 1. Small splats must have high opacity or cover many pixels to compete [S1].
*   **Density Bias:** Without octree refinement, Gaussians in densely captured areas dominate the score, leading to poor reconstruction of sparse regions [S1].

## Subtopics

### [Contribution-based compression integration](8785-contribution-based-compression-integration.md)
Integrates this ordering into compression pipelines to stream high-contribution primitives first, prioritizing semantic importance over spatial/Morton order.

## Operating characteristics
*   **Complexity:** Requires a full forward pass over all training views to compute contributions, $O(M \cdot P \cdot V)$ where $P$ is pixels per view. Octree insertion is $O(M \log M)$ or linear with fixed depth [S1].
*   **Performance:** Significantly improves perceptual quality at low bitrates compared to distance-based or random ordering [S1].
*   **Strengths:** Maximizes visual fidelity for the first few chunks sent; effective for progressive web streaming [S1].
*   **Limitations:** Computationally expensive pre-processing step; large splats are favored, potentially delaying fine details [S1].

## Background
Originating in the PRoGS (Progressive Rendering of Gaussian Splats) paper [S1], this method was developed to address the inefficiency of transmitting full 3DGS models for real-time web viewing. It distinguishes itself from training-time pruning methods like EAGLES by operating post-training to optimize data transmission order [S1].

## Sources
[S1] PRoGS: Progressive Rendering of Gaussian Splats, https://doi.org/10.1109/wacv61041.2025.00308, fetched 2026-10-26
