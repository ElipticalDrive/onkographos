# Regular Tracking

Also called: Semi-Analytic Free-path Sampling, Surface Tracking

## Overview
Regular Tracking is a semi-analytic method for sampling free paths in piecewise-simple media, such as piecewise-constant volumes, by replacing integration with summation over a structured grid [S1, unofficial]. It differs from analytic closed-form tracking by handling discontinuous extinction coefficients and from ray marching by exactly resolving boundary crossings rather than approximating them with fixed steps [S1, unofficial].

## How it works
**Inputs:** A ray origin $\mathbf{o}$ and direction $\mathbf{d}$; a piecewise homogeneous medium defined by a hierarchical voxel grid or octree where each cell has a constant extinction coefficient $\mu_t$; a uniform random number $\xi \in (0, 1]$.
**Outputs:** The distance $t$ to the next collision point.

**Data Structures:**
The medium is represented as a hierarchical grid (e.g., voxel grid or octree) [S1, unofficial][S2]. Each leaf node or cell stores the constant extinction coefficient $\mu_t$ for that region. The structure allows for efficient traversal to identify boundaries along the ray [S2].

**Steps:**
1.  **Initialize:** Set current ray parameter $t_{curr} = 0$ and accumulated optical thickness $\tau_{acc} = 0$.
2.  **Sample Transmittance:** Compute the target transmittance value $T_{target} = \xi$. The goal is to find $t$ such that $e^{-\tau(0, t)} = T_{target}$, or equivalently $\tau(0, t) = -\ln(T_{target})$ [S2].
3.  **Traverse Boundaries:** Identify the sequence of boundaries the ray crosses along its path. This is done by stepping through the grid structure (e.g., using Amanatides-Woo for grids or Hubert-Tremblay et al. for octrees) [S2].
4.  **Accumulate Optical Thickness:** For each segment between consecutive boundaries (or from origin to first boundary), calculate the optical thickness contribution: $\Delta \tau = \mu_t \cdot \Delta t$, where $\mu_t$ is the extinction coefficient of the current cell and $\Delta t$ is the distance to the next boundary [S2].
5.  **Update Accumulator:** Add $\Delta \tau$ to $\tau_{acc}$.
6.  **Check Collision:** If $\tau_{acc} \ge -\ln(T_{target})$, the collision occurs within the current segment. The exact distance $t$ is found by solving for the remaining optical thickness required in the current cell:
    $$ \mu_t \cdot (t - t_{boundary\_start}) = -\ln(T_{target}) - (\tau_{acc} - \Delta \tau) $$
    $$ t = t_{boundary\_start} + \frac{-\ln(T_{target}) - (\tau_{acc} - \Delta \tau)}{\mu_t} $$
    Return $t$ [S2].
7.  **Continue:** If $\tau_{acc} < -\ln(T_{target})$, update $t_{curr}$ to the next boundary distance and repeat from step 4.

**Edge Cases and Pitfalls:**
*   **Zero Extinction:** If $\mu_t = 0$ in a cell, the optical thickness does not increase. The algorithm must handle this to avoid division by zero in step 6 if the collision falls in a vacuum region (though physically, a collision cannot occur in a vacuum, so the search continues) [G].
*   **Boundary Precision:** Numerical errors in boundary intersection calculations can lead to missing or double-counting boundaries. Robust geometric predicates are required [G].
*   **Performance:** The method becomes expensive if the ray crosses many boundaries, as each boundary requires a grid traversal step [S2].

## Subtopics

### Ray Marching
An approximate method that ignores boundaries and uses fixed steps. It is simpler but introduces bias, especially for thin features [S1, unofficial].

### Analytic Tracking
Uses closed-form inverse CDFs for simple extinction functions (constant, exponential). It is faster than regular tracking but limited to specific functional forms [S1, unofficial].

## Operating characteristics
**Time Complexity:** Linear in the number of boundaries crossed by the ray. Grid traversal is $O(k)$ where $k$ is the number of cells intersected [S2].
**Memory Complexity:** Depends on the grid resolution. Hierarchical structures (octrees) reduce memory usage for sparse volumes [S2].
**Strengths:** Exact sampling for piecewise-constant media; no bias from discretization [S1, unofficial].
**Limitations:** Slow when rays cross many boundaries; requires efficient grid traversal structures [S2].
**When to Use:** When the medium is piecewise-constant and represented in a grid, and exact sampling is required [S1, unofficial].
**When Not to Use:** For general heterogeneous volumes without a grid structure, or when performance is critical and approximate results are acceptable [S1, unofficial][S2].

## Background
Regular Tracking, also known as surface tracking, was introduced by Sutton et al. in 1999 [S2]. It is based on the idea of visiting boundaries in piecewise homogeneous regions to integrate optical thickness [S2]. The method relies on efficient boundary traversal algorithms, such as those by Amanatides and Woo (1987) for grids and Hubert-Tremblay et al. (2006) for octrees [S2].

## Sources
[S1] Participating media, https://imaging.cs.cmu.edu/15-468/2024_spring/lectures/lecture_12.pdf, fetched 2026-10-29 (unofficial)
[S2] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
