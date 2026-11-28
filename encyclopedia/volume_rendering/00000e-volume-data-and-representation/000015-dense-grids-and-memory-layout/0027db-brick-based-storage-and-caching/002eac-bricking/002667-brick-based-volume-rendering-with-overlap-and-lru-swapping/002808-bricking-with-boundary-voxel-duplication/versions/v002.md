# Bricking with Boundary Voxel Duplication

Also called: Boundary layer correction, Clipping with pre-integrated volume rendering, Texture Bricking, Volume Boundary Replication

## Overview
Bricking with Boundary Voxel Duplication is a technique used in brick-based volume rendering to prevent interpolation artifacts and data loss at the boundaries between adjacent 3D texture bricks. It solves the problem where standard texture sampling fails to correctly interpolate values across shared brick edges, resulting in visual discontinuities or "black strips" in the composited image. Unlike sibling methods that may rely on complex multi-level interpolation logic or 2D texture tiling, this method ensures geometric consistency by explicitly duplicating a plane of voxels at each brick boundary, allowing hardware texture units to perform correct trilinear interpolation across the shared plane.

## How it works
**Inputs and Outputs:**
*   **Input:** A dense 3D scalar field (volume data) and a partitioning scheme dividing the volume into $N$ subvolumes (bricks).
*   **Output:** A set of 3D texture objects, one for each brick, ready for GPU upload.

**Data Structures:**
Each brick is stored as a 3D texture array or a 3D texture object. The layout differs from the original volume grid to accommodate overlap.
*   Let the original volume have dimensions $W \times H \times D$.
*   Let the bricks be partitioned along one or more axes. Consider a partition along the $z$-axis for simplicity.
*   Original layers: $0$ to $D-1$.
*   Partition point: Layer $k$ is the boundary between Brick $i$ and Brick $i+1$.
*   **Brick $i$ Layout:** Contains layers $0$ to $k$. The size is $W \times H \times (k+1)$.
*   **Brick $i+1$ Layout:** Contains layers $k$ to $D-1$. Crucially, layer $k$ is **duplicated**. The size is $W \times H \times (D-k+1)$.
*   The voxel at index $k$ in Brick $i$ and the voxel at index $0$ (relative to the brick's start) in Brick $i+1$ hold identical data.

**Numbered Steps:**
1.  **Partition the Volume:** Divide the input volume grid into subvolumes (bricks) according to the memory management strategy (e.g., LRU swapping). Identify the boundary planes where bricks meet.
2.  **Identify Boundary Voxels:** For each brick, determine which faces are internal (shared with another brick) and which are external (volume boundary).
3.  **Duplicate Boundary Voxels:**
    *   For every internal boundary plane shared between two adjacent bricks, replicate the plane of voxels into both neighboring bricks.
    *   Specifically, if Brick A ends at layer $k$ and Brick B starts at layer $k$, Brick A includes layer $k$, and Brick B also includes a copy of layer $k$ as its first layer.
    *   This ensures that the texture coordinate $1.0$ (or $0.0$ depending on normalization) on one brick maps to the same physical location as $0.0$ (or $1.0$) on the neighbor.
4.  **Construct Textures:** Upload the modified brick data to GPU memory as 3D textures.
5.  **Render:** During ray casting, when a ray crosses a brick boundary, the texture unit samples the new brick. Because the boundary voxel is duplicated, trilinear interpolation across the brick edge yields the same result as if the entire volume were a single continuous texture.

**Formulas:**
Let $V(x, y, z)$ be the original volume.
Let $B_i$ be the texture data for brick $i$.
If brick $i$ covers $z \in [z_{start}, z_{end}]$ and brick $j$ covers $z \in [z_{end}, z_{end}']$, then:
$$ B_i(x, y, z_{end}) = V(x, y, z_{end}) $$
$$ B_j(x, y, z_{start}) = V(x, y, z_{end}) $$
Where $z_{start}$ of brick $j$ is the duplicated layer.

**Parameters:**
*   **Overlap Depth:** Typically 1 voxel plane. [S1, S2, unofficial]
*   **Interpolation Mode:** Trilinear interpolation must be enabled on the GPU texture sampler. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Black Strips:** If boundary replication is not used and the view vector is parallel to the cut plane, a black strip will appear at each cut boundary in the composited image because ray samples in the cut boundary region are lost. [S2, unofficial]
*   **Interpolation Artifacts:** Without duplication, the texture unit lacks information to consistently interpolate across texture boundaries, leading to incorrect values at the seam. [S1, unofficial]
*   **Memory Overhead:** Duplicating boundary planes increases the memory footprint of each brick. For large datasets, this overhead, combined with the inefficiency of 3D-texture based approaches, can significantly lower frame rates compared to 2D-texture solutions. [S1, unofficial]

## Subtopics

### [Continuous Level Transitions](5275-continuous-level-transitions.md)
Handles LOD transitions where adjacent bricks have different resolutions. It restricts LOD differences to at most one level to ensure valid texture sampling without complex multi-level interpolation.

## Operating characteristics
*   **Time Complexity:** No additional computational cost per ray sample compared to standard volume rendering, as the duplication is a preprocessing step. [S1, unofficial]
*   **Memory Complexity:** Increases memory usage by approximately $2 \times$ the area of the boundary planes per brick. For large volume datasets, this contributes to the inefficiency of the 3D-texture based approach. [S1, unofficial]
*   **Performance:** While superior in image quality to 2D-texture based solutions by removing significant drawbacks, the 3D-texture based approach (including bricking) is extremely inefficient for large volume data sets, resulting in significantly lower frame rates. [S1, unofficial]
*   **Strengths:** Eliminates visual artifacts (black strips, seams) at brick boundaries; enables correct hardware trilinear interpolation across brick edges. [S1, S2, unofficial]
*   **Limitations:** Increased memory usage; lower frame rates for large datasets compared to 2D alternatives. [S1, unofficial]

## Background
The technique originated in the context of parallel ray-traced volume rendering and brick-based rendering on general-purpose graphics hardware. It was developed to address the limitations of simple texture tiling, where hardware interpolation fails at tile boundaries. The method is discussed in early works on data-distributed parallel algorithms for volume rendering and later in dissertations on general-purpose graphics hardware techniques. [S1, S2, unofficial]

## Sources
[S1] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA270501: A Data Distributed Parallel Algorithm for Ray-Traced Volume Rendering, https://archive.org/details/DTIC_ADA270501, fetched 2026-10-26 (unofficial)
