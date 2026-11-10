# Uniform Grid Traversal

Also called: Uniform Grid Acceleration

## Overview
Uniform Grid Traversal accelerates ray traversal through uniformly distributed particles by partitioning the scene into a regular 3D grid [S1]. It differs from its siblings by relying on spatial locality within fixed-size voxels rather than hierarchical structures or occupancy masks. It is chosen when particle distribution is uniform and the grid resolution can be tuned to balance traversal overhead against per-voxel processing cost [S1].

## How it works
**Inputs and Outputs:**
- Input: A set of $N$ particles uniformly distributed in a bounding box, and a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Output: A list of particles intersected by the ray, ordered by distance.

**Data Structures:**
- **Uniform Grid:** A 3D array of size $Res \times Res \times Res$. Each voxel contains a list of particle indices.
- **Particle List:** Stored in each voxel.

**Steps:**
1. **Grid Construction:**
   - Determine grid resolution $Res$. The optimal $Res$ yields roughly one particle per voxel [S1].
   - For each particle, compute its voxel coordinates $(i, j, k)$ based on its position and the grid bounds.
   - Insert the particle index into the list at `Grid[i][j][k]` [S1].

2. **Ray Traversal:**
   - Compute the entry and exit points of the ray with the grid bounding box.
   - Initialize step sizes $\Delta x, \Delta y, \Delta z$ based on the inverse of the ray direction components.
   - Determine the initial voxel $(i, j, k)$ and the next boundary crossings $t_{max\_x}, t_{max\_y}, t_{max\_z}$ [S1].
   - While the ray is within the grid:
     - Retrieve the particle list from `Grid[i][j][k]`.
     - For each particle in the list, test for intersection with the ray.
     - If intersected, record the intersection parameter $t$.
     - Advance to the next voxel by comparing $t_{max\_x}, t_{max\_y}, t_{max\_z}$ and updating $(i, j, k)$ and the corresponding $t_{max}$ value [S1].

**Parameters:**
- **Grid Resolution ($Res$):** No analytical formula exists for the optimal resolution [S1]. Empirically, $Res$ should be chosen such that the average number of particles per voxel is approximately 1 [S1].
  - Example: For a scene with $2 \times 10^5$ particles, $Res = 60$ ($60 \times 60 \times 60$) is optimal [S1].

**Edge Cases and Pitfalls:**
- **Low Resolution:** If $Res$ is too low (e.g., $< 60 \times 60 \times 60$ for the Bench scene with $2 \times 10^5$ particles), performance degrades significantly because too many particles reside in each voxel, increasing per-voxel processing time [S1].
- **High Resolution:** As $Res$ increases, runtime grows steadily due to increased grid traversal overhead [S1].

## Subtopics

### [Computational Space Traversal](10339-computational-space-traversal.md)
Traversing a curvilinear grid in computational space (which is regular) to simplify resampling, using Jacobian matrices for coordinate transformation. This variant allows the use of uniform grid traversal techniques on non-uniform physical grids by mapping the problem to a regular computational domain [10339-computational-space-traversal.md].

## Operating characteristics
- **Time Complexity:** Depends on grid resolution and particle density. Runtime grows steadily as grid resolution increases due to additional grid traversal costs [S1].
- **Memory Complexity:** $O(Res^3 + N)$, where $Res$ is the grid resolution and $N$ is the number of particles.
- **Performance:** Optimal performance is achieved when the grid resolution yields roughly one particle per voxel [S1]. For the Bench scene with $2 \times 10^5$ particles, $60 \times 60 \times 60$ resolution provides the lowest rendering time [S1].
- **Strengths:** Simple implementation; efficient for uniformly distributed particles [S1].
- **Limitations:** Performance degrades if particle distribution is not uniform or if resolution is poorly chosen [S1]. No analytical method to determine optimal resolution [S1].

## Background
Uniform Grid Traversal is a standard acceleration technique in volume rendering and ray tracing for discrete participating media. It is particularly effective when particles are uniformly distributed in the scene [S1]. The method is widely used in rendering systems that employ geometrical optics approximations for discrete media [S1].

## Sources
[S1] Rendering discrete participating media using geometrical optics approximation, https://doi.org/10.1007/s41095-021-0253-5, fetched 2026-10-26
[S2] Compact, Fast and Robust Grids for Ray Tracing, https://doi.org/10.1111/j.1467-8659.2008.01262.x, fetched 2026-10-24
