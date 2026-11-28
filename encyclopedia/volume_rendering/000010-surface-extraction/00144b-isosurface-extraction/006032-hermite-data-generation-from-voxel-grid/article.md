# Hermite Data Generation from Voxel Grid

Also called: Marching Cubes Hermite Data Construction, Voxelized Surface Normal Sampling

## Overview
This method generates Hermite data (surface intersection positions and normals) for a voxel grid by iterating over active cells, determining the Marching Cubes configuration via density thresholds, and sampling intersections along specified edges [S1, unofficial]. It differs from standard Marching Cubes by decoupling the topological edge intersection calculation from the final mesh triangulation, storing the geometric primitives (points and normals) in a structured array for downstream processing such as Surface Nets or Dual Contouring [S1, unofficial]. Unlike [Surface Properties and Estimation](2790-surface-properties-and-estimation.md) which may estimate normals from gradients, this method derives normals analytically or via sampling at the specific intersection points identified by the Marching Cubes topology [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `marked`: An array of linear indices representing cells that intersect the isosurface.
- `markedLen`: The number of active cells in `marked`.
- `storage.cellCount`: The number of cells along one dimension of the voxel grid (grid size is $(N+1) \times (N+1) \times (N+1)$ vertices, $N \times N \times N$ cells).
- `storage.grid`: A 1D array of scalar values (e.g., density) stored in row-major order.
- `offset`: A `float3` representing the global origin offset of the voxel grid in world space.
- `a`: The uniform spacing (voxel size) between grid vertices.
- `acc`: A noise or scalar field accessor object/function used to evaluate density at arbitrary positions.
- `seed`: A seed value for deterministic noise/sampling if required by `acc`.
- `ymin`, `extent`: Parameters defining the sampling range or bounds for the field evaluation.

**Outputs:**
- `data`: A 1D array of `HermiteData` structs. The size is determined by the number of active edges across all marked cells.
- `storage.edgeInfo`: An array of pointers into `data`, indexed by cell, pointing to the start of that cell's Hermite data.

### Data Structures
**UniformVoxelStorage:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct UniformVoxelStorage {
    uint cellCount;       // N: number of cells per axis
    float* grid;          // 1D array of scalar values, size (cellCount + 1)^3
    HermiteData** edgeInfo; // Array of pointers, size cellCount^3
};
```

**HermiteData:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct HermiteData {
    float3 position; // Intersection point in world space
    float3 normal;   // Normal vector at the intersection point
};
```

**Lookup Tables:**
- `specialTable1`: A `256 x 3` integer array. For each of the 256 Marching Cubes configurations, it stores a linked list of edge pairs. Each entry is 3 integers: `edge_index`, `next_entry_index`, `terminator`. The list is terminated by `-2`.
- `specialTable2`: A mapping array used to determine the output index in `data` for each edge pair within a cell. (Note: The source implies a direct mapping based on the edge pair index within the cell's configuration).
- `cornerPoints`: An array of 8 `float3` vertices defining the unit cube corners in the following order:
  0: $(0,0,0)$, 1: $(1,0,0)$, 2: $(1,0,1)$, 3: $(0,0,1)$, 4: $(0,1,0)$, 5: $(1,1,0)$, 6: $(1,1,1)$, 7: $(0,1,1)$.

### Algorithm Steps
1.  **Thread Indexing**: Calculate the global thread index $i_$. If $i_ \ge markedLen$, terminate.
2.  **Cell Retrieval**: Retrieve the linear cell index $i = marked[i_]$.
3.  **Coordinate Decoding**: Decode 3D coordinates $(x, y, z)$ from $i$ using stride $S = storage.cellCount + 1$:
    $$ x = i \pmod S $$
    $$ y = \left\lfloor \frac{i}{S} \right\rfloor \pmod S $$
    $$ z = \left\lfloor \frac{i}{S^2} \right\rfloor \pmod S $$
4.  **Cell Origin**: Calculate the minimum corner position of the cell in world space:
    $$ \mathbf{c}_{min} = \mathbf{offset} + (x \cdot a, y \cdot a, z \cdot a) $$
5.  **Configuration Bitmask**: Construct an 8-bit integer `config`. For each vertex $v_k$ of the cube ($k=0..7$), check if the density value $D(v_k) < 0.0$. If true, set bit $k$ of `config` to 1. The vertex order corresponds to `cornerPoints`.
6.  **Edge Pair Lookup**: Access `specialTable1` at index `config`. Let `entry` point to the start of the list for this configuration.
7.  **Iterate Edge Pairs**: While `entry` is not the terminator (-2):
    a.  Retrieve the edge pair indices $(e_1, e_2)$ from the current entry.
    b.  Get the vertex indices $(v_1, v_2)$ corresponding to these edges from the cube topology.
    c.  Define the edge segment in world space:
        $$ \mathbf{p}_1 = \mathbf{c}_{min} + \mathbf{cornerPoints}[v_1] \cdot a $$
        $$ \mathbf{p}_2 = \mathbf{c}_{min} + \mathbf{cornerPoints}[v_2] \cdot a $$
        $$ \mathbf{edge} = (\mathbf{p}_1, \mathbf{p}_2) $$
    d.  **Sample Intersection**: Find the point $\mathbf{p}_{int}$ on $\mathbf{edge}$ where the scalar field is zero (or crosses the threshold). This is typically done via linear interpolation if the field is piecewise linear, or via root-finding if `acc` provides a continuous function.
        $$ \mathbf{p}_{int} = \text{sampleSurfaceIntersection}(\mathbf{edge}, \mathbf{acc}, \text{seed}, \text{ymin}, \text{extent}) $$
    e.  **Calculate Normal**: Compute the normal $\mathbf{n}$ at $\mathbf{p}_{int}$. This may involve sampling the gradient of the field or using a precomputed normal from the noise function.
        $$ \mathbf{n} = \text{calculateNormal}(\mathbf{p}_{int}, \frac{a}{1024.0}, \text{seed}, \text{ymin}, \text{extent}) $$
    f.  **Store Data**: Store the pair $(\mathbf{p}_{int}, \mathbf{n})$ into the `data` array. The index is determined by `specialTable2` and the current edge pair index.
    g.  **Advance**: Move `entry` to the next item in the linked list.
8.  **Pointer Storage**: Store the pointer to the first Hermite data element for this cell into `storage.edgeInfo` at index $indexCell(N, x, y, z)$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_hermite_data(marked, marked_len, storage, acc, seed, ymin, extent):
    """
    Generates Hermite data for all marked cells.
    Assumes parallel execution where each thread handles one marked cell.
    """
    # 1. Thread Indexing
    # In a real GPU kernel, this would be:
    # i_ = blockIdx.x * blockDim.x + threadIdx.x + indexOffset
    # For sequential simulation:
    for i_ in range(marked_len):
        # 2. Cell Retrieval
        i = marked[i_]
        
        # 3. Coordinate Decoding
        S = storage.cell_count + 1
        x = i % S
        y = (i // S) % S
        z = (i // (S * S)) % S
        
        # 4. Cell Origin
        cell_min = storage.offset + float3(x * storage.a, y * storage.a, z * storage.a)
        
        # 5. Configuration Bitmask
        config = 0
        for k in range(8):
            # Get vertex position in grid
            vx = x + corner_points[k].x
            vy = y + corner_points[k].y
            vz = z + corner_points[k].z
            # Get density value
            density = acc.sample(vx, vy, vz, seed, ymin, extent)
            if density < 0.0:
                config |= (1 << k)
        
        # 6. Edge Pair Lookup
        entry_ptr = special_table1[config]
        
        # 7. Iterate Edge Pairs
        current_entry = entry_ptr
        data_start_index = 0 # Track offset within this cell's allocated block
        
        while current_entry != -2:
            # Get edge pair indices from table
            edge_pair = special_table1[current_entry] # Returns (edge_idx_1, edge_idx_2, next_ptr)
            edge_idx_1, edge_idx_2, next_ptr = edge_pair
            
            # Get vertex indices for the edges
            # Note: Edge indices in MC tables usually refer to specific edges of the cube.
            # We need a mapping from edge index to vertex indices.
            # Assuming edge_pairs table provides vertex indices directly as per source note:
            corners = edge_pairs[current_entry] # Returns (v1, v2)
            v1, v2 = corners
            
            # Define edge segment
            p1 = cell_min + corner_points[v1] * storage.a
            p2 = cell_min + corner_points[v2] * storage.a
            edge = (p1, p2)
            
            # Sample Intersection
            p_int = sample_surface_intersection(edge, acc, seed, ymin, extent)
            
            # Calculate Normal
            # The source uses a step size of a/1024.0 for normal calculation
            normal = calculate_normal(p_int, storage.a / 1024.0, seed, ymin, extent)
            
            # Store Data
            # The source uses special_table2 to determine the index in the global data array
            # Assuming special_table2 maps the edge pair index to a local offset
            local_offset = special_table2[current_entry]
            global_index = i_ * max_edges_per_cell + local_offset # Simplified indexing
            # In the source, data is written to data[3 * i_ + specialTable2[curEntry]]
            # This implies a fixed-size allocation per cell or a compacting scheme.
            # Given the source writes to 'data' directly, we assume 'data' is pre-allocated
            # or we are writing to a thread-local buffer that is later copied.
            # The source says: data[3 * i_ + specialTable2[curEntry]]
            # This suggests a fixed stride of 3 * marked_len? No, that would be huge.
            # It likely means data is an array of HermiteData, and the index is calculated.
            # Let's assume the source's 'data' array is large enough and indexed by i_ and edge.
            data[global_index] = HermiteData(p_int, normal)
            
            # Advance
            current_entry = next_ptr
            data_start_index += 1
            
        # 8. Pointer Storage
        cell_idx = index_cell(storage.cell_count, x, y, z)
        storage.edge_info[cell_idx] = data + (i_ * max_edges_per_cell) # Approximate pointer
```

### Parameters
- `offset`: Global offset of the voxel grid origin.
- `extent`: Sampling extent, calculated as $size \cdot a / 2$.
- `a`: Voxel spacing.
- `specialTable1`: 256x3 integer array mapping Marching Cubes configuration (0-255) to a linked list of edge indices (terminated by -2).
- `cornerPoints`: Array of 8 `float3` vertices: $(0,0,0), (1,0,0), (1,0,1), (0,0,1), (0,1,0), (1,1,0), (1,1,1), (0,1,1)$.

### Edge Cases and Pitfalls
- **Empty Configurations**: The code assumes that if a cell is marked, its configuration will have at least one valid edge pair (i.e., `curEntry != -2` initially). This is guaranteed by the `markCell` function which filters out empty configurations [S1, unofficial].
- **Precision**: The normal calculation uses a small step size ($a/1024.0$) for finite differences or sampling, which may introduce precision errors if $a$ is very small or very large [S1, unofficial].
- **Boundary Conditions**: The decoding of coordinates assumes the grid is regular and the `marked` array only contains valid indices within the grid bounds [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(M)$, where $M$ is the number of marked cells. Each cell is processed in constant time (256 configurations, fixed number of edges).
- **Memory Complexity**: $O(M \cdot E)$, where $E$ is the average number of edges per cell (typically 3-4). The `edgeInfo` array is $O(N^3)$.
- **Performance**: The method is highly parallelizable, as each cell's Hermite data can be computed independently. It is suitable for GPU implementation [S1, unofficial].
- **Strengths**: Decouples topology from geometry, allowing for flexible downstream meshing (e.g., Surface Nets, Dual Contouring). Efficient for sparse surfaces.
- **Limitations**: Requires pre-marking of cells. The quality of normals depends on the `calculateNormal` implementation.

## Background
This method is derived from the implementation in the `voxelized3d` repository, which implements a voxel-based surface extraction technique using Marching Cubes topology to generate Hermite data for further processing [S1, unofficial]. It is part of a pipeline that converts volumetric data into a mesh representation.

## Sources
[S1] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
