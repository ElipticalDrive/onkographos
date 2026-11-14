# Floating-Point Optimal Scanline

Also called: Optimal Scanline, Zhang et al. Optimal Scanline

## Overview
The Floating-Point Optimal Scanline method is a triangle-based voxelization algorithm that fills the interior of a triangle by slicing it along a dominant axis and performing 2D line voxelizations between the sliced edges [S1, unofficial]. It distinguishes itself from sibling methods like [3D Scan-conversion](1895-3d-scan-conversion.md) by using an "optimal" distance criterion to determine which voxels are intersected by the scanlines, ensuring a tighter fit to the geometric surface than standard grid-aligned line algorithms [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A single triangle defined by three vertices $V_0, V_1, V_2$ in 3D space and the triangle's normal vector $\mathbf{n} = (n_x, n_y, n_z)$.
*   **Output:** A set of voxel coordinates $(x, y, z)$ that intersect the triangle.

**Data Structures:**
*   **Vertices:** Array of 3 vectors, each containing 3 floating-point coordinates.
*   **Normal:** Vector of 3 floating-point components.
*   **Edge Slices:** List of pairs of 2D points (or 3D points with fixed dominant coordinate) representing the intersection of the triangle edges with integer planes of the dominant axis.

**Steps:**

1.  **Determine Dominant Axis:**
    Calculate the absolute values of the normal's components: $|n_x|, |n_y|, |n_z|$.
    Identify the axis corresponding to the maximum value.
    *   If $|n_x|$ is max, dominant axis is X.
    *   If $|n_y|$ is max, dominant axis is Y.
    *   If $|n_z|$ is max, dominant axis is Z.
    *   *Note:* The algorithm description assumes the dominant axis is Z for the subsequent steps. If another axis is dominant, the coordinate roles ($x, y, z$) must be permuted accordingly [S1, unofficial].

2.  **Sort Vertices:**
    Sort the triangle vertices $V_0, V_1, V_2$ based on their coordinates along the dominant axis. Let the sorted vertices be $V_{min}, V_{mid}, V_{max}$ such that their dominant-axis coordinates satisfy $z_{min} \le z_{mid} \le z_{max}$ (assuming Z is dominant) [S1, unofficial].

3.  **Voxelize Edges (RLV):**
    Perform line voxelization between each pair of sorted vertices $(V_{min}, V_{mid})$ and $(V_{mid}, V_{max})$ using a Ray Casting Line Voxelization (RLV) algorithm. This produces the voxelized edges of the triangle [S1, unofficial].
    *   *Note:* The specific RLV implementation is not detailed in the source, but it serves to define the boundary voxels [S1, unofficial].

4.  **Slice Edges:**
    Split the voxelized edges into slices along the dominant axis. A slice is defined by an integer value $k$ of the dominant axis. For each integer $k$ between the minimum and maximum dominant-axis coordinates of the triangle, identify the two points on the triangle edges that lie on the plane $z=k$ (or $x=k$ or $y=k$). Let these points be $P_{left}(k)$ and $P_{right}(k)$ in the plane perpendicular to the dominant axis [S1, unofficial].

5.  **Calculate Optimal Distance:**
    For each slice $k$, calculate the optimal distance $l$ using the formula:
    $$ l = |dsl_x| + |dsl_y| $$
    where $dsl$ is the voxel's diagonal projected onto the scanline direction within the 2D plane of the slice. Specifically, if the scanline connects $P_{left}$ and $P_{right}$, the direction vector is $\mathbf{d} = P_{right} - P_{left}$. The term $dsl$ represents the projection of the unit voxel diagonal onto this direction. The source defines $l$ as the length of the voxel's diagonal projected onto the scanline direction [S1, unofficial].

6.  **Perform 2D Optimal Scanline Voxelization:**
    For each slice $k$, determine the voxels intersected by the line segment between $P_{left}(k)$ and $P_{right}(k)$.
    *   Use the optimal distance $l$ to decide voxel inclusion. The theorem states that if the distance between two parallel lines is less than $l$, no voxel can exist between them without being intersected.
    *   The algorithm classifies the scanline into four cases based on the signs of the components of $dsl$ to determine the exact set of voxels [S1, unofficial].
    *   *Implementation Note:* The source does not provide the explicit pseudocode for the 4-case classification. It implies that standard 2D line rasterization is modified or filtered by this optimal distance criterion to exclude voxels that do not truly intersect the line within the optimal tolerance [S1, unofficial].

**Parameters:**
*   **Voxel Size:** Implicitly 1.0 in the normalized coordinate system where the diagonal projection is calculated.
*   **Dominant Axis:** Determined dynamically per triangle.

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** If the triangle is degenerate (area zero), the normal is undefined. The source does not specify handling for this case [S1, unofficial].
*   **Precision:** The algorithm relies on floating-point arithmetic for sorting and interpolation. Precision errors in determining the dominant axis or vertex sorting may lead to incorrect edge voxelization [S1, unofficial].
*   **Boundary Conditions:** The slicing step requires careful handling of vertices that lie exactly on integer planes to avoid double-counting or missing voxels [S1, unofficial].

## Subtopics

### Polygon Rasterization (2D)
A 2D scan-line algorithm that rasterizes a polygon into a P-buffer by decomposing it into triangles and flipping presence bits for covered positions [S2, unofficial]. It sets $x_l=x_r=C.x$ and $y=C.y$ (where C is the vertex with the least y), initializes slopes $i_l, i_r$ for segments CA and CB, and increments y while updating $x_l, x_r$ [S2, unofficial]. It constructs triangles between the centroid and polygon edges, rasterizing them in the P-buffer by flipping bits, then transfers positions with presence value 1 to the framebuffer [S2, unofficial]. This method is distinct from the 3D Floating-Point Optimal Scanline as it operates in 2D and uses a bit-flipping P-buffer approach rather than optimal distance-based voxel inclusion [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels intersected by the triangle, plus the cost of sorting vertices and slicing edges. The dominant factor is the 2D line voxelization for each slice [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of voxels in the triangle's bounding box. The algorithm processes one triangle at a time, so it does not require global storage for the triangle itself [S1, unofficial].
*   **Strengths:** Provides a tighter voxelization than standard grid-aligned line algorithms by using the optimal distance criterion, reducing artifacts [S1, unofficial].
*   **Limitations:** Requires calculation of the triangle normal and sorting of vertices. The specific implementation of the 4-case optimal distance check is not detailed, potentially making implementation complex [S1, unofficial].
*   **When to use:** When high-quality triangle voxelization is required and the computational cost of determining the optimal voxel set is justified [S1, unofficial].

## Background
The Floating-Point Optimal Scanline algorithm was proposed by Zhang et al. [S1, unofficial]. It is part of a family of optimal scanline voxelization techniques. The method builds on the concept of line voxelization and extends it to 2D slices within a 3D triangle [S1, unofficial]. The theoretical basis involves a theorem regarding the maximum distance between parallel lines such that no voxel can exist between them without intersection [S1, unofficial].

## Sources
[S1] A Comparison of Optimal Scanline Voxelization Algorithms, https://liu.diva-portal.org/smash/get/diva2:1459330/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S2] Voxelization of solids using simplicial coverings, http://wscg.zcu.cz/wscg2004/Papers_2004_Short/J53.pdf, fetched 2026-10-26 (unofficial)
