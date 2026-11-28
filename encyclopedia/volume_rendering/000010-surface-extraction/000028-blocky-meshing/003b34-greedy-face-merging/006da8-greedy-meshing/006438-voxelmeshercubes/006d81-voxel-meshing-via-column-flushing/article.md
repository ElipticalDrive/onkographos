# Voxel Meshing via Column Flushing

Also called: Compute Shader Voxel Meshing, Voxel Meshing v2

## Overview
Voxel Meshing via Column Flushing is a GPU-accelerated surface extraction method that converts a sparse voxel grid into a triangle mesh by sweeping along one axis and merging contiguous filled voxels into quads. It differs from [VoxelMesherCubes](19075-voxelmeshercubes.md) by generating a single quad per merged column segment rather than individual cube faces, significantly reducing vertex and index counts for blocky geometry. It builds on the greedy face merging concept but implements it via column flushing in compute shaders, avoiding the need for complex neighbor lookups or post-processing sorting.

## How it works
The method processes the voxel grid using compute shaders, sweeping along the Y-axis (or other axes) to detect boundaries between filled and empty columns. It maintains state for the start position and color index of active column segments. When a boundary is detected (transition from filled to empty or vice versa, or change in color index), the current segment is "flushed" as a quad. The method handles six face directions (-X, +X, -Y, +Y, -Z, +Z) by processing different sweep directions or by checking neighbors within the sweep.

### Inputs and Outputs
**Inputs:**
- `voxels`: A buffer of `VoxelData` structs, storing 4 voxel color indices packed into a 32-bit integer (8 bits per voxel).
- `model_size`: A `vec3` specifying the dimensions of the voxel grid $(W, H, D)$.

**Outputs:**
- `vertices`: A buffer of `Vertex` structs, storing positions.
- `indices`: A buffer of `uint` indices, forming triangles.
- `packedData`: A buffer of `uint` values, storing color index and face normal index per face.
- `indirect_command`: A `DrawElementsIndirectCommand` struct, with `count` updated atomically to track the number of triangles generated.

### Data Structures
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct VoxelData {
    uint four_voxels; // 4 voxels, 8 bits each, packed into 32 bits
};

struct Vertex {
    float16_t x, y, z; // Half-precision float positions
};

