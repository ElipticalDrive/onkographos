# TSDF-to-Opacity Pruning

Also called: TSDF Pruning, Opacity Thresholding

## Overview
TSDF-to-Opacity Pruning reduces the memory footprint of sparse voxel representations by removing voxels that contribute negligibly to the final rendered image. It solves the problem of excessive storage and traversal cost in dense Signed Distance Field (SDF) or TSDF grids by converting geometric distance values into volumetric opacity and discarding low-opacity regions. It differs from sibling methods like [Compact pointer tree](844-compact-pointer-tree.md) or [Implicit Octree Storage](939-implicit-octree-storage.md) by operating on the semantic content (distance-to-surface) rather than just structural sparsity or raw density values, allowing for geometry-aware compression [S1, unofficial].

## How it works
The method converts a TSDF grid into a sparse opacity representation through mapping and thresholding.

**Inputs:**
- A TSDF grid where each voxel stores a signed distance value $F$ (negative inside, positive outside).
- A surface thickness parameter $a$ (distance from surface to the inner boundary of the transition band) [S1, unofficial].
- A target opacity $p$ for the inner boundary [S1, unofficial].
- A transition sharpness parameter $\beta$ [S1, unofficial].
- A pruning threshold $T_{op}$ (opacity value below which voxels are removed) [S1, unofficial].
- Minimum and maximum opacity bounds $[min\_op, max\_op]$ for rescaling [S1, unofficial].

**Outputs:**
- A sparse set of voxels with opacity values in $[min\_op, max\_op]$.

**Data Structures:**
- The input is typically a structured grid or hash map of TSDF values.
- The output is a sparse structure (e.g., an octree or hash map) containing only voxels where $opacity \ge T_{op}$.

**Steps:**

1. **Calculate Transition Sharpness ($\beta$):**
   Determine $\beta$ such that a voxel at distance $F = -a$ (just inside the surface) maps to opacity $p$. Using the sigmoid mapping function $\phi_{sig}$:
   $$ \beta = -\frac{a}{\ln\left(\frac{1}{p} - 1\right)} $$
   This ensures a consistent "thickness" of the surface band across scenes [S1, unofficial].

2. **Map TSDF to Opacity:**
   For each voxel with signed distance $F$, compute the raw opacity $\alpha_{raw}$ using one of two mapping functions [S1, unofficial]:
   
   *Sigmoid Mapping ($\phi_{sig}$):*
   $$ \alpha_{raw} = \frac{1}{1 + e^{F/\beta}} $$
   
   *Bell Mapping ($\phi_{bell}$):*
   The source notes that $\phi_{bell}$ concentrates opacity more tightly than $\phi_{sig}$, producing a thinner band and sparser octree for the same threshold [S1, unofficial]. The exact formula for $\phi_{bell}$ is not provided in the source text, but it is defined as an alternative to $\phi_{sig}$ [S1, unofficial].

3. **Rescale Opacity:**
   Rescale $\alpha_{raw}$ to the target range $[min\_op, max\_op]$:
   $$ \alpha_{final} = min\_op + (\alpha_{raw} - \alpha_{min}) \frac{max\_op - min\_op}{\alpha_{max} - \alpha_{min}} $$
   where $\alpha_{min}$ and $\alpha_{max}$ are the minimum and maximum possible values of $\alpha_{raw}$ (typically 0 and 1 for sigmoid) [S1, unofficial].

4. **Prune Voxels:**
   Compare $\alpha_{final}$ against the predefined threshold $T_{op}$.
   - If $\alpha_{final} < T_{op}$, prune (remove) the voxel.
   - If $\alpha_{final} \ge T_{op}$, keep the voxel with opacity $\alpha_{final}$.

**Parameters:**
- $T_{op}$: The opacity threshold. The specific value is not given in the sources, but it directly controls the sparsity of the resulting octree [S1, unofficial].
- $a$: Surface thickness parameter.
- $p$: Prescribed opacity at $F=-a$.
- $\beta$: Transition sharpness.

**Edge Cases and Pitfalls:**
- **Precision:** The choice between $\phi_{sig}$ and $\phi_{bell}$ affects sparsity. $\phi_{bell}$ yields fewer above-threshold voxels (thinner band) [S1, unofficial].
- **Boundary Conditions:** Voxels far outside the surface ($F \gg 0$) will have opacity near 0 and be pruned. Voxels far inside ($F \ll -a$) will have opacity near 1 (or $max\_op$) and be kept, potentially filling the interior unless interior pruning is also applied (not specified in this method) [S1, unofficial].

## Subtopics

### Density Threshold Pruning
A variant that thresholds raw density values rather than TSDF-derived opacity. It is used for aggressive pruning in bounded scenes with solid surfaces. It uses a density threshold (e.g., $\alpha = 0.3$) and high grid resolution (e.g., $1024^3$) to significantly speed up rendering [S2]. However, it is less suitable for complex real-world scenes with reflective/transparent objects or unbounded scenes, as it can cause box-like artifacts in VR [S2].

## Operating characteristics
- **Time Complexity:** Pruning is $O(N)$ where $N$ is the number of voxels in the initial grid, as it requires a single pass to map and threshold [S1, unofficial].
- **Memory Complexity:** Reduces memory usage proportional to the sparsity achieved by the threshold. The resulting structure is an octree or sparse grid with fewer nodes [S1, unofficial].
- **Performance:** Aggressive pruning (using density thresholds) can approximately double FPS by rendering 1/4 as many pixels, though per-frame overhead may limit gains at very low resolutions [S2].
- **Strengths:** Geometry-aware compression; $\phi_{bell}$ provides tighter opacity concentration for sparser results [S1, unofficial].
- **Limitations:** Aggressive pruning fails on complex scenes (reflective, transparent, unbounded) causing artifacts [S2]. The exact threshold $T_{op}$ is not standardized and must be tuned per scene [S1, unofficial].

## Background
The method is described in the context of sparse-voxel surface reconstruction and VR-NeRF rendering. It is used to manage the trade-off between visual fidelity and rendering performance in neural radiance field (NeRF) and TSDF-based representations [S1, unofficial][S2].

## Sources
[S1] Advancing Structured Priors for Sparse-Voxel Surface Reconstruction, https://doi.org/10.48550/arxiv.2601.17720, fetched 2026-10-26 (unofficial)
[S2] VR-NeRF: High-Fidelity Virtualized Walkable Spaces, https://doi.org/10.1145/3610548.3618139, fetched 2026-10-28
