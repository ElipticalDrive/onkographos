# Greedy Meshing for Voxel Faces

Also called: Greedy Meshing for Voxels, Greedy Meshing with Face Attributes, Greedy meshing for complex voxel data, Quad merging, Voxel face greedy meshing, Voxel face merging

## Overview
Greedy Meshing for Voxel Faces is a mesh simplification technique that reduces the number of triangles and draw calls in voxel rendering by merging adjacent coplanar voxel faces into larger rectangular quads [S1, S5, unofficial]. Unlike standard greedy meshing, which merges voxels based solely on existence or a single block type, this method compares voxel faces based on multiple attributes (such as material type, transparency, and lighting) to determine mergeability [S1, S5, unofficial]. It builds on [Greedy Meshing](3519-greedy-meshing.md) by extending the 2D slice processing to handle face-level attributes, ensuring that only faces with identical attributes are merged [S1, S5, unofficial].

## How it works
The algorithm processes a voxel chunk by iterating through six face directions (Top, Bottom, Left, Right, Front, Back). For each direction, it extracts a 2D cross-section of visible faces, generates a mask of mergeable regions, and creates quads for maximal rectangles of identical faces [S1, S5, unofficial].

### Inputs and Outputs
*   **Input**: A 3D voxel grid `voxels[CHUNK_WIDTH][CHUNK_HEIGHT][CHUNK_DEPTH]` where each voxel is a `VoxelFace` object. A `VOXEL_SIZE` scalar for world coordinate scaling.
*   **Output**: A list of quads, each defined by four vertices (position, color/UV), and a list of triangle indices.

### Data Structures
*   **VoxelFace**: Encapsulates face attributes.
    *   `transparent` (boolean): If true, the face is culled and not rendered [S1, S5, unofficial].
    *   `type` (int): The material or block type identifier [S1, S5, unofficial].
    *   `side` (int): The face normal direction (0=SOUTH, 1=NORTH, 2=EAST, 3=WEST, 4=TOP, 5=BOTTOM) [S1, S5, unofficial].
    *   `equals(VoxelFace other)`: Returns true if `this.transparent == other.transparent` and `this.type == other.type` [S1, S5, unofficial]. Note: Some implementations also require corner lighting to be identical to preserve ambient occlusion [S2, unofficial].
*   **Mask**: A 1D array `mask[CHUNK_WIDTH * CHUNK_HEIGHT]` (or 2D equivalent) storing `VoxelFace` references or null. It tracks which faces have been merged into existing quads [S1, S5, unofficial].
*   **Working Arrays**:
    *   `x[3]`: Current position in the chunk (x, y, z) [S1, S5, unofficial].
    *   `q[3]`: Direction vector for neighbor checks (one component is 1, others 0) [S1, S5, unofficial].
    *   `du[3]`, `dv[3]`: Delta vectors for quad width and height [S1, S5, unofficial].

### Algorithm Steps

1.  **Initialization**:
    *   Initialize `x = [0, 0, 0]`, `q = [0, 0, 0]`, `du = [0, 0, 0]`, `dv = [0, 0, 0]` [S1, S5, unofficial].
    *   Create `mask` array of size `CHUNK_WIDTH * CHUNK_HEIGHT` (for the 2D slice) initialized to null [S1, S5, unofficial].
    *   Initialize empty vertex and triangle lists for the chunk [S3, unofficial].

2.  **Direction Loop**:
    *   Iterate through six face directions. This is often implemented using a boolean toggle loop to handle back-face and front-face culling or orientation:
        ```python
        backFace = True
        b = False
        while b != backFace:
            # Process current direction
            # ...
            backFace = backFace and b
            b = not b
        ```
        This loop runs twice, covering all six axes (3 dimensions × 2 directions) [S1, S5, unofficial].

3.  **Axis Processing**:
    *   For each direction, iterate `d` from 0 to 2 (representing X, Y, Z axes) [S1, S5, unofficial].
    *   Calculate perpendicular axes: `u = (d + 1) % 3`, `v = (d + 2) % 3` [S1, S5, unofficial].
    *   Set `q[d] = 1` (direction of sweep) and `q[u] = q[v] = 0` [S1, S5, unofficial].
    *   Determine the `side` constant based on `d` and `backFace` (e.g., if `d=0` and `backFace` is true, `side = WEST`) [S1, S5, unofficial].
    *   Reset `x[0] = x[1] = x[2] = 0` [S1, S5, unofficial].

