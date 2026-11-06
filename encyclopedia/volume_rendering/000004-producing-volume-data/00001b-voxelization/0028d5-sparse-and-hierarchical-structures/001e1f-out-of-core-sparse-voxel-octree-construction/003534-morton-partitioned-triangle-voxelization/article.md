# Morton-Partitioned Triangle Voxelization

Also called: Out-of-Core SVO Builder (adapted to in-core), SVO Builder

## Overview
Morton-Partitioned Triangle Voxelization is a streaming voxelization pipeline that constructs a Sparse Voxel Octree (SVO) from a triangle mesh by processing a high-resolution 3D grid in Morton order [S1, unofficial]. It partitions the voxel grid into subgrids, assigns triangles to subgrids via axis-aligned bounding box (AABB) overlap, and marks voxels using triangle-box intersection tests [S1, unofficial]. It sets itself apart from [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md) by managing memory usage logarithmically through out-of-core partitioning, allowing the construction of large SVOs that exceed available RAM [S1, unofficial].

## How it works
**Inputs:** Triangle mesh (vertices and indices), maximum memory limit in MB ($m_{MemoryLimit}$), target voxel grid resolution along one axis ($gridSize$, must be a power of 2) [S1, unofficial].
**Outputs:** A Sparse Voxel Octree (SVO) stored on disk or in memory [S1, unofficial].

**Data Structures:**
*   **Morton:** Encodes/decodes 3D coordinates $(x,y,z)$ to/from a 64-bit integer using bit-interleaving [S1, unofficial].
    *   `encodeMortonMagic(MortonCoords m)`: Interleaves bits of $m.x, m.y, m.z$ into a 64-bit code.
    *   `decodeMortonMagic()`: Extracts $m.x, m.y, m.z$ from the 64-bit code.
*   **MortonCoords:** Struct containing `m_x`, `m_y`, `m_z` (unsigned integers) [S1, unofficial].
*   **Node:** Represents an octree node.
    *   `m_Data`: Node data (e.g., density) [S1, unofficial].
    *   `m_ChildrenBase`: Offset to children array [S1, unofficial].
    *   `m_ChildrenOffset[8]`: Offsets to 8 children; `-1` indicates no child [S1, unofficial].
    *   `HasChild(i)`: Returns true if `m_ChildrenOffset[i] != -1` [S1, unofficial].
    *   `GetChildPos(i)`: Returns `m_ChildrenBase + m_ChildrenOffset[i]` if child exists, else 0 [S1, unofficial].
