# Sparse Voxel Octree Construction from Density Field

Also called: ConstructOctreeFromField pipeline, OpenCL Octree Construction from Density Field

## Overview
This method converts a 3D density field into a compact sparse voxel octree (SVO) structure using an OpenCL-based pipeline [S1, unofficial]. It identifies active voxels, compacts them into a contiguous node list, and computes leaf node data including quadratic error function (QEF) fits, normals, and materials [S1, unofficial]. It is distinguished by its explicit handling of OpenCL/OpenGL interop limitations, opting for a download-reupload strategy to avoid rendering thread blocking [S1, unofficial].

## How it works
**Inputs:**
- `field`: A structure containing the 3D density field data, including `field.numEdges`, `field.materials`, `field.edgeIndices`, `field.normals`, and `field.size` [S1, unofficial].
- `meshGen`: A configuration structure containing `voxelsPerChunk` [S1, unofficial].
- `octree`: The target octree structure with buffers `octree->d_nodeCodes`, `octree->d_nodeMaterials`, `octree->d_vertexPositions`, and `octree->d_vertexNormals` [S1, unofficial].

**Outputs:**
- `octree`: Populated with compacted node codes, materials, vertex positions, and normals for active chunks [S1, unofficial].

**Data Structures:**
- `chunkBufferSize`: Integer, size `meshGen->voxelsPerChunk * meshGen->voxelsPerChunk * meshGen->voxelsPerChunk` [S1, unofficial].
- `d_leafOccupancy`, `d_leafEdgeInfo`, `d_leafCodes`, `d_leafMaterials`, `d_voxelScan`: `cl::Buffer` of type `CL_MEM_READ_WRITE`, size `chunkBufferSize * sizeof(int)` [S1, unofficial].
- `d_compactLeafEdgeInfo`, `octree->d_nodeCodes`, `octree->d_nodeMaterials`: `CL_MEM_READ_WRITE` buffers, size `octree->numNodes * sizeof(int)` [S1, unofficial].
- `octree->d_vertexPositions`, `octree->d_vertexNormals`: `CL_MEM_READ_WRITE` buffers, size `octree->numNodes * sizeof(cl_float4)` [S1, unofficial].
- `d_qefs`: `CL_MEM_READ_WRITE` buffer, size `octree->numNodes * sizeof(QEFData)` [S1, unofficial].
- `edgeHashTable`: Cuckoo hash table initialized with `field.numEdges` [S1, unofficial].

**Parameters:**
- `sampleScale`: Scaling factor for leaf creation, calculated as `field.size / (meshGen->voxelsPerChunk * LEAF_SIZE_SCALE)` [S1, unofficial]. `LEAF_SIZE_SCALE` is not defined in the sources [G].

