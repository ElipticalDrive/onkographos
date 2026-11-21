# WriteLeafOctree

Also called: Leaf Node Traversal, Brick Population

## Overview
WriteLeafOctree is a compute shader that traverses a Sparse Voxel Octree (SVO) to locate the leaf node corresponding to each input voxel, then maps the voxel's attributes into a 3D brick pool texture [S1, unofficial]. It differs from its parent, [Octree Node Lookup by Voxel](5120-octree-node-lookup-by-voxel.md), by not just returning the leaf index but actively populating a structured brick-based texture memory with the voxel data, effectively bridging the gap between sparse tree representation and dense brick-based rendering buffers.

## How it works
**Inputs and Outputs:**
*   **Input:** A list of voxels (`fragmentsList`) containing position and attributes (color, normal, irradiance).
*   **Input:** An SVO node pool (`nodesPool`) representing the tree structure.
*   **Input:** Constant buffers (`cbInfo`, `cbBrickInfo`, `cbGroupInfo`) providing traversal parameters, brick dimensions, and thread group sizes.
*   **Output:** Brick pool textures (`bricksPool_color`, `bricksPool_normal`, `bricksPool_irradiance`) storing the mapped voxel attributes.

**Data Structures:**
*   **Node:** `struct Node { uint brickPtr; uint childPtr; };`
    *   `brickPtr`: Linear index of the leaf brick in the pool.
    *   `childPtr`: Linear index of the child node in the lower 30 bits; the top 2 bits are unused or serve as flags [S1, unofficial].
*   **Voxel:** `struct Voxel { float3 pos; uint normal; uint color; };`
    *   `pos`: 3D position of the voxel.
    *   `normal`: Encoded normal vector.
    *   `color`: Encoded color value.

**Parameters:**
*   `curLevel`, `totalLevel`: Control traversal depth. `totalLevel` defines the maximum depth of the octree [S1, unofficial].
*   `sumVoxels`: Total number of voxels to process [S1, unofficial].
*   `curNode`: Current node index (used in parent context, initialized to 0 here) [S1, unofficial].
*   `brickExtent`: 3D dimensions of each brick in the pool (e.g., `uint3(2, 2, 2)` for a 2x2x2 brick) [S1, unofficial].
*   `groupSize`: Thread group dimensions, typically `uint2(16, 1)` for the dispatch [S1, unofficial].

**Algorithm Steps:**
1.  **Thread Indexing:** Calculate the local voxel index for the current thread.
    $$ \text{voxelIndex} = \text{DTid}.x + \text{DTid}.y \times \text{groupSize}.x \times 16 $$
    where `DTid` is the dispatch thread ID [S1, unofficial].
2.  **Bounds Check:** If `voxelIndex >= sumVoxels`, terminate the shader invocation [S1, unofficial].
3.  **Load Voxel:** Retrieve the voxel data from the input list:
    $$ \text{voxel} = \text{fragmentsList}[\text{voxelIndex}] $$
4.  **Initialize Traversal:** Set the current node index to the root:
    $$ \text{index} = 0 $$
5.  **Tree Traversal:** Iterate from level 0 to `totalLevel - 2` (inclusive) to navigate down the octree:
    *   Compute the scaling factor for the current level:
        $$ tt = 2^{\text{level}} $$
    *   Determine the child index along each axis using the fractional part of the scaled position:
        $$ x = \lfloor \text{frac}(\text{voxel.pos}.x \times tt) \times 2 \rfloor $$
        $$ y = \lfloor \text{frac}(\text{voxel.pos}.y \times tt) \times 2 \rfloor $$
        $$ z = \lfloor \text{frac}(\text{voxel.pos}.z \times tt) \times 2 \rfloor $$
    *   Update the node index by extracting the child pointer and adding the offset for the child $(x, y, z)$:
        $$ \text{index} = \text{extractPos}(\text{nodesPool}[\text{index}].\text{childPtr}) + x + y \times 2 + z \times 4 $$
    *   Note: `extractPos` retrieves the lower 30 bits of `childPtr` [S1, unofficial].
