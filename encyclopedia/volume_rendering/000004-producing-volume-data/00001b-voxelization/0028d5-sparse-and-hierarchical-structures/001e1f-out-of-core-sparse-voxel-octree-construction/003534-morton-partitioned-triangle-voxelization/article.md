# Morton-Partitioned Triangle Voxelization

Also called: Out-of-Core SVO Builder (adapted to in-core), SVO Builder

## Overview
Morton-Partitioned Triangle Voxelization is a method for constructing a Sparse Voxel Octree (SVO) from a triangle mesh by processing the geometry in streaming chunks defined by Morton order. It solves the memory bottleneck of global voxelization by partitioning the 3D grid into fixed-size subgrids, assigning triangles to these subgrids via axis-aligned bounding box (AABB) overlap, and marking voxels within each partition independently [S1, unofficial]. It differs from the Basic Version [1694-sparse-voxel-octree-construction-basic-version.md] by handling out-of-core constraints through explicit spatial partitioning, and from Grid-Free methods [9464-grid-free-out-of-core-voxelization-to-sparse-voxel-octrees.md] by relying on a dense intermediate voxel grid per partition rather than direct primitive-to-node mapping [S1, unofficial][S2].

## How it works
**Inputs:** Triangle mesh (list of triangles), memory limit for the voxel array ($m\_MemoryLimit$ in MB), target grid resolution ($gridSize$).
**Outputs:** A Sparse Voxel Octree (SVO) structure.

**Data Structures:**
*   **Morton:** A class handling Morton code encoding/decoding using bit-interleaving [S1, unofficial].
    *   `encodeMortonMagic(MortonCoords _m)`: Interleaves x, y, z coordinates into a 64-bit integer.
    *   `decodeMortonMagic()`: Returns `MortonCoords` struct containing x, y, z components.
*   **MortonCoords:** Struct with fields `m_x`, `m_y`, `m_z` (unsigned integers).
*   **Node:** Represents an SVO node.
    *   `m_Data`: Node data.
    *   `m_ChildrenBase`: Offset to children data.
    *   `m_ChildrenOffset[8]`: Offsets for 8 children, initialized to -1 (empty).
    *   `HasChild(i)`: Returns true if `m_ChildrenOffset[i] != -1`.
    *   `GetChildPos(i)`: Returns `m_ChildrenBase + m_ChildrenOffset[i]`.
