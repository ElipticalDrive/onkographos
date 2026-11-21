# Sparse Voxel Octree Leaf Population

Also called: Leaf Population, Voxel Fragment to Octree Leaf Writing, WriteLeafs

## Overview
This method populates the leaf nodes of a Sparse Voxel Octree (SVO) with voxel color, normal, and irradiance data derived from a flat list of voxel fragments [S1, unofficial]. It serves as the final stage of the SVO construction pipeline, translating raw voxelized geometry into the hierarchical brick structure required for ray-marching or cone-tracing rendering [S1, unofficial]. Unlike sibling methods that focus on octree topology generation or mipmap creation, this step is strictly concerned with data transfer and packing into the leaf-level texture buffers [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a list of voxel fragments (positions, colors, normals) and the current state of the SVO node pool as input. It outputs populated `brickPool` textures (color, normal, irradiance) for the leaf nodes.

**Data Structures:**
The implementation relies on OpenGL Image Units (UAVs) for atomic or unordered access. The specific layouts are:
*   `voxelFragList_position`: `uimageBuffer` with format `r32ui`. Stores packed 30-bit positions (10 bits per axis) [S1, unofficial].
*   `voxelFragTex_color`: `uimage3D` with format `r32ui`. Stores packed color values per voxel [S1, unofficial].
*   `voxelFragTex_normal`: `uimage3D` with format `r32ui`. Stores packed normal values per voxel [S1, unofficial].
*   `nodePool_color`: `uimageBuffer` with format `r32ui`. Stores the brick pool index (address) for each octree node [S1, unofficial].
*   `brickPool_color`: `image3D` with format `rgba8`. Stores color data per brick corner [S1, unofficial].
*   `brickPool_normal`: `image3D` with format `rgba8`. Stores normal data per brick corner [S1, unofficial].
*   `brickPool_irradiance`: `image3D` with format `rgba8`. Stores irradiance data per brick corner [S1, unofficial].

**Parameters:**
*   `voxelGridResolution`: The resolution of the input voxel grid, used to normalize voxel positions to the [0,1] range [S1, unofficial].
*   `numLevels`: The total number of levels in the octree [S1, unofficial].

**Algorithm Steps:**
1.  **Invocation:** Each shader invocation processes one voxel fragment. The invocation index is derived from `gl_VertexID` [S1, unofficial].
2.  **Position Unpacking:** Load the packed position from `voxelFragList_position`. Unpack the 30-bit integer into a `uvec3 voxelPos` representing the 10-bit per axis coordinates [S1, unofficial].
3.  **Normalization:** Compute the normalized texture coordinate `posTex` by dividing `voxelPos` by `voxelGridResolution`:
    $$ posTex = \frac{vec3(voxelPos)}{vec3(voxelGridResolution)} $$
4.  **Tree Traversal:** Call `traverseOctree_posOut(posTex, onLevel)` to determine the address of the leaf node in the octree corresponding to this voxel [S1, unofficial]. This function returns `nodeAddress` [S1, unofficial].
5.  **Node Data Load:** Load the brick pool index from `nodePool_color` at `nodeAddress`:
    ```glsl
    uint nodeColorU = imageLoad(nodePool_color, nodeAddress).x;
    ```
    A `memoryBarrier()` is required here to ensure visibility of previous writes to the node pool [S1, unofficial].
6.  **Brick Coordinate Calculation:** Convert the 10-bit `nodeColorU` into 3D brick coordinates `brickCoords`:
    ```glsl
    ivec3 brickCoords = ivec3(uintXYZ10ToVec3(nodeColorU));
    ```
7.  **Child Offset Calculation:** Determine the specific corner within the brick (child node) to store the data.
    *   Compute `offVec`:
        ```glsl
        uvec3 offVec = uvec3(2.0 * posTex);
        ```
    *   Compute linear offset `off`:
        $$ off = offVec.x + 2U \cdot offVec.y + 4U \cdot offVec.z $$
    *   The final index within the brick is determined by `childOffsets[off]`. The target location is `brickCoords + 2 * ivec3(childOffsets[off])` [S1, unofficial]. Note: The factor `2 *` suggests the brick storage might be interleaved or padded, or `childOffsets` maps to a specific stride. The source code explicitly uses `2 * ivec3(childOffsets[off])` [S1, unofficial].
8.  **Data Storage:** Store the voxel's color and normal to the respective brick pools. The values are converted from packed uints to `vec4` and scaled by $1/255.0$ to fit the `rgba8` format [S1, unofficial]. Irradiance is initialized to zero [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode representing the GLSL compute shader logic
# Assumes standard math/vector operations and image load/store semantics

def process_voxel_fragment(gl_VertexID):
    # 1. Load Position
    packed_pos = image_load(voxelFragList_position, gl_VertexID)
    voxel_pos = unpack_30bit_to_uvec3(packed_pos) # 10 bits per axis
    
    # 2. Normalize
    pos_tex = vec3(voxel_pos) / vec3(voxelGridResolution)
    
    # 3. Traverse Octree to find leaf node address
    # Note: traverseOctree_posOut is a helper function defined in the parent SVO Builder context
    node_address = traverse_octree_pos_out(pos_tex, numLevels)
    
    # 4. Load Node Brick Index
    node_color_u = image_load(nodePool_color, node_address).x
    memory_barrier() # Ensure node pool write is visible
    
    # 5. Convert Node Index to Brick Coordinates
    brick_coords = uint_xyz10_to_vec3(node_color_u)
    
    # 6. Calculate Child Offset within Brick
    off_vec = uvec3(2.0 * pos_tex)
    off = off_vec.x + 2U * off_vec.y + 4U * off_vec.z
    
    # Helper: childOffsets is a pre-defined lookup table mapping linear offset 0-7 to 3D child indices
    # The source uses: brickCoords + 2 * ivec3(childOffsets[off])
    target_offset = 2 * ivec3(child_offsets[off])
    target_coord = brick_coords + target_offset
    
    # 7. Load Voxel Color and Normal
    voxel_color_u = image_load(voxelFragTex_color, gl_VertexID).x
    voxel_normal_u = image_load(voxelFragTex_normal, gl_VertexID).x
    
    # 8. Store to Brick Pools
    # convRGBA8ToVec4 converts packed uint to vec4, scaled by 1/255.0
    color_val = conv_rgba8_to_vec4(voxel_color_u) / 255.0
    normal_val = conv_rgba8_to_vec4(voxel_normal_u) / 255.0
    irradiance_val = vec4(0.0, 0.0, 0.0, 1.0)
    
    image_store(brickPool_color, target_coord, color_val)
    image_store(brickPool_normal, target_coord, normal_val)
    image_store(brickPool_irradiance, target_coord, irradiance_val)
```

**Edge Cases and Pitfalls:**
*   **Memory Barriers:** The `memoryBarrier()` after loading `nodeColorU` is critical. If the node pool is written by a previous pass (e.g., node allocation) and read here without synchronization, race conditions may occur, leading to incorrect brick addresses [S1, unofficial].
*   **Precision:** The use of `r32ui` for positions and colors avoids floating-point precision loss during the initial packing/unpacking stages [S1, unofficial].
*   **Boundary Conditions:** The `traverseOctree_posOut` function must handle voxels that fall outside the defined octree bounds or correspond to empty nodes. The source does not specify error handling for invalid nodes, implying the input fragment list is pre-filtered or the traversal guarantees a valid leaf address for all valid inputs [S1, unofficial].
*   **Irradiance Initialization:** Irradiance is hardcoded to zero. This indicates that this step only handles direct lighting data (color/normal). Indirect lighting (irradiance) must be computed in a subsequent pass [S1, unofficial].

## Subtopics

### [Sparse Voxel Octree Construction from Density Field](18778-sparse-voxel-octree-construction-from-density-field.md)
This method converts a 3D density field into a compact sparse voxel octree (SVO) structure using an OpenCL-based pipeline. It is distinguished by its explicit handling of OpenCL/OpenGL interop limitations, opting for a download-reupload strategy to avoid rendering thread blocking.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxel fragments, $O(N)$, where $N$ is the number of active voxels. Each fragment requires one tree traversal and a constant number of memory accesses.
*   **Memory Complexity:** The memory footprint is determined by the size of the `brickPool` textures, which scale with the number of leaf nodes and the brick resolution (typically $2 \times 2 \times 2$ or similar small brick sizes).
*   **Performance:** The method is designed for GPU execution via compute shaders, leveraging parallelism to process thousands of fragments simultaneously. The use of `uimageBuffer` and `image3D` allows for efficient, unstructured memory access patterns typical of sparse data structures [S1, unofficial].
*   **Strengths:** It decouples the topological construction of the octree from the data population, allowing for flexible data formats in the leaf nodes (color, normal, irradiance) [S1, unofficial].
*   **Limitations:** It assumes a valid SVO topology has already been constructed. It does not handle the creation of internal nodes or the management of the octree structure itself [S1, unofficial].

## Background
This method originates from the Voxel Cone Tracing implementation by league1991, which demonstrates a real-time global illumination technique using sparse voxel octrees [S1, unofficial]. The specific leaf population step is part of the broader SVO construction pipeline described in the associated GitHub repository [S1, unofficial].

## Sources
[S1] league1991/Voxel-Cone-Tracing, https://github.com/league1991/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