6.  **Locate Leaf Brick:** After the loop, the current `index` points to the leaf node. Extract the brick pointer:
    $$ \text{brickIndex} = \text{nodesPool}[\text{index}].\text{brickPtr} $$
7.  **Decode Brick Coordinates:** Decompose `brickIndex` into brick-local coordinates $(bx, by, bz)$ using `brickExtent`:
    $$ bz = \lfloor \text{brickIndex} / (\text{brickExtent}.x \times \text{brickExtent}.y) \rfloor $$
    $$ \text{brickIndex} = \text{brickIndex} \pmod{(\text{brickExtent}.x \times \text{brickExtent}.y)} $$
    $$ by = \lfloor \text{brickIndex} / \text{brickExtent}.x \rfloor $$
    $$ bx = \text{brickIndex} \pmod{\text{brickExtent}.x} $$
8.  **Compute Voxel Offset within Brick:** Determine the position of the voxel within its 2x2x2 brick. The offset is calculated based on the total level:
    $$ ttt = 2^{\text{totalLevel} - 2} $$
    $$ vx = \lfloor \text{frac}(\text{voxel.pos}.x \times ttt) \times 2 \rfloor $$
    $$ vy = \lfloor \text{frac}(\text{voxel.pos}.y \times ttt) \times 2 \rfloor $$
    $$ vz = \lfloor \text{frac}(\text{voxel.pos}.z \times ttt) \times 2 \rfloor $$
    This yields integer offsets $vx, vy, vz \in \{0, 1\}$ [S1, unofficial].
9.  **Write Attributes:** Write the voxel's attributes to the brick pool textures at the computed global brick-local coordinate $(bx \times 2 + vx, by \times 2 + vy, bz \times 2 + vz)$:
    $$ \text{bricksPool\_color}[\text{uint3}(bx \times 2 + vx, by \times 2 + vy, bz \times 2 + vz)] = \text{voxel.color} $$
    $$ \text{bricksPool\_normal}[\text{uint3}(bx \times 2 + vx, by \times 2 + vy, bz \times 2 + vz)] = \text{voxel.normal} $$
    $$ \text{bricksPool\_irradiance}[\text{uint3}(bx \times 2 + vx, by \times 2 + vy, bz \times 2 + vz)] = 11111 $$
    Note: The irradiance value is hardcoded to `11111` in the source implementation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants and Buffers
# nodesPool: Array of Node structs
# fragmentsList: Array of Voxel structs
# bricksPool_color, bricksPool_normal, bricksPool_irradiance: 3D Textures
# cbInfo: { curLevel, totalLevel, sumVoxels, curNode }
# cbBrickInfo: { brickExtent: uint3 }
# cbGroupInfo: { groupSize: uint2 }

struct Node:
    brickPtr: uint
    childPtr: uint

struct Voxel:
    pos: float3
    normal: uint
    color: uint

def extractPos(childPtr: uint) -> uint:
    # Extract lower 30 bits
    return childPtr & 0x3FFFFFFF

