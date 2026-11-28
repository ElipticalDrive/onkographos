# Sparse Voxel Octree Construction via GPU Coarse-to-Fine Voxelization

Also called: GPU Coarse-to-Fine SVO Construction, Iterative Coarse-to-Fine SVO Builder

## Overview
An algorithm to construct a Sparse Voxel Octree (SVO) from a mesh by iteratively voxelizing the scene at increasing resolutions (coarse nodes), compacting occupied nodes, and building the octree hierarchy level by level on the GPU using thrust scans and custom CUDA kernels [S1, unofficial]. It differs from standard SVO builders by starting with a coarse voxelization of the entire bounding box and refining only occupied regions, reducing memory overhead for sparse scenes compared to dense voxelization approaches [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `surfaceVoxelGridSize` (initial coarse grid resolution, `Eigen::Vector3i`), `modelBBox` (model bounding box), and a mesh.
*   **Output:** A complete Sparse Voxel Octree stored in a flat array `d_SVONodeArray`.

**Data Structures:**
*   `SVONode`: Contains `mortonCode` (uint32_t), `origin` (Vector3f), `width` (float), `isLeaf` (bool), `childs` (array of 8 uint32_t indices), and `parent` (uint32_t index).
*   `d_CNodeMortonArray`: Device array of uint32_t storing Morton codes of occupied coarse voxels.
*   `d_isValidCNodeArray`: Device array of uint32_t (flag array) indicating valid nodes.
*   `d_esumCNodesArray`: Device array of size_t for exclusive scan results.
*   `d_pactCNodeArray`: Device array of uint32_t storing compacted, valid Morton codes with flags.
*   `d_numTreeNodesArray`: Device array of size_t, one entry per compacted node, storing the number of children (8) or 0.
*   `d_sumTreeNodesArray`: Device array of size_t for inclusive scan results.
*   `d_nodeArray`: Device array of SVONode, storing the current level's nodes.
*   `d_begMortonArray`: Device array of uint32_t, storing the base Morton code for each parent node's children block.
*   `d_SVONodeArray`: Global device array accumulating all tree nodes.
*   `d_esumTreeNodesArray`: Device array of size_t, storing cumulative sums of nodes per depth level.

**Constants:**
*   `E_MORTON_32_FLAG`: `0x80000000` (Sign bit, indicates valid/parent node).
*   `D_MORTON_32_FLAG`: `0x7FFFFFFF` (Mask for lower 31 bits).
*   `LOWER_3BIT_MASK`: `0x00000007` (Mask for child index 0-7).

**Steps:**

1.  **Initialization:**
    *   Calculate initial coarse grid size: `gridCNodeSize = mortonEncode_LUT(x-1, y-1, z-1) + 1` for the bounding box dimensions [S1, unofficial].
    *   Align `gridTreeNodeSize` to the next multiple of 8 [S1, unofficial].
    *   Allocate device memory for `gridCNodeSize`, `gridTreeNodeSize`, `gridOrigin`, `unitVoxelSize`, and `unitNodeWidth`. Copy host values to device.
    *   Initialize `d_CNodeMortonArray` of size `gridCNodeSize` with zeros.
    *   Launch `meshVoxelize` kernel to populate `d_CNodeMortonArray` with Morton codes of occupied coarse voxels [S1, unofficial].

2.  **Compaction Loop (Per Depth Level):**
    *   Resize `d_isValidCNodeArray` to `gridCNodeSize`.
    *   Transform `d_CNodeMortonArray` to `d_isValidCNodeArray` using `scanMortonFlag<uint32_t>()`. This sets flags for valid nodes [S1, unofficial].
    *   Perform exclusive scan on `d_isValidCNodeArray` into `d_esumCNodesArray`.
    *   Calculate `numCNodes = last element of d_esumCNodesArray + last element of d_isValidCNodeArray`. If `numCNodes` is 0, break loop and print `treeDepth` [S1, unofficial].
    *   (Debug) Copy `d_pactCNodeArray` to host vector `h_pactCNodeArray` if `NDEBUG` is not defined [S1, unofficial].
    *   If `numCNodes > 1`:
        *   Resize `d_numTreeNodesArray` to `numCNodes` with 0.
        *   Clear and resize `d_CNodeMortonArray` to `gridTreeNodeSize`.
        *   Get first parent Morton code from `d_pactCNodeArray[0]` and set `d_CNodeMortonArray[firstMortonCode] = firstMortonCode | E_MORTON_32_FLAG`.
        *   Launch `cpNumNodes` kernel to populate `d_numTreeNodesArray` [S1, unofficial].
    *   Perform inclusive scan on `d_numTreeNodesArray` into `d_sumTreeNodesArray`.
    *   Calculate `numNodes = last element of d_sumTreeNodesArray + 8`. Push `numNodes` to `depthNumNodes`.
    *   Clear and resize `d_nodeArray` to `numNodes` with default `SVONode()`.
    *   Get `maxMortonCode` from last element of `d_pactCNodeArray`.
    *   Clear and resize `d_begMortonArray` to `numCNodes`.

3.  **Node Creation:**
    *   If `treeDepth < 2`:
        *   Launch `createNode_1` kernel with `numCNodes` threads. Pass `d_sumTreeNodesArray`, `d_pactCNodeArray`, `d_gridOrigin`, `unitNodeWidth`, `d_begMortonArray`, `d_nodeArray`.
        *   Push 0 to `d_esumTreeNodesArray`.
    *   If `treeDepth >= 2`:
        *   Launch `createNode_2` kernel with `numCNodes` threads. Pass `d_preChildDepthTreeNodes` (second to last of `d_esumTreeNodesArray`), `d_preDepthTreeNodes` (last of `d_esumTreeNodesArray`), `d_sumTreeNodesArray`, `d_pactCNodeArray`, `d_gridOrigin`, `unitNodeWidth`, `d_begMortonArray`, `d_nodeArray`, and `d_SVONodeArray` offset by `d_preChildDepthTreeNodes`.
    *   Append `d_nodeArray` to `d_SVONodeArray`.
    *   Push cumulative sum of `numNodes` to `d_esumTreeNodesArray`.

4.  **Termination Check:**
    *   Find max element in `d_CNodeMortonArray`.
    *   Check if bit 31 is set (`E_MORTON_32_FLAG`). Calculate `numParentCNodes = (max & D_MORTON_32_FLAG) + isValidMorton`. If `numParentCNodes != 0`:
        *   Resize `d_CNodeMortonArray` to `numParentCNodes`.
        *   Double `unitNodeWidth`. Copy new `unitNodeWidth` to device.
        *   Update `gridCNodeSize` and `gridTreeNodeSize`.
        *   If `numNodes/8 == 0`, break. Else continue loop [S1, unofficial].
    *   After loop, set `numTreeNodes` to last element of `d_esumTreeNodesArray`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
E_MORTON_32_FLAG = 0x80000000
D_MORTON_32_FLAG = 0x7FFFFFFF
LOWER_3BIT_MASK = 0x00000007

def build_svo_coarse_to_fine(mesh, surfaceVoxelGridSize, modelBBox):
    # 1. Initialization
    gridOrigin = modelBBox.min
    unitVoxelSize = (modelBBox.max - modelBBox.min) / surfaceVoxelGridSize
    unitNodeWidth = unitVoxelSize.x  # Assuming uniform scaling initially
    
    # Calculate initial coarse grid size based on bounding box and voxel size
    # mortonEncode_LUT is a lookup table based Morton encoder
    gridCNodeSize = morton_encode_3d(surfaceVoxelGridSize.x - 1, 
                                     surfaceVoxelGridSize.y - 1, 
                                     surfaceVoxelGridSize.z - 1) + 1
    gridTreeNodeSize = align_to_multiple(gridCNodeSize, 8)
    
    # Allocate device arrays
    d_CNodeMortonArray = allocate_device_array(gridCNodeSize, dtype=uint32, init=0)
    d_gridOrigin = copy_to_device(gridOrigin)
    d_unitNodeWidth = copy_to_device(unitNodeWidth)
    
    # Voxelize mesh into coarse grid
    launch_kernel(meshVoxelize, gridCNodeSize, args=[d_CNodeMortonArray, mesh, gridOrigin, unitVoxelSize])
    
    treeDepth = 0
    d_SVONodeArray = allocate_device_array(0, dtype=SVONode)
    d_esumTreeNodesArray = allocate_device_array(0, dtype=size_t)
    
    while True:
        # 2. Compaction
        d_isValidCNodeArray = allocate_device_array(gridCNodeSize, dtype=uint32)
        
        # Flag valid nodes and scan
        launch_kernel(scanMortonFlag, gridCNodeSize, args=[d_CNodeMortonArray, d_isValidCNodeArray])
        
        d_esumCNodesArray = exclusive_scan(d_isValidCNodeArray)
        
        last_valid = d_isValidCNodeArray[gridCNodeSize - 1]
        last_esum = d_esumCNodesArray[gridCNodeSize - 1]
        numCNodes = last_esum + last_valid
        
        if numCNodes == 0:
            break
            
        # Prepare for node creation
        d_pactCNodeArray = compact(d_CNodeMortonArray, d_isValidCNodeArray, d_esumCNodesArray)
        
        if numCNodes > 1:
            d_numTreeNodesArray = allocate_device_array(numCNodes, dtype=size_t, init=0)
            # cpNumNodes kernel sets 8 for each parent node
            launch_kernel(cpNumNodes, numCNodes, args=[d_pactCNodeArray, d_numTreeNodesArray])
            
            d_sumTreeNodesArray = inclusive_scan(d_numTreeNodesArray)
            numNodes = d_sumTreeNodesArray[numCNodes - 1] + 8
        else:
            # If only 1 node, it might be a leaf or root, handle accordingly
            # Based on logic, if numCNodes <= 1, we might not create children
            numNodes = 8 # Placeholder for single node expansion if needed
            d_sumTreeNodesArray = [numNodes]
            
        # Resize and clear arrays for current level
        d_nodeArray = allocate_device_array(numNodes, dtype=SVONode, init_default)
        d_begMortonArray = allocate_device_array(numCNodes, dtype=uint32)
        
        # 3. Node Creation
        if treeDepth < 2:
            launch_kernel(createNode_1, numCNodes, args=[
                numCNodes, d_sumTreeNodesArray, d_pactCNodeArray, 
                d_gridOrigin, d_unitNodeWidth, d_begMortonArray, d_nodeArray
            ])
            push_to_device_array(d_esumTreeNodesArray, 0)
        else:
            # Get previous depth sums
            preChildDepthTreeNodes = d_esumTreeNodesArray[-2] if len(d_esumTreeNodesArray) >= 2 else 0
            preDepthTreeNodes = d_esumTreeNodesArray[-1]
            
            launch_kernel(createNode_2, numCNodes, args=[
                numCNodes, preChildDepthTreeNodes, preDepthTreeNodes,
                d_sumTreeNodesArray, d_pactCNodeArray, d_gridOrigin, 
                d_unitNodeWidth, d_begMortonArray, d_nodeArray, d_SVONodeArray
            ])
            
        # Append current level nodes to global array
        append_to_device_array(d_SVONodeArray, d_nodeArray)
        push_to_device_array(d_esumTreeNodesArray, numNodes)
        
        # 4. Termination Check
        maxMorton = max(d_CNodeMortonArray)
        isValidMorton = 1 if (maxMorton & E_MORTON_32_FLAG) else 0
        numParentCNodes = (maxMorton & D_MORTON_32_FLAG) + isValidMorton
        
        if numParentCNodes == 0:
            break
            
        # Prepare for next level
        d_CNodeMortonArray = allocate_device_array(numParentCNodes, dtype=uint32)
        unitNodeWidth *= 2
        d_unitNodeWidth = copy_to_device(unitNodeWidth)
        
        # Update grid sizes for next iteration
        # Note: gridCNodeSize and gridTreeNodeSize need to be recalculated based on new unitNodeWidth
        # and the bounding box, or derived from numParentCNodes if it represents the new max index
        # The source implies resizing d_CNodeMortonArray to numParentCNodes, 
        # but subsequent steps use gridCNodeSize. 
        # Assuming gridCNodeSize is updated to accommodate the new max morton code + 1
        gridCNodeSize = numParentCNodes + 1 # Approximation based on source logic
        gridTreeNodeSize = align_to_multiple(gridCNodeSize, 8)
        
        if numNodes / 8 == 0:
            break
            
        treeDepth += 1
        
    return d_SVONodeArray

def createNode_1(pactSize, d_sumNodesArray, d_pactDataArray, d_gridOrigin, d_width, d_begMortonArray, d_nodeArray):
    tid = threadIdx.x + blockIdx.x * blockDim.x
    if tid < pactSize:
        gridOrigin = d_gridOrigin[0]
        width = d_width[0]
        sumNodes = d_sumNodesArray[tid]
        pactData = d_pactDataArray[tid]
        
        key = pactData & LOWER_3BIT_MASK
        morton = pactData & D_MORTON_32_FLAG
        
        address = sumNodes + key
        tNode = d_nodeArray[address]
        
        tNode.mortonCode = morton
        x, y, z = morton3D_32_decode(morton)
        tNode.origin = gridOrigin + width * Vector3f(x, y, z)
        tNode.width = width
        # isLeaf is not set in createNode_1, implying it might be a leaf or default
        # Based on createNode_2 setting isLeaf=false, createNode_1 nodes are likely leaves or root
        
        d_begMortonArray[tid] = (morton // 8) * 8

def createNode_2(pactSize, d_preChildDepthTreeNodes, d_preDepthTreeNodes, d_sumNodesArray, d_pactDataArray, d_gridOrigin, d_width, d_begMortonArray, d_nodeArray, d_childArray):
    tid = threadIdx.x + blockIdx.x * blockDim.x
    if tid < pactSize:
        gridOrigin = d_gridOrigin[0]
        width = d_width[0]
        sumNodes = d_sumNodesArray[tid]
        pactData = d_pactDataArray[tid]
        
        key = pactData & LOWER_3BIT_MASK
        morton = pactData & D_MORTON_32_FLAG
        
        address = sumNodes + key
        tNode = d_nodeArray[address]
        
        tNode.mortonCode = morton
        x, y, z = morton3D_32_decode(morton)
        tNode.origin = gridOrigin + width * Vector3f(x, y, z)
        tNode.width = width
        tNode.isLeaf = False
        
        d_begMortonArray[tid] = (morton // 8) * 8
        
        for i in range(8):
            tNode.childs[i] = d_preChildDepthTreeNodes + tid * 8 + i
            d_childArray[tid * 8 + i].parent = d_preDepthTreeNodes + sumNodes + key

def createRemainNode(nNodes, d_gridOrigin, d_width, d_begMortonArray, d_nodeArray):
    # Shared memory for base morton codes
    sh_begMortonArray = allocate_shared_array(num_threads // 8, dtype=uint32)
    
    tid = threadIdx.x + blockIdx.x * blockDim.x
    if tid < nNodes:
        sh_begMortonArray[threadIdx.x // 8] = d_begMortonArray[tid // 8]
        __syncthreads()
        
        if d_nodeArray[tid].mortonCode == 0:
            gridOrigin = d_gridOrigin[0]
            width = d_width[0]
            key = tid & LOWER_3BIT_MASK
            morton = sh_begMortonArray[threadIdx.x // 8] + key
            
            tNode = d_nodeArray[tid]
            tNode.mortonCode = morton
            x, y, z = morton3D_32_decode(morton)
            tNode.origin = gridOrigin + width * Vector3f(x, y, z)
            tNode.width = width
```

## Operating characteristics
*   **Time Complexity:** Dominated by the voxelization step and the hierarchical scan operations. Each level requires a scan and kernel launches proportional to the number of occupied nodes. The number of levels is logarithmic with respect to the resolution.
*   **Memory Complexity:** Proportional to the number of nodes in the SVO. The algorithm uses temporary arrays for scans and flags, roughly doubling the memory requirement per level during construction.
*   **Performance:** Efficient for sparse scenes as it avoids allocating memory for empty coarse voxels. Performance scales with the number of occupied voxels and the depth of the tree.
*   **Strengths:** Reduces memory overhead for sparse data; leverages GPU parallelism for voxelization and tree construction.
*   **Limitations:** Requires careful management of device memory and synchronization. The termination condition must be robust to avoid infinite loops if the max Morton code is 0.

## Background
This method is described in the GitHub repository "GPU-SVO" by Alan-Leo-Wong [S1, unofficial]. It implements a coarse-to-fine voxelization strategy to build an SVO, optimizing for sparse datasets by only refining occupied regions.

## Sources
[S1] GitHub - Alan-Leo-Wong/GPU-SVO, https://github.com/Alan-Leo-Wong/GPU-SVO, fetched 2026-10-29 (unofficial)
