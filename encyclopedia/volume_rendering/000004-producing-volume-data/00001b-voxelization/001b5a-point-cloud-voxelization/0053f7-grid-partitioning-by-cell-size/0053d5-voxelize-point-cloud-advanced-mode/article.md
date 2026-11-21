# Voxelize Point Cloud (Advanced Mode)
Also called: Advanced partitioning mode
## Overview
Voxelize Point Cloud (Advanced Mode) is a voxelization method that maps a 3D point cloud to a regular grid using a user-defined cell length (spacing) parameter. It differs from [Voxelize Point Cloud (Basic Mode)](15079-voxelize-point-cloud-basic-mode.md) by explicitly accepting a spacing parameter rather than deriving it from geometry, and from [Voxelize Point Cloud (Existing Partition Grid Mode)](15084-voxelize-point-cloud-existing-partition-grid-mode.md) by not requiring an existing image geometry as a reference. Unlike the Existing Partition Grid path, which rejects zero spacing with an error, Advanced Mode allows zero spacing, resulting in a silent failure where all points are skipped.

## How it works
**Inputs:**
*   Point cloud geometry (e.g., `VertexGeom`).
*   Cell length parameter $s$ (scalar).
*   Grid origin $o$ (optional, defaults may apply).
*   Grid dimensions $D_x, D_y, D_z$ (optional, may be derived or fixed).

**Outputs:**
*   A binary mask or data array associated with the grid geometry. The tuple shape is typically $\{D_z, D_y, D_x\}$ [S1, unofficial].

**Data Structures:**
*   The output is an image geometry with dimensions $D_x, D_y, D_z$.
*   The mask is a 1D array of tuples with shape $\{D_z, D_y, D_x\}$ [S1, unofficial].

**Steps:**
1.  **Validate Input:** Check if the cell length $s$ is negative. If $s < 0$, reject with an error [S1, unofficial].
2.  **Calculate Spacing:** Set spacing $s_{calc} = s$. If $s = 0$, $s_{calc} = 0$ [S1, unofficial].
3.  **Compute Inverse Spacing:** Calculate $s_{inv} = 1.0 / s_{calc}$ [S1, unofficial].
    *   If $s_{calc} = 0$, $s_{inv}$ becomes $\infty$ or `NaN` [S1, unofficial].
4.  **Map Points to Voxels:** For each point $p = (p_x, p_y, p_z)$:
    *   Calculate voxel coordinates:
        $$ v_x = (p_x - o_x) \times s_{inv} $$
        $$ v_y = (p_y - o_y) \times s_{inv} $$
        $$ v_z = (p_z - o_z) \times s_{inv} $$
    *   If $s_{calc} = 0$, $v_i$ becomes $\infty$ or `NaN` [S1, unofficial].
5.  **Index and Clamp:** Convert $v_i$ to integer indices $(v_x, v_y, v_z)$ and clamp to grid bounds $[0, D_i-1]$ [S1, unofficial].
    *   If $v_i$ is $\infty$ or `NaN`, the point is skipped [S1, unofficial].
6.  **Set Mask:** Set the corresponding voxel in the output mask to 1 (or relevant data) [S1, unofficial].
7.  **Finalize:** If all points were skipped (e.g., due to $s=0$), the output mask is all zeros. A warning is issued, but the filter completes successfully [S1, unofficial].

**Pitfalls:**
*   **Zero Spacing:** If $s=0$, the division $1.0/0$ results in infinity/NaN, causing all points to be skipped. The filter produces an all-zero mask and only a warning, unlike the Existing Partition Grid path which errors out [S1, unofficial].
*   **Negative Spacing:** Rejected by `DataCheckCellLength` [S1, unofficial].
*   **Origin/Spacing Mismatch:** A sign error on origin subtraction or swapped inverse spacing components can lead to incorrect voxel mapping [S1, unofficial].

## Subtopics

### Voxelize Point Cloud (Bounding Box Mode)
A voxelization mode that accepts min/max bounding box coordinates to determine grid extent. If min equals max on any axis, the spacing becomes zero, causing all points to be skipped. Unlike the Advanced Mode which uses a fixed cell length, this mode derives spacing from the bounding box. If min == max, `DataCheckBoundingBoxCoords` does not reject it (only min > max is rejected), leading to $s=0$, $s_{inv}=\infty$, and an all-zero mask with only a warning [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of points $N$, $O(N)$, as each point is mapped once.
*   **Memory Complexity:** Proportional to the grid volume $D_x \times D_y \times D_z$ for the output mask.
*   **Strengths:** Explicit control over voxel size via cell length parameter.
*   **Limitations:** Silent failure on zero spacing; no automatic grid dimension derivation if not provided.
*   **When to Use:** When a specific voxel resolution is required independent of the input geometry's extent.
*   **When Not to Use:** When zero spacing is possible and silent failure is unacceptable; use Existing Partition Grid Mode for stricter validation.

## Background
This mode is part of the Voxelize Point Cloud filter family in the simplnx library. It was introduced to provide explicit control over voxelization parameters. The behavior regarding zero spacing was noted in code review discussions, highlighting a discrepancy with the Existing Partition Grid path [S1, unofficial].

## Sources
[S1] FILT: Voxelize Point Cloud, https://github.com/BlueQuartzSoftware/simplnx/pull/1724, fetched 2026-10-27 (unofficial)