**Steps:**
1. Check if `field.numEdges == 0`. If true, return `CL_SUCCESS` immediately [S1, unofficial].
2. Launch `FindActiveVoxels` kernel with NDRange `(meshGen->voxelsPerChunk, meshGen->voxelsPerChunk, meshGen->voxelsPerChunk)`. Bind `field.materials`, `d_leafOccupancy`, `d_leafEdgeInfo`, `d_leafCodes`, and `d_leafMaterials` [S1, unofficial].
3. Compute `octree->numNodes` using an exclusive scan on `d_leafOccupancy` into `d_voxelScan` with size `chunkBufferSize` [S1, unofficial].
4. If `octree->numNodes <= 0`, set `octree->numNodes = 0` and return the error code [S1, unofficial].
5. Launch `CompactVoxels` kernel with NDRange `chunkBufferSize`. Bind `d_leafOccupancy`, `d_leafEdgeInfo`, `d_leafCodes`, `d_leafMaterials`, `d_voxelScan`, `octree->d_nodeCodes`, `d_compactLeafEdgeInfo`, and `octree->d_nodeMaterials` [S1, unofficial].
6. Initialize Cuckoo hash table: `Cuckoo_InitialiseTable(&edgeHashTable, field.numEdges)` [S1, unofficial].
7. Insert keys: `Cuckoo_InsertKeys(&edgeHashTable, field.edgeIndices, field.numEdges)` [S1, unofficial].
8. Launch `CreateLeafNodes` kernel with NDRange `octree->numNodes`. Bind `sampleScale`, `octree->d_nodeCodes`, `d_compactLeafEdgeInfo`, `field.normals`, `octree->d_vertexNormals`, `d_qefs`, and cuckoo table components (`table`, `stash`, `prime`, `hashParams`, `stashUsed`) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ConstructOctreeFromDensityField(field, meshGen, octree):
    # Step 1: Early exit check
    if field.numEdges == 0:
        return CL_SUCCESS

    # Calculate buffer sizes
    voxelsPerChunk = meshGen.voxelsPerChunk
    chunkBufferSize = voxelsPerChunk * voxelsPerChunk * voxelsPerChunk
    
    # Step 2: Find Active Voxels
    # Launch OpenCL kernel: FindActiveVoxels
    # NDRange: (voxelsPerChunk, voxelsPerChunk, voxelsPerChunk)
    # Bindings: field.materials, d_leafOccupancy, d_leafEdgeInfo, d_leafCodes, d_leafMaterials
    execute_kernel_FindActiveVoxels(
        ndrange=(voxelsPerChunk, voxelsPerChunk, voxelsPerChunk),
        materials=field.materials,
        leaf_occupancy=d_leafOccupancy,
        leaf_edge_info=d_leafEdgeInfo,
        leaf_codes=d_leafCodes,
        leaf_materials=d_leafMaterials
    )

    # Step 3: Compute Node Count via Exclusive Scan
    # ExclusiveScan(ctx->queue, d_leafOccupancy, d_voxelScan, chunkBufferSize)
    execute_kernel_ExclusiveScan(
        queue=ctx.queue,
        input=d_leafOccupancy,
        output=d_voxelScan,
        size=chunkBufferSize
    )
    
    # Retrieve octree->numNodes from d_voxelScan result (implementation detail not specified)
    octree.numNodes = get_num_nodes_from_scan(d_voxelScan)

    # Step 4: Edge Case Handling
    if octree.numNodes <= 0:
        octree.numNodes = 0
        return ERROR_CODE_NO_SURFACE # Placeholder for specific error code

    # Step 5: Compact Voxels
    # Launch OpenCL kernel: CompactVoxels
    # NDRange: chunkBufferSize
    # Bindings: d_leafOccupancy, d_leafEdgeInfo, d_leafCodes, d_leafMaterials, 
    #           d_voxelScan, octree->d_nodeCodes, d_compactLeafEdgeInfo, octree->d_nodeMaterials
    execute_kernel_CompactVoxels(
        ndrange=chunkBufferSize,
        leaf_occupancy=d_leafOccupancy,
        leaf_edge_info=d_leafEdgeInfo,
        leaf_codes=d_leafCodes,
        leaf_materials=d_leafMaterials,
        voxel_scan=d_voxelScan,
        node_codes=octree.d_nodeCodes,
        compact_leaf_edge_info=d_compactLeafEdgeInfo,
        node_materials=octree.d_nodeMaterials
    )

    # Step 6: Initialize Cuckoo Hash Table
    edgeHashTable = Cuckoo_InitialiseTable(field.numEdges)

    # Step 7: Insert Edge Indices
    Cuckoo_InsertKeys(edgeHashTable, field.edgeIndices, field.numEdges)

    # Step 8: Create Leaf Nodes
    # Calculate sampleScale
    LEAF_SIZE_SCALE = 1.0 # Not given in sources [G]
    sampleScale = field.size / (voxelsPerChunk * LEAF_SIZE_SCALE)

    # Launch OpenCL kernel: CreateLeafNodes
    # NDRange: octree->numNodes
    # Bindings: sampleScale, octree->d_nodeCodes, d_compactLeafEdgeInfo, field.normals,
    #           octree->d_vertexNormals, d_qefs, table, stash, prime, hashParams, stashUsed
    execute_kernel_CreateLeafNodes(
        ndrange=octree.numNodes,
        sample_scale=sampleScale,
        node_codes=octree.d_nodeCodes,
        compact_leaf_edge_info=d_compactLeafEdgeInfo,
        normals=field.normals,
        vertex_normals=octree.d_vertexNormals,
        qefs=d_qefs,
        cuckoo_table=edgeHashTable.table,
        cuckoo_stash=edgeHashTable.stash,
        cuckoo_prime=edgeHashTable.prime,
        cuckoo_hash_params=edgeHashTable.hashParams,
        cuckoo_stash_used=edgeHashTable.stashUsed
    )
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Linear with respect to the number of active voxels and edges [S1, unofficial].
- **Performance:** The pipeline downloads generated mesh data to main memory before reuploading to OpenGL, which is less than optimal [S1, unofficial].
- **Strengths:** Avoids rendering thread blocking associated with OpenCL/OpenGL interop [S1, unofficial].
- **Limitations:** Data transfer overhead between APIs; potential resource starvation causing hitches in OpenGL operations [S1, unofficial].
- **When to use:** When using OpenCL for voxel generation and OpenGL for rendering, and Vulkan is not available [S1, unofficial].
- **When not to use:** When low-latency interop is required; Vulkan is suggested as a solution [S1, unofficial].

## Background
This method originates from the "leven" project by Nick Gildea, an unofficial personal project using OpenCL for GPGPU voxel component generation and OpenGL for rendering [S1, unofficial]. The author noted that while OpenCL and OpenGL might seem compatible, their interop causes the rendering thread to block during GPU data transfer, leading to unworkable performance [S1, unofficial]. The pipeline was designed as a workaround to these interop issues [S1, unofficial].

## Sources
[S1] nickgildea/leven, https://github.com/nickgildea/leven, fetched 2026-10-26 (unofficial)
