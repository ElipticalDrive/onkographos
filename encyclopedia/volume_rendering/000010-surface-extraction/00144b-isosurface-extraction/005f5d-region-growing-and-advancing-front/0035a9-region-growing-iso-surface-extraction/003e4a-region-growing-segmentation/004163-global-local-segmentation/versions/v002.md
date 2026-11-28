# Global/Local Segmentation

Also called: Global Segmentation via Mask, Mask-constrained Segmentation

## Overview
Global/Local Segmentation is a region-growing technique that mitigates errors from local decision-making by confining automatic segmentation to a user-specified mask [S1]. It sets itself apart from siblings like [Restricted Region Growing](9750-restricted-region-growing.md) by explicitly decoupling the spatial search domain (defined globally by the user) from the boundary optimization logic (which may use global criteria like dynamic programming) [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume $V$ and a binary mask volume $M$ of the same dimensions, where $M$ indicates the subset of $V$ for automatic segmentation [S1].
- Output: A segmented scene $D_p$ [S1].

**Data Structures:**
- The mask $M$ is typically a thin shell around the boundary of interest to minimize user interaction time [S1].
- The output volume $D_p$ stores boundary likelihood values for voxels in the mask shell $V_1$ and $0$ for voxels outside $V_1$ [S1].

**Steps:**
1. **User Interaction:** The user interactively indicates a subset of $V$ to serve as the mask $M$ [S1].
2. **Confinement:** Automatic segmentation is confined to this mask [S1].
3. **Boundary Optimization (Optional):** Within the mask, globally optimal boundaries are found using 2D dynamic programming [S1].

**Formulas:**
Let $V_1$ be the set of voxels in the mask shell [S1]. Let $B_i$ be the operator finding the optimal boundary within $V_1$ [S1]. The resulting scene $D_p$ is defined as:
$$
D_p(x,y,z) = 
\begin{cases} 
\text{boundary likelihood value} & \text{if } (x,y,z) \in V_1 \\
0 & \text{if } (x,y,z) \notin V_1 
\end{cases}
$$
[S1]

**Parameters:**
- Mask size: A shell around the boundary is often all that is needed [S1].

**Pitfalls:**
- Not given in the sources.

## Subtopics

### [Global Threshold Segmentation](22205-global-threshold-segmentation.md)
Global threshold segmentation is a technique that applies a single intensity threshold uniformly across all voxels or slices of a dataset to isolate a target structure. It differs from region-specific methods by using a constant cutoff value for the entire volume, avoiding the computational overhead and manual effort of per-slice adjustments, but failing to account for intra-structure density variations.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Considerable success reported in automatically finding globally optimal boundaries using 2D dynamic programming when the search is confined to a mask [S1]. User interaction time can be kept to a minimum [S1].
- **Limitations:** Relies on user-provided masks; automatic mask generation is noted as a useful but separate problem to pursue [S1].
- **When to use:** When local decision-making leads to errors and a rough spatial prior (mask) is available [S1].

## Background
Originated in the context of imaging transforms for visualizing surfaces and volumes [S1]. The approach addresses the difficulty of designing global criteria to ward off errors caused by local decision-making in scene segmentation [S1]. It is applied in various visualization applications where automatic boundary finding is required within a constrained region [S1].

## Sources
[S1] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