4.  **Sweep and Mask Generation**:
    *   Sweep along dimension `d` from `x[d] = -1` to `x[d] < CHUNK_SIZE[d]` [S1, S5, unofficial].
    *   For each slice `x[d]`:
        *   Reset `n = 0` (index into mask) [S1, S5, unofficial].
        *   Iterate `x[v]` from 0 to `CHUNK_SIZE[v]-1` and `x[u]` from 0 to `CHUNK_SIZE[u]-1` [S1, S5, unofficial].
        *   Get `voxelFace` at `x` (if `x[d] >= 0`) and `voxelFace1` at neighbor `x + q` (if `x[d] < CHUNK_SIZE[d] - 1`) [S1, S5, unofficial].
        *   If both faces exist and `voxelFace.equals(voxelFace1)` is true, set `mask[n] = null` (indicating these faces are merged/hidden) [S1, S5, unofficial].
        *   Otherwise, set `mask[n] = voxelFace1` if `backFace` is true, else `mask[n] = voxelFace` [S1, S5, unofficial].
        *   Increment `n` [S1, S5, unofficial].
    *   Increment `x[d]` [S1, S5, unofficial].

5.  **Quad Generation**:
    *   Reset `n = 0` [S1, S5, unofficial].
    *   Iterate `j` (corresponding to `v`) from 0 to `CHUNK_SIZE[v]-1` and `i` (corresponding to `u`) from 0 to `CHUNK_SIZE[u]-1` [S1, S5, unofficial].
    *   If `mask[n]` is not null:
        *   **Calculate Width `w`**: Start `w = 1`. While `i + w < CHUNK_SIZE[u]` and `mask[n + w]` is not null and `mask[n + w].equals(mask[n])`, increment `w` [S1, S5, unofficial].
        *   **Calculate Height `h`**: Start `h = 1`. Loop while `j + h < CHUNK_SIZE[v]`:
            *   Loop `k` from 0 to `w-1`.
            *   Check if `mask[n + k + h * CHUNK_SIZE[u]]` is null or does not equal `mask[n]`.
            *   If so, set `done = true` and break inner loop [S1, S5, unofficial].
            *   If `done` is true, break height loop [S1, S5, unofficial].
        *   **Create Quad**:
            *   Set `x[u] = i`, `x[v] = j` [S1, S5, unofficial].
            *   Set `du[u] = w`, `du[v] = du[d] = 0` [S1, S5, unofficial].
            *   Set `dv[v] = h`, `dv[u] = dv[d] = 0` [S1, S5, unofficial].
            *   If `mask[n].transparent` is false, generate vertices for the quad using `x`, `du`, `dv`, and `VOXEL_SIZE` [S1, S5, unofficial].
            *   Vertices are typically:
                1.  `x`
                2.  `x + du`
                3.  `x + du + dv`
                4.  `x + dv`
            *   Add triangle indices (e.g., 0,1,2 and 0,2,3) to the triangle list, adding an offset if combining results from multiple directions [S3, unofficial].
            *   **Zero Out Mask**: Loop `l` from 0 to `h-1` and `k` from 0 to `w-1`, set `mask[n + k + l * CHUNK_SIZE[u]] = null` to mark as processed [S1, S5, unofficial].
            *   Increment `i` by `w` and `n` by `w * h` (or appropriate stride) to skip processed region [S1, S5, unofficial].

6.  **Combine Results**:
    *   After processing all six directions, combine the vertex and triangle lists. Ensure triangle indices are offset by the cumulative vertex count from previous directions [S3, unofficial].

### Parameters
*   `VOXEL_SIZE`: Size of a single voxel in world units. Default is 1. Used to scale vertex coordinates [S1, S5, unofficial].
*   `CHUNK_WIDTH`, `CHUNK_HEIGHT`, `CHUNK_DEPTH`: Dimensions of the voxel chunk. Typically 16x16x16 or similar powers of two [S1, S5, unofficial].