*   **Cube:** A helper geometry for visualization, not part of the core algorithm logic.
*   **BBoxWrapper:** Helper for bounding box operations, including `clamp` [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing:**
    *   Load the mesh and compute its global bounding box $B_{global} = [minX, maxX] \times [minY, maxY] \times [minZ, maxZ]$ [S1, unofficial].
    *   **Cubify:** If the bounding box is not a cube, expand the smaller dimensions to match the largest dimension to prevent voxelization errors [S1, unofficial].
    *   **Translate:** Translate all vertices so that the minimum bounds of the cubified box are at the origin $(0,0,0)$ [S1, unofficial]. This prevents coordinate system mismatches and negative bound errors [S1, unofficial].

2.  **Partition Sizing:**
    *   Determine the maximum number of voxels that fit within $m\_MemoryLimit$.
    *   Calculate initial grid size $S$ such that $S^3 \times sizeof(char) \le m\_MemoryLimit \times 1024^2$ [S1, unofficial].
    *   If the memory exceeds the limit, divide $S$ by 2 (reducing volume by 8) and multiply the number of partitions by 8 until the condition is met [S1, unofficial].
    *   Let $gridSize$ be the final $S$. Note: $gridSize$ must be a power of 2 for Morton encoding compatibility [S1, unofficial].
    *   Calculate the total number of voxels in the grid: $V_{total} = gridSize^3$.
    *   Calculate the size of each partition in voxels: $morton\_partition = V_{total} / numPartitions$ [S1, unofficial].

3.  **Partition Boundaries:**
    *   Compute the physical unit length of a voxel: $unitLength = (maxX - minX) / gridSize$ [S1, unofficial].
    *   For each partition index $i$ from $0$ to $numPartitions - 1$:
        *   Calculate the Morton code range for the partition:
            *   $mcMin = m.decodeMortonMagic(morton\_partition \times i)$
            *   $mcMax = m.decodeMortonMagic(morton\_partition \times (i+1) - 1)$ [S1, unofficial].
        *   Compute the world-space bounding box for this partition:
            *   $minx = unitLength \times mcMin.m\_x$
            *   $maxx = unitLength \times (mcMax.m\_x + 1)$
            *   $miny = unitLength \times mcMin.m\_y$
            *   $maxy = unitLength \times (mcMax.m\_y + 1)$
            *   $minz = unitLength \times mcMin.m\_z$
            *   $maxz = unitLength \times (mcMax.m\_z + 1)$ [S1, unofficial].

4.  **Triangle Assignment and Voxelization (Per Partition):**
    *   For each partition, determine which triangles overlap with the partition's world-space bounding box using AABB intersection tests [S1, unofficial].
    *   If the partition contains no triangles, skip to the next partition.
    *   Allocate a voxel array `m_Voxels` of size $morton\_partition$ (initialized to empty/0) [S1, unofficial].
    *   Compute the inverse unit length: $M = 1.0f / unitLength$ [S1, unofficial].
    *   For each triangle $T$ assigned to this partition:
        *   Get the triangle's bounding box $B_T$ [S1, unofficial].
        *   Compute voxel-space bounds for the triangle:
            *   $LX_{min} = B_T.minX() \times M$
            *   $LX_{max} = B_T.maxX() \times M$
            *   $LY_{min} = B_T.minY() \times M$
            *   $LY_{max} = B_T.maxY() \times M$
            *   $LZ_{min} = B_T.minZ() \times M$
            *   $LZ_{max} = B_T.maxZ() \times M$ [S1, unofficial].
        *   Perform triangle-box intersection tests (Schwarz's method) to determine which voxels within the partition's bounds are covered by the triangle [S1, unofficial].
        *   Mark the corresponding voxels in `m_Voxels` as occupied.

5.  **SVO Construction:**
    *   After voxelizing all partitions, the occupied voxels are used to build the SVO nodes. The specific logic for converting the flat voxel array to the hierarchical `Node` structure (handling `m_ChildrenOffset`, `m_ChildrenBase`) is implied by the `Node` class methods (`Refine`, `AddEmptyVoxel`) but not fully detailed in the source pseudocode [S1, unofficial]. The source indicates that `m_DataPosition` and `m_NodePosition` track offsets for file output [S1, unofficial].

**Parameters:**
*   $gridSize$: Resolution of the voxel grid along one axis. Must be a power of 2 [S1, unofficial].
*   $m\_MemoryLimit$: Maximum memory in MB allowed for the voxel array before partitioning [S1, unofficial].

**Pitfalls:**
*   **Non-cube bounding boxes:** Cause voxelization errors if not cubified [S1, unofficial].
*   **Negative bounds/Coordinate mismatches:** Cause incorrect voxel mapping if vertices are not translated to the origin [S1, unofficial].

## Subtopics

### Morton-Partitioned Triangle Voxelization with Batch Determination
This variant structures and sorts triangles using Morton order properties to enable sequential batch processing and determine when voxels can be finalized for SVO creation [S2]. It calculates the bounding box for each triangle $T_i$ and computes the smallest Morton index $m_{e\_min}(T_i)$ of this bounding box as a lower bound for the minimal Morton index $m_{min}(T_i)$ of the triangle [S2]. Triangles are sorted by $m_{e\_min}(T_i)$. A batch $B_j = \{T_{j\_min}, \dots, T_{j\_max}\}$ is formed such that the sum of maximum voxels per triangle in the batch does not exceed a user-defined limit $K_{vox/batch}^{max}$:
$$ \sum_{i=j\_min}^{j\_max} N_{vox/tri}^{max}(T_i) \le K_{vox/batch}^{max} $$
If this condition is not fulfilled, the batch is finished and a new one is created [S2]. The index $m_{e\_min}(T_{j\_min})$ serves as a stop index, indicating the first voxel in the prior batch that cannot be used for node creation of the SVO, preventing Morton queue fragmentation and memory overflow [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and the number of voxels in the active partitions. Sorting triangles adds $O(N \log N)$ where $N$ is the number of triangles [S2].
*   **Memory Complexity:** Logarithmic in the size of the SVO, as it processes partitions sequentially and does not store the full global voxel grid in memory [S1, unofficial]. Memory usage is bounded by $m\_MemoryLimit$ plus the overhead of the SVO structure being built [S1, unofficial].
*   **Strengths:** Handles large meshes that do not fit in memory; Morton ordering ensures spatial locality which is efficient for cache and SVO construction [S1, unofficial][S2].
*   **Limitations:** Requires careful tuning of $gridSize$ and $m\_MemoryLimit$ [S1, unofficial]. Batch determination requires predicting voxel counts per triangle, which may be approximate [S2].

## Background
The method originates from work on out-of-core sparse voxel octree construction, specifically detailed in Giles Penfold's Master Thesis [S1, unofficial] and the paper "Grid-free out-of-core voxelization to sparse voxel octrees on GPU" [S2]. It builds on the concept of Morton order for spatial indexing [Morton 1966, cited in S2] and adapts it for streaming voxelization to avoid global memory allocation.

## Sources
[S1] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
[S2] Grid-free out-of-core voxelization to sparse voxel octrees on GPU, https://doi.org/10.1145/2790060.2790067, fetched 2026-11-05
