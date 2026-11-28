# SVO Node Visualization via Instanced Cubes

Also called: Node Cube Rendering, SVO Visualizer

## Overview
A compute-to-draw pipeline that converts Sparse Voxel Octree (SVO) nodes into instanced cubes for visualization [S1, unofficial]. It differs from [Cube Rasterization via SSBO](2267-cube-rasterization-via-ssbo.md) by using a compute shader to explicitly generate instance data (positions and sizes) from SVO node metadata, rather than relying on a pre-populated SSBO of voxel data [S1, unofficial]. This allows for efficient visualization of the tree structure itself, including empty nodes, by rendering only the active or relevant nodes at a specific level [S1, unofficial].

## How it works
**Inputs:**
- Current SVO level index (`CURLEVEL`) [S1, unofficial].
- SVO node pool (`mNodesPool`) accessed via SRV [S1, unofficial].
- Constant buffers: `mCBInfo`, `mCBAttri` (containing `maxSide` and `box.minP`/`maxP`), `mCBBrickInfo` [S1, unofficial].
- Atomic counter (`visualIndex`) to track generated instances [S1, unofficial].
- Unordered Access View (UAV) for the visual package buffer (`mVisualPool`) [S1, unofficial].

**Outputs:**
- Instanced cube geometry rendered via indexed draw call [S1, unofficial].
- `mVisualInstanceBuf` containing instance indices [S1, unofficial].

**Data Structures:**
1.  **Vertex Buffer (`mVisualVB`)**: Immutable, 8 vertices, each `XMFLOAT3` (3 floats, 12 bytes). Total size: 96 bytes. Defines a unit cube centered at origin [S1, unofficial].
2.  **Index Buffer (`mVisualIB`)**: Immutable, 36 indices, each `u32` (4 bytes). Total size: 144 bytes. Defines 6 faces (2 triangles each). Indices layout:
    -   Front: 0-5
    -   Back: 6-11
    -   Right: 12-17
    -   Left: 18-23
    -   Top: 24-29
    -   Bottom: 30-35 [S1, unofficial].
3.  **Visual Package Buffer (`mVisualPool`)**: UAV/SRV buffer containing `mNumNodePerLevel[level]` elements. Each element likely contains instance position and size data [S1, unofficial].
4.  **Instance Buffer (`mVisualInstanceBuf`)**: Dynamic buffer sized to `sum` (the final atomic counter value), storing indices for the draw call [S1, unofficial].

**Steps:**
1.  **Calculate Cube Size**: For each node at `CURLEVEL`, calculate the cube size based on the SVO level and bounding box extent [S1, unofficial].
    $$ res = 2^{level} $$
    $$ maxSide = \max(box.extent(0), box.extent(1), box.extent(2)) $$
    $$ size = \frac{maxSide}{res} \times 0.5 $$
    Define 8 cube vertices at $\pm size$ relative to the node's center [S1, unofficial].
2.  **Dispatch Compute Shader (`mVisualCS`)**:
    -   Bind constant buffers (`mCBInfo`, `mCBAttri`, `mCBBrickInfo`) [S1, unofficial].
    -   Bind SRV for `mNodesPool` [S1, unofficial].
    -   Bind UAVs for `visualIndex` (atomic counter), `mVisualPool`, and `mBricksPool` [S1, unofficial].
    -   Dispatch with thread group size equal to $2^{level}$ in each dimension ($ures = 2^{level}$) [S1, unofficial].
    -   The compute shader populates `mVisualPool` with instance data and increments the `visualIndex` atomic counter for each valid node [S1, unofficial].
3.  **Read Atomic Counter**: Read the sum from `visualIndex` to determine the total number of instances (`sum`) [S1, unofficial].
4.  **Create Instance Buffer**: Allocate `mVisualInstanceBuf` with `sum` indices [S1, unofficial].
5.  **Draw Call**:
    -   Bind vertex buffer `mVisualVB` (stride: sizeof(XMFLOAT3)) [S1, unofficial].
    -   Bind instance buffer `mVisualInstanceBuf` (stride: sizeof(u32)) [S1, unofficial].
    -   Bind index buffer `mVisualIB` (format: R32_UINT) [S1, unofficial].
    -   Perform indexed instanced draw: 36 indices, `mNumNodePerLevel[CURLEVEL]` instances [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Compute Shader Dispatch (mVisualCS)
# Inputs: mNodesPool (SRV), mCBInfo, mCBAttri, mCBBrickInfo
# Outputs: visualIndex (Atomic Counter), mVisualPool (UAV)

def dispatch_visual_cs(level, ures):
    num_threads = ures * ures * ures
    # Bind resources
    bind_constant_buffers(mCBInfo, mCBAttri, mCBBrickInfo)
    bind_srv(mNodesPool)
    bind_uav(visual_index_counter)
    bind_uav(mVisualPool)
    bind_uav(mBricksPool)
    
    # Dispatch compute shader
    # The compute shader iterates over nodes at 'level'
    # For each valid node:
    #   1. Calculate size = (maxSide / 2^level) * 0.5
    #   2. Write instance data (position, size) to mVisualPool
    #   3. Increment visual_index_counter atomically
    dispatch_compute(num_threads)

# Step 2: Read Instance Count
def get_instance_count():
    return visual_index_counter.get_value()

# Step 3: Setup Draw Call
def draw_instanced_cubes(level, instance_count):
    # Allocate instance buffer based on actual count
    sum = get_instance_count()
    mVisualInstanceBuf = create_buffer(size=sum * sizeof(u32))
    
    # Bind buffers
    bind_vertex_buffer(mVisualVB, stride=sizeof(XMFLOAT3))
    bind_instance_buffer(mVisualInstanceBuf, stride=sizeof(u32))
    bind_index_buffer(mVisualIB, format=R32_UINT)
    
    # Draw indexed instanced
    # 36 indices per cube, 'instance_count' instances
    draw_indexed_instanced(index_count=36, instance_count=instance_count)
```

**Parameters:**
-   `CURLEVEL`: Current SVO level index used for visualization dispatch and draw call instance count [S1, unofficial].

**Edge Cases:**
-   Precision: Floating-point calculations for `size` may suffer from precision issues at very high levels [G].
-   Boundaries: Ensure `mVisualPool` is large enough to hold all nodes at `CURLEVEL` [G].
-   Degenerate Input: If `mNumNodePerLevel[level]` is 0, no draw call is needed [G].

## Operating characteristics
-   **Time Complexity**: $O(N)$ where $N$ is the number of nodes at `CURLEVEL` [S1, unofficial].
-   **Memory Complexity**: $O(N)$ for `mVisualPool` and `mVisualInstanceBuf` [S1, unofficial].
-   **Performance**: Depends on the number of nodes and the efficiency of the compute shader [S1, unofficial].
-   **Strengths**: Efficient visualization of SVO structure, including empty nodes [S1, unofficial].
-   **Limitations**: Requires compute shader support [S1, unofficial].
-   **When to Use**: Debugging SVO structure, visualizing tree levels [S1, unofficial].
-   **When Not to Use**: Final rendering of dense volumes [S1, unofficial].

## Background
Originated in the SVODemo project by Throbbing [S1, unofficial]. Used for visualizing Sparse Voxel Octree structures in real-time applications [S1, unofficial].

## Sources
[S1] Throbbing/SVODemo, https://github.com/Throbbing/SVODemo, fetched 2026-10-26 (unofficial)
