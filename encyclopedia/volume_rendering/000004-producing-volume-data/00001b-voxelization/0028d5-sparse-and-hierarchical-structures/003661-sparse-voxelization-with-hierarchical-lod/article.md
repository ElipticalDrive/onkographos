# Sparse Voxelization with Hierarchical LoD

Also called: Fabre et al. voxelization, Fast Voxelization and Level of Detail for Microgeometry Rendering

## Overview
This method is a voxelization pipeline designed for high-frequency sparse geometric objects (fibers, hair, thin planes) that processes volumetric data into multiple Levels of Detail (LoD), preserving appearance information by separating geometric/directional data from material properties [S1]. It differs from general-purpose voxelizers by explicitly handling microgeometry via subsampling and encoding directional distributions into SGGX functions, rather than relying on simple occupancy or fixed-resolution grids [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh or a collection of splines (e.g., Catmull-Rom).
*   **Output:** A hierarchical sparse voxel structure with multiple LoD levels, where leaf nodes contain aggregated directional data (SGGX) and material properties.

**Data Structures:**
1.  **Node Structure (Initial):** Each node stores:
    *   Translation $p=(x,y,z) \in \mathbb{R}^3$ [S1].
    *   Normal $n=(n_x,n_y,n_z) \in \mathbb{R}^3$ with $\|n\|=1$ [S1].
    *   Tangent $t=(t_x,t_y,t_z) \in \mathbb{R}^3$ with $\|t\|=1$ [S1].
    *   Properties ID: A 32-bit integer $id \in \mathbb{Z}^{32}$ [S1].
2.  **Per-Sample Data Structure (Table 1):** Used during generation [S1]:
    *   2nd level index: integer, 4 bytes.
    *   3rd level index: integer, 4 bytes.
    *   Tangent data: u-integer, 3x1 bytes.
    *   Normal data: u-integer, 3x1 bytes.
    *   Material ID: u-integer, 4 bytes.

**Parameters:**
*   $Resolution$: The target resolution for the first two levels. The third level uses a fixed resolution of 8 voxels per axis in the examples [S1].
*   $\delta$: Maximum generation radius (e.g., max yarn radius or triangle edge length) [S1].
*   $S_{block}$: Block size [S1].

**Steps:**

1.  **Global Setup:** Compute a global bounding box $B(b_{min}, b_{max})$ and the maximum generation radius $\delta$ for the input model [S1].
2.  **Coordinate Transformation:** Transform samples from global space to normalized coordinates in range $[0, Resolution)$ using:
    $$ P_{normalized} = \frac{P - b_{min}}{b_{max} - b_{min}} $$
    $$ P_{volume} = P_{normalized} * Resolution $$
    [S1]
3.  **Block Filtering (GPU Kernel 1):** Generate maximum nodes for first-level blocks. Identify and discard empty blocks by checking control nodes using a distance function. For a block with position $P_{block}$ and 3D index $I_{block}$, compute the node position relative to the block $P_{relative}$:
    $$ P_{block} = S_{block} * I_{block} $$
    $$ P_{relative} = P_{volume} - P_{block} $$
    $$ Q = P_{relative} - \frac{S_{block}}{2} $$
    $$ D = \| \max(Q, 0) \| + \min(\max(Q_x, Q_y, Q_z), 0) $$
    Discard nodes where $D \ge \delta^2$ [S1].
4.  **Mask Generation:** Initialize a CPU flat array with space for the number of nodes per block and an extra counter. Use a kernel to write the number of valid nodes and their IDs for each block. Copy the mask to CPU to identify empty first-level blocks and generate an axis-aligned stencil for volumetric path tracing [S1].
5.  **Sparse Data Sampling (Kernels 2-4, for non-empty blocks only):**
    *   **Prepare Initial Nodes:** Reorder, duplicate, or generate additional data to ensure consecutive samples are accessible [S1].
    *   **Subsample:**
        *   *For Splines (Catmull-Rom):* Define the spline from 4 points and generate additional samples using linear spline interpolation [S1].
        *   *For Triangles:* Generate additional points on the surface using barycentric coordinates [S1].
        *   *Distribution:* Distribute samples equally based on kernel indices (not random). For triangles, weight sample count by relative area to the largest triangle in the model [S1].
    *   **Generate Voxel Nodes:** Create voxel nodes from the new samples [S1].
6.  **Sample Gathering:** Move samples to a grid-like structure. Save only the last sample generated at each voxel asynchronously to avoid implicit ordering of draw calls [S1].
7.  **Block Aggregation:** Order samples by second-level block ID. Use a kernel to detect vector positions where the ID changes to determine block start positions and sample counts [S1].
8.  **Directional Data Encoding:** Compute a distribution histogram for directional data (normals/tangents) at every voxel with density $> 0$. Fit each computed histogram to an SGGX function [18] to encode the distribution using 6 values [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_voxelization_hierarchical_lod(mesh_or_splines, resolution, delta):
    # 1. Global Setup
    b_min, b_max = compute_global_bounding_box(mesh_or_splines)
    
    # 2. Coordinate Transformation
    # P_volume is computed per node/sample
    def transform_to_volume(P):
        P_norm = (P - b_min) / (b_max - b_min)
        return P_norm * resolution

    # 3. Block Filtering
    # Compute block indices and filter empty blocks
    valid_blocks = []
    for node in mesh_or_splines.nodes:
        P_vol = transform_to_volume(node.position)
        I_block = floor(P_vol / S_block)
        P_block = I_block * S_block
        P_rel = P_vol - P_block
        Q = P_rel - (S_block / 2)
        D = norm(max(Q, 0)) + min(max(Q.x, Q.y, Q.z), 0)
        
        if D < delta**2:
            valid_blocks.append((I_block, node))
            
    # Identify unique valid first-level blocks
    unique_valid_blocks = set(I_block for I_block, _ in valid_blocks)
    
    # Generate stencil for empty blocks
    stencil = generate_axis_aligned_stencil(unique_valid_blocks)
    
    # 4. Mask Generation
    # Initialize CPU array for valid nodes per block
    block_node_counts = {block_id: 0 for block_id in unique_valid_blocks}
    block_node_ids = {block_id: [] for block_id in unique_valid_blocks}
    
    for I_block, node in valid_blocks:
        block_node_counts[I_block] += 1
        block_node_ids[I_block].append(node.id)
        
    # Copy mask to CPU (simulated)
    empty_blocks = [b for b in unique_valid_blocks if block_node_counts[b] == 0]
    
    # 5. Sparse Data Sampling (for non-empty blocks)
    voxel_samples = []
    
    for block_id in unique_valid_blocks:
        nodes = block_node_ids[block_id]
        
        # Step 5.1: Prepare Initial Nodes
        prepared_nodes = prepare_initial_nodes(nodes)
        
        # Step 5.2: Subsample
        subsampled_points = []
        for node in prepared_nodes:
            if node.is_spline:
                # Catmull-Rom: 4 points define spline, linear interpolation for more
                pts = get_spline_points(node)
                subsampled_points.extend(subsample_spline(pts))
            elif node.is_triangle:
                # Barycentric sampling
                # Weight by relative area to largest triangle
                area_weight = node.area / max_area_in_model
                num_samples = calculate_samples(area_weight)
                subsampled_points.extend(subsample_triangle(node, num_samples))
        
        # Step 5.3: Generate Voxel Nodes
        for pt in subsampled_points:
            P_vol = transform_to_volume(pt)
            voxel_samples.append({
                'position': P_vol,
                'normal': pt.normal,
                'tangent': pt.tangent,
                'material_id': pt.material_id,
                'block_id': block_id
            })
            
    # 6. Sample Gathering
    # Save last sample at each voxel asynchronously
    voxel_grid = {}
    for sample in voxel_samples:
        voxel_key = tuple(floor(sample['position']))
        voxel_grid[voxel_key] = sample # Overwrites previous, keeping last
        
    # 7. Block Aggregation
    # Order by second-level block ID
    samples_by_block = group_by_block(voxel_grid.values())
    
    # Detect block boundaries
    block_boundaries = detect_id_changes(samples_by_block)
    
    # 8. Directional Data Encoding
    final_voxels = {}
    for block_id, samples in samples_by_block.items():
        if len(samples) > 0:
            # Compute histogram of normals/tangents
            hist = compute_directional_histogram(samples)
            # Fit to SGGX
            sggx_params = fit_sggx(hist)
            final_voxels[block_id] = {
                'sggx': sggx_params,
                'density': compute_density(samples),
                'material_id': samples[0]['material_id'] # Example aggregation
            }
            
    return final_voxels, stencil
```

## Operating characteristics
*   **Time Complexity:** Dominated by GPU kernels for sampling and aggregation. The method avoids fixed-size memory allocation for sub-block grids, scaling with the number of samples rather than voxel resolution [S1].
*   **Memory Complexity:** Memory usage scales with the number of generated samples, not the voxel resolution. Empty voxels do not waste memory [S1].
*   **Strengths:** Efficiently handles high-frequency sparse details (fibers, hair) without rasterization artifacts or fixed-size memory overhead. Preserves appearance information via SGGX encoding [S1].
*   **Limitations:** Requires careful subsampling to avoid discontinuities. The method is specific to sparse microgeometry and may not be optimal for dense, solid volumes [S1].

## Background
The method was introduced by Javier Fabre et al. in "Fast Voxelization and Level of Detail for Microgeometry Rendering" [S1]. It addresses the complexity of aggregating fine sparse details into coarser blocks, which usually implies rasterization and fixed-size memory allocation for sub-block grids [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