*   **Octree Class Fields:** `m_Children` (vector of vectors of Nodes), `m_MaxDepth`, `m_CurrentMorton`, `m_MaxMorton`, `m_DataPosition`, `m_NodePosition`, `m_GridSize`, `m_Compression`, `m_File`, `m_FileName`, `_cube`, `m_Nodes`, `m_outStream`, `m_inStream`, `m_Data`, `m_DebugData`, `counter` [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing:** Load the mesh. Compute its bounding box $B_{mesh}$. Cubify $B_{mesh}$ to ensure equal dimensions along all axes. Translate all mesh vertices so that the minimum bounds of the cubified box are at the origin $(0,0,0)$ [S1, unofficial]. This prevents voxelization errors caused by non-cube bounding boxes or negative coordinates [S1, unofficial].
2.  **Partition Sizing:** Calculate the number of partitions ($numPartitions$) based on $gridSize$ and $m_{MemoryLimit}$.
    *   Initial memory estimate: $size = (gridSize^3 \times sizeof(char)) / 1024^2$ MB [S1, unofficial].
    *   While $size > m_{MemoryLimit}$, divide $size$ by 8 and multiply $numPartitions$ by 8 [S1, unofficial].
    *   Compute unit length: $unitLength = (maxX - minX) / gridSize$ [S1, unofficial].
    *   Compute voxels per partition: $morton\_partition = gridSize^3 / numPartitions$ [S1, unofficial].
3.  **Partition Iteration:** For each partition index $i$ from $0$ to $numPartitions - 1$:
    *   **Compute World Bounds:**
        *   Decode Morton codes for the start and end of the partition's voxel range:
            $mcMin = m.decodeMortonMagic(morton\_partition \times i)$
            $mcMax = m.decodeMortonMagic(morton\_partition \times (i+1) - 1)$ [S1, unofficial].
        *   Compute world-space bounding box for the partition:
            $minx = unitLength \times mcMin.m_x$
            $maxx = unitLength \times (mcMax.m_x + 1)$
            $miny = unitLength \times mcMin.m_y$
            $maxy = unitLength \times (mcMax.m_y + 1)$
            $minz = unitLength \times mcMin.m_z$
            $maxz = unitLength \times (mcMax.m_z + 1)$ [S1, unofficial].
    *   **Triangle Assignment:** Identify triangles whose AABB overlaps with the partition's world-space bounding box. If no triangles overlap, skip this partition [S1, unofficial].
    *   **Voxelization:**
        *   Allocate a voxel array `m_Voxels` of size $morton\_partition$ [S1, unofficial].
        *   Compute inverse scale: $M = 1.0f / unitLength$ [S1, unofficial].
        *   For each assigned triangle:
            *   Get its bounding box $B_{tri}$ [S1, unofficial].
            *   Compute voxel-space bounds:
                $LX = B_{tri}.minX() \times M$
                $LY = B_{tri}.minY() \times M$
                (and similarly for max bounds) [S1, unofficial].
            *   Perform Schwarz's triangle-box overlap tests to determine which voxels in the current partition are intersected by the triangle [S1, unofficial].
            *   Mark the corresponding voxels in `m_Voxels` [S1, unofficial].
    *   **Octree Construction:** Process the `m_Voxels` array to build the octree structure for this partition, handling empty and full nodes efficiently (e.g., using `FastAddEmptyVoxel` and `Refine`) [S1, unofficial].
4.  **Finalization:** After all partitions are processed, finalize the octree structure and write it to disk [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def morton_partitioned_voxelization(mesh, gridSize, memoryLimitMB):
    # Step 1: Preprocessing
    bbox = compute_bounding_box(mesh.vertices)
    cubify_bbox(bbox) # Ensure max - min is equal for x, y, z
    translate_vertices(mesh.vertices, -bbox.min) # Shift min to origin
    
    # Step 2: Partition Sizing
    # sizeof(char) assumed for voxel storage
    size_mb = (gridSize ** 3 * 1) / (1024 ** 2)
    numPartitions = 1
    while size_mb > memoryLimitMB:
        size_mb /= 8
        numPartitions *= 8
        
    unitLength = (bbox.max.x - bbox.min.x) / gridSize
    morton_partition = (gridSize ** 3) / numPartitions
    
    # Step 3: Partition Iteration
    for i in range(numPartitions):
        # Compute Morton bounds
        mcMin = decode_morton(morton_partition * i)
        mcMax = decode_morton(morton_partition * (i + 1) - 1)
        
        # Compute world-space bounds for partition
        part_min_x = unitLength * mcMin.x
        part_max_x = unitLength * (mcMax.x + 1)
        part_min_y = unitLength * mcMin.y
        part_max_y = unitLength * (mcMax.y + 1)
        part_min_z = unitLength * mcMin.z
        part_max_z = unitLength * (mcMax.z + 1)
        
        part_bbox = BBox(part_min_x, part_min_y, part_min_z, part_max_x, part_max_y, part_max_z)
        
        # Triangle Assignment
        assigned_tris = [t for t in mesh.triangles if aabb_overlap(t.bbox, part_bbox)]
        
        if not assigned_tris:
            continue
            
        # Voxelization
        voxels = allocate_array(morton_partition) # Initialize to empty
        M = 1.0 / unitLength
        
        for tri in assigned_tris:
            tri_bbox = tri.bbox
            # Voxel space bounds
            lx = tri_bbox.min.x * M
            ly = tri_bbox.min.y * M
            lz = tri_bbox.min.z * M
            hx = tri_bbox.max.x * M
            hy = tri_bbox.max.y * M
            hz = tri_bbox.max.z * M
            
            # Mark voxels using Schwarz's triangle-box test
            mark_voxels(voxels, lx, ly, lz, hx, hy, hz, morton_partition, gridSize)
            
        # Build Octree for this partition
        build_octree_partition(voxels, morton_partition, gridSize)
        
    # Step 4: Finalization
    finalize_octree()
```

**Parameters:**
*   $gridSize$: Resolution of the voxel grid along one axis. Must be a power of 2 for Morton encoding compatibility [S1, unofficial].
*   $m_{MemoryLimit}$: Maximum memory in MB allowed for the voxel array before partitioning [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Non-cube bounding boxes cause voxelization errors. Handled by cubifying the bounding box before processing [S1, unofficial].
*   Coordinate system mismatches or negative bounds cause incorrect voxel mapping. Handled by translating all vertices so the minimum bounds are at the origin [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and voxels, but partitioned. The Morton ordering allows efficient spatial locality during voxel marking and octree construction.
*   **Memory Complexity:** Logarithmic in the size of the SVO. By processing partitions sequentially, the peak memory usage is bounded by the size of one partition plus the octree structure built so far, rather than the entire voxel grid [S1, unofficial].
*   **Strengths:** Enables construction of very large SVOs that would otherwise exceed RAM. Morton ordering ensures spatial coherence, which is beneficial for compression and traversal.
*   **Limitations:** Requires careful tuning of $gridSize$ and $m_{MemoryLimit}$. Morton decoding/encoding adds computational overhead. Precision issues may arise if $gridSize$ is very large or coordinates are not properly normalized.

## Background
This method originates from Giles Penfold's Master's Thesis on Sparse Voxel Octrees [S1, unofficial]. It was developed to address the memory constraints of building large-scale SVOs from triangle meshes, providing a practical out-of-core solution. The implementation details are found in the associated GitHub repository [S1, unofficial].

## Sources
[S1] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