### Edge Cases and Pitfalls
*   **Transparency**: Transparent faces must be skipped during meshing to avoid rendering culled geometry. The algorithm checks `mask[n].transparent` before adding a quad [S1, S5, unofficial].
*   **Lighting/AO**: A face whose four corners have different lighting values must not be merged with neighbors to preserve correct ambient occlusion shading. This requires the `equals` method to compare corner lighting attributes [S2, unofficial].
*   **Boundary Conditions**: If no adjacent chunk exists in a direction (e.g., UP, DOWN), the corresponding block array is created with size 0 to prevent null access [S3, unofficial]. If no chunk exists above the top layer, the top face generation is skipped entirely [S3, unofficial].
*   **Index Offsets**: Each directional job starts with an empty vertex list, so triangle indices within each job are relative to 0. When combining, an additional offset must be added to triangle indices to account for the accumulated vertex count from previous directions [S3, unofficial].
*   **Performance**: This method requires more calculations per chunk on initial world build or when blocks are added/removed, making it expensive for dynamic updates [S4, unofficial].

## Subtopics

### [Greedy Meshing with Block Type Masking](14703-greedy-meshing-with-block-type-masking.md)
Extends this method by ensuring that adjacent voxel faces are merged into a single quad only if they share the same block type (e.g., material, texture) and orientation. Unlike standard greedy meshing, which merges all coplanar faces regardless of material, this method preserves visual distinctions between different block types (e.g., keeping grass and dirt transitions separate) while still maximizing quad size within homogeneous regions.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel face is visited a constant number of times during the sweep and mask generation. However, the constant factor is higher than simple face culling due to the mask generation and quad expansion logic [S4, unofficial].
*   **Memory Complexity**: $O(N)$ for the mask array and vertex/triangle buffers. The mask array size is proportional to the surface area of the chunk in each direction [S1, S5, unofficial].
*   **Measured Performance**: On a flat chunk of ground, the number of quads is reduced from 2,048 to 2 [S2, unofficial]. This drastically reduces draw calls for large flat surfaces [S2, unofficial].
*   **Strengths**: Significantly reduces vertex and triangle count for large flat surfaces, improving rendering performance by reducing draw calls [S2, S4, unofficial]. Allows for attribute-based merging (type, transparency, lighting) [S1, S5, unofficial].
*   **Limitations**: More expensive calculation per chunk on initial world build or when the player removes/adds a block [S4, unofficial]. Performance may not improve or may even slow down due to texture tiling in the shader and the cost of re-generating chunks for lightmap changes [S4, unofficial].
*   **When to Use**: Ideal for static or semi-static voxel worlds with large flat areas. Not recommended for highly dynamic scenes with frequent block changes unless combined with incremental meshing techniques.

## Background
Greedy Meshing for Voxel Faces was developed by Robert O'Leary, based on the work of Mikola Lysenko [S1, unofficial]. It was implemented in the JMonkeyEngine and later adapted for other engines like 'mira' (in Rust) and Unity [S1, S2, S3, unofficial]. The technique is a variation of the original greedy meshing algorithm described by Lysenko in 2012, extended to handle complex voxel data with multiple face attributes [S1, unofficial]. It is widely used in voxel engines to optimize rendering performance by reducing the number of draw calls [S4, unofficial].

## Sources
[S1] roboleary/GreedyMesh, https://github.com/roboleary/GreedyMesh, fetched 2026-10-26 (unofficial)
[S2] Voxel faces that lie side by side are joined into large quads, https://github.com/mcdearman/mira/pull/82, fetched 2026-10-26 (unofficial)
[S3] BloodyFish/UnityVoxelEngine, https://github.com/BloodyFish/UnityVoxelEngine, fetched 2026-10-27 (unofficial)
[S4] How can I improve rendering speeds of a Voxel/Minecraft type game?, https://gamedev.stackexchange.com/questions/22664/how-can-i-improve-rendering-speeds-of-a-voxel-minecraft-type-game, fetched 2026-10-27 (unofficial)
[S5] Greedy meshing for complex voxel data, https://hub.jmonkeyengine.org/t/greedy-meshing-for-complex-voxel-data/28017, fetched 2026-10-27 (unofficial)