[numthreads(16, 16, 1)]
def main(DTid: uint3):
    # 1. Calculate thread-local voxel index
    voxelIndex = DTid.x + DTid.y * cbGroupInfo.groupSize.x * 16
    
    # 2. Check bounds
    if voxelIndex >= cbInfo.sumVoxels:
        return

    # 3. Load voxel data
    voxel = fragmentsList[voxelIndex]
    
    # 4. Initialize octree traversal index to root
    index = 0
    
    # 5. Iterate through octree levels to find leaf node
    # Loop from level 0 to totalLevel - 2
    for level in range(0, cbInfo.totalLevel - 1):
        tt = 2 ** level
        # Compute child coordinates using fractional position scaled by 2^level
        x = int(frac(voxel.pos.x * tt) * 2.0)
        y = int(frac(voxel.pos.y * tt) * 2.0)
        z = int(frac(voxel.pos.z * tt) * 2.0)
        
        # Update index: extract child pointer and add offset
        child_ptr = nodesPool[index].childPtr
        base_child = extractPos(child_ptr)
        index = base_child + x + y * 2 + z * 4
        
    # 6. Extract leaf brick pointer
    brickIndex = nodesPool[index].brickPtr
    
    # 7. Decode brick coordinates from brickIndex using brickExtent
    brick_extent_x = cbBrickInfo.brickExtent.x
    brick_extent_y = cbBrickInfo.brickExtent.y
    brick_extent_z = cbBrickInfo.brickExtent.z # Not used in division but part of extent
    
    bz = brickIndex // (brick_extent_x * brick_extent_y)
    brickIndex = brickIndex % (brick_extent_x * brick_extent_y)
    by = brickIndex // brick_extent_x
    bx = brickIndex % brick_extent_x
    
    # 8. Compute voxel position within the brick using totalLevel
    ttt = 2 ** (cbInfo.totalLevel - 2)
    vx = int(frac(voxel.pos.x * ttt) * 2)
    vy = int(frac(voxel.pos.y * ttt) * 2)
    vz = int(frac(voxel.pos.z * ttt) * 2)
    
    # 9. Write voxel attributes to brick pool textures
    target_coord = uint3(bx * 2 + vx, by * 2 + vy, bz * 2 + vz)
    
    bricksPool_color[target_coord] = voxel.color
    bricksPool_normal[target_coord] = voxel.normal
    bricksPool_irradiance[target_coord] = 11111
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of `frac` and floating-point multiplication can introduce precision errors for voxels near brick boundaries or at high octree depths. [G] Ensure that `voxel.pos` is normalized or consistently scaled relative to the octree's world space to avoid misalignment.
*   **Boundary Conditions:** The loop runs up to `totalLevel - 1`. If `totalLevel` is 0 or 1, the loop may not execute as expected, potentially leaving `index` at the root. [G] Validate that `totalLevel` is sufficient to reach leaf nodes for the given voxel resolution.
*   **Hardcoded Irradiance:** The irradiance value is hardcoded to `11111`. This is likely a placeholder or specific to the demo's lighting model and should be replaced with actual computed irradiance in a production implementation [S1, unofficial].
*   **Child Pointer Flags:** The top 2 bits of `childPtr` are ignored. If these bits are used for flags (e.g., "is leaf"), the traversal logic must account for them to avoid misinterpreting internal nodes as leaves or vice versa. The current implementation assumes `childPtr` always points to a valid child node for internal nodes.

## Operating characteristics
*   **Time Complexity:** $O(N \times D)$, where $N$ is the number of voxels and $D$ is the depth of the octree (`totalLevel`). Each voxel requires a traversal from root to leaf.
*   **Memory Complexity:** $O(N)$ for the input voxel list and output brick textures. The SVO node pool is $O(M)$, where $M$ is the number of nodes.
*   **Performance:** Performance is heavily dependent on the efficiency of the compute shader dispatch and memory bandwidth for writing to the brick textures. The use of a fixed 2x2x2 brick size simplifies the offset calculation but may lead to fragmentation if the voxel distribution is irregular.
*   **Strengths:** Simple implementation, parallelizable across voxels, efficient for dense regions where many voxels map to nearby bricks.
*   **Limitations:** Requires a pre-constructed SVO. The hardcoded irradiance value limits its general applicability. Precision issues with floating-point arithmetic may cause artifacts in high-resolution scenes.

## Background
WriteLeafOctree is part of the SVODemo project, an unofficial implementation of Sparse Voxel Octree rendering [S1, unofficial]. It serves as a bridge between the sparse tree structure and the brick-based rendering pipeline, allowing for efficient storage and access of voxel attributes in a texture-friendly format.

## Sources
[S1] Throbbing/SVODemo, https://github.com/Throbbing/SVODemo, fetched 2026-10-26 (unofficial)
