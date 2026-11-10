# Sparse Voxelization with Hierarchical LoD

Also called: Fabre et al. voxelization, Fast Voxelization and Level of Detail for Microgeometry Rendering

## Overview
Sparse Voxelization with Hierarchical LoD is a voxelization pipeline designed for high-frequency sparse geometric objects (fibers, hair, thin planes) that processes volumetric data into multiple Levels of Detail (LoD), preserving appearance information by separating geometric/directional data from material properties [S1]. It differs from general-purpose voxelizers by explicitly handling anisotropic microgeometry through directional histogram fitting (SGGX) and asynchronous sample gathering to avoid draw-call ordering artifacts [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D primitive collection (triangle mesh or Catmull-Rom spline collection) with associated material properties [S1].
*   **Output:** A hierarchical sparse voxel structure (typically 3 levels) where leaf nodes store aggregated directional data (SGGX) and material IDs [S1].

**Data Structures**
The method uses a hierarchical block structure [S1]. The specific layout for initial sample data (used to compute leaf node information) is defined in Table 1 of [S1]:

| Field | Type | Size | Description |
| :--- | :--- | :--- | :--- |
| 2nd level index | integer | 4 bytes | Block identifier |
| 3rd level index | integer | 4 bytes | Sub-block identifier |
| Tangent data | u-integer | 3x1 bytes | Quantized tangent vector |
| Normal data | u-integer | 3x1 bytes | Quantized normal vector |
| Material ID | u-integer | 4 bytes | Property identifier |

Each node in the hierarchy stores:
*   Translation $p=(x,y,z) \in \mathbb{R}^3$ [S1]
*   Normal $n=(n_x,n_y,n_z) \in \mathbb{R}^3$ with $\|n\|=1$ [S1]
*   Tangent $t=(t_x,t_y,t_z) \in \mathbb{R}^3$ with $\|t\|=1$ [S1]
*   Properties ID: a 32-bit integer $id \in \mathbb{Z}^{32}$ [S1].

**Parameters**
*   $Resolution$: The target resolution for the first two levels [S1].
*   $\delta$: Maximum generation radius (e.g., max yarn radius or triangle edge length) [S1].
*   $S_{block}$: Block size [S1].

**Steps**

1.  **Bounding Box and Coordinate Transformation**
    Compute the global bounding box $B(b_{min}, b_{max})$ and the maximum generation radius $\delta$ for the input model [S1]. Transform samples from global space to normalized coordinates in range $[0, Resolution)$:
    $$ P_{normalized} = \frac{P - b_{min}}{b_{max} - b_{min}} $$
    $$ P_{volume} = P_{normalized} \times Resolution $$
    [S1]

2.  **First-Level Block Generation and Culling**
    Run a parallel GPU kernel to generate maximum nodes for first-level blocks. Identify and discard empty blocks by checking control nodes using a distance function [S1].
    Compute the block position $P_{block}$ and relative position $P_{relative}$:
    $$ P_{block} = S_{block} \times I_{block} $$
    $$ P_{relative} = P_{volume} - P_{block} $$
    [S1]
    Compute the distance $D$ to determine if a node should be kept (even if outside the immediate block, it might generate samples within it):
    $$ Q = P_{relative} - \frac{S_{block}}{2} $$
    $$ D = \| \max(Q, 0) \| + \min(\max(Q_x, Q_y, Q_z), 0) $$
    Discard nodes where $D \ge \delta^2$ [S1].

3.  **Masking and Stencil Generation**
    Initialize a CPU flat array with space for the number of nodes per block and an extra counter. Use a kernel to write the number of valid nodes and their IDs for each block. Copy the mask to CPU to identify empty first-level blocks and generate an axis-aligned stencil for volumetric path tracing [S1].

4.  **Sparse Data Sampling (for non-empty blocks)**
    Run three kernels:
    *   **Prepare initial nodes:** Reorder, duplicate, or generate additional data to ensure continuous coverage [S1].
    *   **Subsample:** Generate additional points.
        *   For **Catmull-Rom splines**: Define the spline from 4 points and generate additional samples using linear spline interpolation [S1].
        *   For **Triangles**: Generate additional points on the surface using barycentric coordinates [S1].
    *   **Generate voxel nodes:** Distribute samples equally based on kernel indices (not random). For triangles, weight sample count by relative area to the largest triangle in the model [S1].

5.  **Sample Gathering**
    Move samples to a grid-like structure. Save only the last sample generated at each voxel asynchronously to avoid implicit ordering of draw calls [S1]. Order samples by second-level block ID. Use a kernel to detect vector positions where the ID changes to determine block start positions and sample counts [S1].

6.  **Directional Data Aggregation**
    Compute a distribution histogram for directional data (normals/tangents) at every voxel with density $> 0$. Fit each computed histogram to an SGGX function [18] to encode the distribution using 6 values. This enables easy sampling and aggregation [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_voxelization_hierarchical_lod(primitives, resolution, delta, S_block):
    # 1. Bounding Box and Coordinate Transformation
    b_min = min(v.global_pos for prim in primitives for v in prim.vertices)
    b_max = max(v.global_pos for prim in primitives for v in prim.vertices)
    
    for prim in primitives:
        for v in prim.vertices:
            v.volume_pos = ((v.global_pos - b_min) / (b_max - b_min)) * resolution

    # 2. First-Level Block Generation and Culling
    num_blocks = int(resolution / S_block) + 1
    block_counts = [0] * num_blocks
    block_ids = [[] for _ in range(num_blocks)]
    
    for block_idx in range(num_blocks):
        block_start = block_idx * S_block
        valid_nodes = []
        
        for prim in primitives:
            for v in prim.vertices:
                P_relative = v.volume_pos - block_start
                Q = P_relative - (S_block / 2.0)
                Q_max = max(Q[0], max(Q[1], Q[2]))
                D = sum(max(q, 0.0) for q in Q) + min(Q_max, 0.0)
                
                if D < delta ** 2:
                    if prim.type == 'spline':
                        # Catmull-Rom: define from 4 points, linear interpolation
                        samples = generate_spline_samples(prim)
                    else:
                        # Triangles: barycentric sampling, weighted by relative area
                        samples = generate_triangle_samples(prim)
                        
                    for s in samples:
                        if block_start <= s.volume_pos < block_start + S_block:
                            valid_nodes.append(s)
        
        block_counts[block_idx] = len(valid_nodes)
        block_ids[block_idx] = [id(n) for n in valid_nodes]

    # 3. Masking and Stencil Generation
    empty_blocks = [i for i, c in enumerate(block_counts) if c == 0]
    stencil = generate_axis_aligned_stencil(empty_blocks, resolution) # [G] standard rasterization step

    # 4. Sparse Data Sampling (for non-empty blocks)
    final_leaf_nodes = []
    
    for block_idx in range(num_blocks):
        if block_counts[block_idx] == 0:
            continue
            
        # Kernel 1: Prepare initial nodes
        prepared = prepare_initial_nodes(block_ids[block_idx]) # [G] reorder/duplicate/generate
        
        # Kernel 2: Subsample
        subsampled = subsample_data(prepared) # [G] linear interp for splines, barycentric for triangles
        
        # Kernel 3: Generate voxel nodes
        voxel_nodes = generate_voxel_nodes(subsampled)
        
        # 5. Sample Gathering
        gathered = gather_samples_asynchronously(voxel_nodes)
        gathered.sort(key=lambda s: s.block_2nd_level_id)
        
        # Detect block boundaries
        boundaries = []
        for i in range(1, len(gathered)):
            if gathered[i].block_2nd_level_id != gathered[i-1].block_2nd_level_id:
                boundaries.append(i)
        boundaries.append(len(gathered))
        
        # 6. Directional Data Aggregation
        start = 0
        for end in boundaries:
            block_samples = gathered[start:end]
            start = end
            if len(block_samples) > 0:
                hist = compute_directional_histogram(block_samples)
                sggx_params = fit_sggx(hist) # [G] standard SGGX fitting algorithm
                final_leaf_nodes.append({
                    'sggx': sggx_params,
                    'material_id': block_samples[0].material_id,
                    'position': (block_idx * S_block + S_block / 2.0) * (b_max - b_min) / resolution + b_min
                })
                
    return final_leaf_nodes, stencil
```

**Edge Cases and Pitfalls**
*   **Precision:** The distance function $D$ uses floating-point arithmetic; ensure sufficient precision to correctly cull nodes near the boundary $\delta^2$ [S1].
*   **Draw Call Ordering:** Asynchronous saving of the last sample per voxel is critical to avoid implicit ordering artifacts from parallel draw calls [S1].
*   **Sparse Input:** For very sparse inputs, higher output resolutions require higher numbers of sub-samples to avoid discontinuities [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Memory usage is proportional to the number of valid samples, not the total voxel volume [S1].
*   **Strengths:** Efficiently handles high-frequency sparse geometry (hair, fibers) without wasting memory on empty space [S1]. The SGGX fitting allows compact storage of directional data [S1].
*   **Limitations:** The method is specialized for microgeometry rendering [S1]. Not given in the sources.

## Background
The method was introduced by Javier Fabre et al. in "Fast Voxelization and Level of Detail for Microgeometry Rendering" [S1]. It addresses the challenge of aggregating fine sparse details into coarser blocks, a task that usually implies rasterization and fixed-size memory allocation for sub-block grids [S1]. The approach leverages SGGX functions for orientation distribution representation [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
