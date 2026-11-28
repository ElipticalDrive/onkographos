# Sparse Voxel List and Indirect Commands

Also called: Sparse Voxel Occupancy List, Indirect Command Buffer for Voxelization

## Overview
This method maintains a compact list of occupied voxel coordinates during GPU-based voxelization to optimize subsequent rendering and data processing passes. It solves the inefficiency of iterating over empty space in dense voxel textures by restricting compute shader execution and indirect draw calls to only those voxels that contain geometry. It differs from its parent method, [Atomic Color Averaging in Voxelization](3529-atomic-color-averaging-in-voxelization.md), by decoupling the accumulation of color data from the tracking of spatial occupancy, allowing for specialized optimization of the final data unpacking and rendering phases.

## How it works
**Inputs and Outputs:**
- Input: The voxel texture populated by the atomic accumulation pass (typically `RGBA8` or similar integer format).
- Output: A sparse voxel list (array of coordinates) and an indirect command buffer.

**Data Structures:**
1. **Sparse Voxel List:** A dynamic array or buffer storing the 3D coordinates (e.g., `uvec3`) of all voxels where the occupancy count is non-zero.
2. **Indirect Command Buffer:** A buffer containing `DrawIndirect` or `DispatchIndirect` structures, one per occupied voxel or group of voxels, specifying the number of threads and invocation counts.

**Steps:**
1. **Occupancy Tracking:** During the voxelization pass (described in [Atomic Color Averaging in Voxelization](3529-atomic-color-averaging-in-voxelization.md)), alongside the atomic update of the voxel color and count, a secondary mechanism records the coordinate of any voxel where the count transitions from 0 to 1. This can be done using atomic append operations or a pre-allocated buffer with atomic indices.
2. **List Population:** After voxelization, the sparse voxel list contains all occupied voxel coordinates. This list is used to identify the subset of the texture that contains valid data.
3. **Indirect Command Buffer Construction:** For each entry in the sparse voxel list, an indirect command structure is generated. This structure defines the parameters for a compute dispatch or draw call that targets only that specific voxel or a small tile of voxels.
4. **Optimized Rendering/Unpacking:**
   - **Rendering:** Instead of rendering the entire voxel volume, the renderer issues indirect draw calls using the command buffer. This restricts rasterization or ray-casting to only occupied regions.
   - **Unpacking:** To convert the internal voxel representation (e.g., packed `uint` with RGB and count) into a standard `RGBA8` texture for display or further processing, a compute shader iterates only over the indices in the sparse voxel list, unpacking the data and writing to the final texture.

**Formulas:**
The occupancy check is implicit: a voxel is included in the list if its count field $C > 0$. The indirect command for a voxel at coordinate $\mathbf{v}$ specifies a dispatch size of $(1, 1, 1)$ or a tile size, targeting $\mathbf{v}$.

**Parameters:**
- **Batch Size:** The number of voxels grouped into a single indirect command. Not given in the sources.
- **List Capacity:** Must be at least the number of fragments processed. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Empty Scene:** If no voxels are occupied, the list is empty, and no indirect commands are issued. The renderer must handle this gracefully.
- **List Overflow:** If the number of occupied voxels exceeds the pre-allocated buffer size for the sparse list, data loss occurs. Dynamic resizing or double-buffering is required. Not given in the sources.
- **Precision:** Coordinates must be stored with sufficient precision to map back to the voxel grid. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume voxel_data is a 3D image buffer populated by atomic operations
# Assume sparse_list is a dynamic array of uvec3
# Assume indirect_cmds is a dynamic array of DrawIndirectStruct

# Step 1: During voxelization (conceptual, integrated with atomic pass)
# When atomicCompareSwap succeeds (count goes from 0 to 1):
#   atomicAdd(sparse_list_index, 1) -> idx
#   sparse_list[idx] = voxel_coord

# Step 2: Construct Indirect Command Buffer
for i in range(0, sparse_list.size()):
    coord = sparse_list[i]
    # Define a dispatch for this voxel
    cmd = DrawIndirectStruct()
    cmd.vertexCount = 1 # Or tile size
    cmd.instanceCount = 1
    cmd.firstVertex = 0
    cmd.firstInstance = 0
    # Store command, possibly with metadata for the shader to know which coord to process
    indirect_cmds.append(cmd)

# Step 3: Unpack to RGBA8 using sparse list
output_texture = Image2D(width, height, format=RGBA8)
for i in range(0, sparse_list.size()):
    coord = sparse_list[i]
    packed = imageLoad(voxel_data, coord)
    # Unpack logic as defined in Atomic Color Averaging
    # avg = vec4((avg.rgb * count + color.rgb) / float(count + 1), float(count + 1) / 255.0f);
    # Here we reverse the packing or read the final state
    color = unpack_rgba8(packed)
    imageStore(output_texture, coord.xy, color)

# Step 4: Render using indirect commands
# glDrawArraysIndirect or similar API call using indirect_cmds.buffer
```

## Subtopics

### [Voxel List Construction Using Histogram Pyramids](20207-voxel-list-construction-using-histogram-pyramids.md)
This method initializes a sparse voxel list on the GPU by mapping linear indices of non-zero voxels back to their spatial coordinates. It solves the problem of parallel coordinate reconstruction by using a histogram pyramid to perform a binary-search-like traversal. It differs from atomic accumulation methods by avoiding race conditions and sorting overhead during the initial population phase, instead relying on a deterministic traversal of a precomputed reduction structure.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of occupied voxels for list construction and unpacking. The indirect command dispatch is $O(1)$ per occupied voxel (or tile).
- **Memory Complexity:** $O(N)$ for the sparse list and indirect command buffer. This is significantly less than the full voxel texture size if the scene is sparse.
- **Performance:** Reduces thread usage in compute shaders and draw calls in rendering by skipping empty space. Measured performance depends on the sparsity of the scene. Not given in the sources.
- **Strengths:** Efficient rendering of sparse volumes; reduced memory bandwidth for unpacking.
- **Limitations:** Overhead of maintaining the list and indirect commands; not beneficial for dense volumes.
- **When to use:** When the voxelized scene is sparse and subsequent passes (like conetracing) can be optimized to skip empty space.
- **When not to use:** When the volume is dense, or the overhead of managing the list exceeds the savings from skipping empty voxels.

## Background
This method is part of a voxel cone tracing renderer implementation, specifically optimizing the transition from voxelization to rendering. It is described in a graphics programming portfolio by Otávio Peixoto.

## Sources
[S1] Voxel Cone Tracing Renderer - Graphics Programming Portfolio, https://otaviopeixoto1.github.io/portfolio/vctgi/, fetched 2026-10-27 (unofficial)