struct DrawElementsIndirectCommand {
    uint count;         // Number of indices (6 per quad)
    uint instanceCount; // Always 1
    uint firstIndex;    // Always 0
    uint baseVertex;    // Always 0
    uint baseInstance;  // Always 0
};
```

### Numbered Steps
1. **Initialization**: For each thread (representing an $(x, z)$ coordinate in the grid if sweeping Y), initialize state variables: `prev_col_neg = 0`, `start_neg = vec3(-1)`, `prev_col_pos = 0`, `start_pos = vec3(-1)`. These track the previous column's color index and the start position of the current segment for the negative and positive neighbors in the sweep direction.
2. **Sweep Loop**: Iterate through the Y-axis from $0$ to $H-1$.
   a. **Get Current Voxel**: Calculate the color index `curr` for the voxel at $(x, y, z)$. If the voxel is empty (`curr == 0`), set `col_neg = 0` and `col_pos = 0`. If filled, check neighbors in the X-direction (for Y-sweep) or Z-direction (for other sweeps) to determine `col_neg` and `col_pos`. Specifically, `col_neg` is `curr` if the neighbor in the negative direction is empty, otherwise `0`. `col_pos` is `curr` if the neighbor in the positive direction is empty, otherwise `0`.
   b. **Process Faces**: For each face direction (e.g., -Y and +Y when sweeping Y):
      i. **Boundary Detection**: If the current column's color index (`col_neg` or `col_pos`) differs from the previous column's (`prev_col_neg` or `prev_col_pos`):
         - If the previous column was filled (`prev_col != 0`), flush the face segment from `start` to `current_pos - vec3(0,1,0)` (or appropriate offset) with color `prev_col` and normal index.
         - If the current column is filled (`col != 0`), update `start` to `current_pos`.
      ii. **Update State**: Set `prev_col_neg = col_neg` and `prev_col_pos = col_pos`.
   c. **End of Sweep**: After the loop, if `prev_col_neg` or `prev_col_pos` is non-zero, flush the final segment from `start` to `end_pos` (where `end_pos` is at $y = H-1$).
3. **Flush Face**: When flushing a face:
   a. Calculate the face length as the difference in the sweep axis between `end` and `start`.
   b. Atomically add 6 to `indirect_command.count` to reserve space for 2 triangles (6 indices).
   c. Calculate `base_idx` as the new count value.
   d. Calculate `base_vtx` as `(base_idx / 6) * 4` (4 vertices per quad).
   e. Pack data: `packed_val = (col & 255) | (normal_idx << 8)`. Store in `packedData[base_idx / 6]`.
   f. Generate 4 vertices for the quad based on the face normal and positions.
   g. Generate 6 indices for the 2 triangles.

### Formulas
- **Voxel Index**: $idx = x + y \cdot W + z \cdot W \cdot H$
- **Packed Voxel Data**: $voxel\_color = (voxels[idx >> 2].four\_voxels >> (8 \cdot (idx \& 3))) \& 255$
- **Face Length**: $L = end_{sweep\_axis} - start_{sweep\_axis}$
- **Packed Data**: $packed = (color\_index \& 255) | (normal\_index \ll 8)$
- **Base Vertex Index**: $base\_vtx = (base\_idx / 6) \cdot 4$

### Parameters
- `model_size`: $(W, H, D)$, the dimensions of the voxel grid.
- `local_size`: $(16, 16, 1)$ for the compute shader.

### Edge Cases and Pitfalls
- **Boundary Handling**: Faces at the end of a column (e.g., $y = H-1$) must be explicitly flushed if a starting position is active. Failure to do so results in the last segment of the face being lost.
- **Empty Voxel Handling**: When an empty voxel is encountered, any active face segment must be flushed immediately to prevent merging with subsequent filled voxels.
- **Precision**: Using `float16_t` for vertex positions may lead to precision issues for large grids. Consider using `float` if precision is critical.
- **Atomic Contention**: High contention on `indirect_command.count` can reduce performance. Ensure sufficient buffer sizes for vertices, indices, and packed data.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Y-sweep (slicing_y.comp)
# Assumes global invocation id (ix, iz) maps to a column in Y

def voxel_meshing_column_flushing(voxels, model_size):
    W, H, D = model_size
    vertices = []
    indices = []
    packed_data = []
    indirect_count = 0

    def get_voxel(x, y, z):
        idx = x + y * W + z * W * H
        packed = voxels[idx >> 2].four_voxels
        return (packed >> (8 * (idx & 3))) & 255

    def create_face(start, end, col, normal_idx):
        nonlocal indirect_count
        face_length = end.y - start.y
        base_idx = indirect_count
        indirect_count += 6
        base_vtx = (base_idx // 6) * 4
        
        packed_val = (col & 255) | (normal_idx << 8)
        packed_data.append(packed_val)
        
        x_offset = 1.0 if normal_idx == 1 else 0.0
        base = start + vec3(x_offset, 0.0, 0.0)
        
        pos0 = base
        pos1 = base + vec3(0.0, 1.0 + face_length, 0.0)
        pos2 = base + vec3(0.0, 1.0 + face_length, 1.0)
        pos3 = base + vec3(0.0, 0.0, 1.0)
        
        vertices.extend([pos0, pos1, pos2, pos3])
        
        if normal_idx == 1:
            indices.extend([base_vtx, base_vtx + 2, base_vtx + 1,
                            base_vtx, base_vtx + 1, base_vtx + 2])
        else:
            # Assuming other normals have similar winding or specific offsets
            # This part needs to be adapted for other normal indices
            pass

    for ix in range(W):
        for iz in range(D):
            prev_col_neg = 0
            start_neg = vec3(-1)
            prev_col_pos = 0
            start_pos = vec3(-1)
            
            for iy in range(H):
                curr = get_voxel(ix, iy, iz)
                
                col_neg = 0
                col_pos = 0
                
                if curr != 0:
                    neigh_col_neg = get_voxel(ix - 1, iy, iz) if ix > 0 else 0
                    neigh_col_pos = get_voxel(ix + 1, iy, iz) if ix < W - 1 else 0
                    
                    if neigh_col_neg == 0:
                        col_neg = curr
                    if neigh_col_pos == 0:
                        col_pos = curr
                
                current_pos = vec3(ix, iy, iz)
                
                # Process -Y face
                if col_neg != prev_col_neg:
                    if prev_col_neg != 0:
                        create_face(start_neg, current_pos - vec3(0, 1, 0), prev_col_neg, 0)
                    if col_neg != 0:
                        start_neg = current_pos
                prev_col_neg = col_neg
                
                # Process +Y face
                if col_pos != prev_col_pos:
                    if prev_col_pos != 0:
                        create_face(start_pos, current_pos - vec3(0, 1, 0), prev_col_pos, 1)
                    if col_pos != 0:
                        start_pos = current_pos
                prev_col_pos = col_pos
            
            # Flush remaining faces at end of column
            end_pos = vec3(ix, H - 1, iz)
            if prev_col_neg != 0:
                create_face(start_neg, end_pos, prev_col_neg, 0)
            if prev_col_pos != 0:
                create_face(start_pos, end_pos, prev_col_pos, 1)

    return vertices, indices, packed_data, indirect_count
```

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of voxels, as each voxel is processed once per relevant sweep direction.
- **Memory Complexity**: $O(F)$, where $F$ is the number of faces generated. The output buffers scale with the mesh complexity.
- **Performance**: Highly parallelizable on GPU due to independent column processing. Performance is limited by memory bandwidth for voxel data access and atomic operations for index generation.
- **Strengths**: Efficient for sparse voxel grids, reduces vertex count compared to per-voxel meshing, simple implementation in compute shaders.
- **Limitations**: Generates blocky meshes, not suitable for smooth surfaces. Precision issues with `float16_t` for large grids.

## Background
Voxel Meshing via Column Flushing is an unofficial method developed by knosvoxel, as seen in the [knosvoxel/voxel-meshing-v2](https://github.com/knosvoxel/voxel-meshing-v2) repository. It is a variant of greedy meshing optimized for GPU compute shaders, focusing on column-based merging to reduce geometry complexity.

## Sources
[S1] knosvoxel/voxel-meshing-v2, https://github.com/knosvoxel/voxel-meshing-v2, fetched 2026-10-26 (unofficial)
