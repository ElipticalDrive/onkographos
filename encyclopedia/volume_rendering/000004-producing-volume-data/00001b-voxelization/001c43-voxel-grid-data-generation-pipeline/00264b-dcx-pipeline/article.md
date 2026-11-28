# DCX Pipeline

Also called: Deep Marching Cubes, Deep Marching Cubes Data Pipeline & Mesh Extraction

## Overview
The DCX Pipeline converts point clouds or volumetric inputs into explicit surface meshes by learning a discrete topology class and continuous vertex offsets for each voxel grid cell, then extracting the mesh via a C++/CUDA extension [S1, unofficial]. It differs from its siblings by explicitly predicting topology classes per voxel to control mesh connectivity, rather than relying solely on implicit field isosurfaces or standard marching cubes lookup tables [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud (NumPy array `pts` of shape `(num_data, num_sample, 3)`, dtype float32) or a voxel grid (TSDF/binary occupancy) [S1, unofficial]. Parameters include `resolution` (integer, voxel grid dimension per axis), `bbox` (1D array of 6 floats: `[min_x, min_y, min_z, max_x, max_y, max_z]`), and optional flags for thinning and hole filling [S2, unofficial].
*   **Outputs:** Extracted mesh vertices (dtype float32) and faces (dtype int64) [S2, unofficial].

**Data Structures:**
*   **Voxel Grid Boundaries:** Created via `x_grids = np.arange(0, args.num_cells+1, args.len_cell)` (and identical for y and z axes) [S1, unofficial].
*   **Topology:** PyTorch tensor of shape `(num_cells, num_cells, num_cells)`, dtype long, storing the discrete topology class index per voxel cell [S1, unofficial].
*   **Offset:** PyTorch tensor of shape `(num_cells, num_cells, num_cells, 3)`, dtype float32, storing vertex displacement offsets per cell [S1, unofficial].
*   **Internal C++/CUDA Structures:**
    *   `voxel_ids_arr`: 1D array of long long integers representing voxel IDs [S2, unofficial].
    *   `voxel_points_arr`: 1D array of floats representing voxel points (flattened 3D coordinates) [S2, unofficial].
    *   `cube_types_arr`: 1D array of integers representing cube types [S2, unofficial].
    *   `orders_arr`: 1D array of integers representing orders [S2, unofficial].
    *   `query_points_arr`: 2D array of floats with shape `(N, 3)` representing query points for hole filling [S2, unofficial].
*   **Pre-allocated Output Tensors:** `vertices` of shape `(num_cells**3 * 12, 3)`, dtype float32; `faces` of shape `(num_cells**3 * 12, 3)`, dtype long [S1, unofficial].

**Steps:**
The pipeline executes the following sequential steps [S2, unofficial]:

1.  **points_to_voxels:** Convert input points to voxels [S2, unofficial].
2.  **get_cube_types:** Compute cube types from the voxelized data [S2, unofficial].
3.  **thinning:** Run thinning on voxels and cube types [S2, unofficial].
4.  **reconstruction:** Extract the mesh from the thinned voxels and cube types using a C++/CUDA extension [S1, unofficial][S2, unofficial]. This step uses the predicted topology and offsets to generate vertices and faces [S1, unofficial].
5.  **filling_holes_prepare:** Prepare query points for hole filling [S2, unofficial].
6.  **filling_holes_apply:** Apply selected supplementary points to fill holes [S2, unofficial].

**Pseudocode:**
The following Python-like pseudocode illustrates the pipeline's structure, integrating the data preparation and core pipeline calls [S2, unofficial]. Note that the core `reconstruction` and `thinning` are implemented in C++/CUDA and called via bindings [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def dcx_pipeline(points, bbox, resolution, enable_thinning=False, enable_supplementary_sampling=False, enable_postprocessing=False, dataname="default"):
    """
    Main DCX Pipeline function.
    
    Args:
        points: NumPy array of shape (N, 3), dtype float32, input point cloud.
        bbox: 1D array of 6 floats [min_x, min_y, min_z, max_x, max_y, max_z].
        resolution: Integer, voxel grid resolution per dimension.
        enable_thinning: Boolean, whether to run thinning.
        enable_supplementary_sampling: Boolean, whether to fill holes.
        enable_postprocessing: Boolean, whether to enable post-processing in reconstruction.
        dataname: String, name for the data.
        
    Returns:
        vertices: NumPy array of extracted mesh vertices.
        faces: NumPy array of extracted mesh faces.
    """
    
    # Step 1: points_to_voxels
    # [S2, unofficial] Converts input points to voxel representation.
    # Implementation is a C++/CUDA binding.
    voxel_ids_arr, voxel_points_arr = points_to_voxels(points, bbox, resolution)
    
    # Step 2: get_cube_types
    # [S2, unofficial] Computes cube types from voxels.
    # Implementation is a C++/CUDA binding.
    cube_types_arr, cube_ids_arr, orders_arr = get_cube_types(voxel_ids_arr, voxel_points_arr, resolution)
    
    # Step 3: thinning (Optional)
    if enable_thinning:
        # [S2, unofficial] Runs thinning on voxels and cube types.
        # Implementation is a C++/CUDA binding.
        voxel_ids_arr, voxel_points_arr, cube_types_arr, cube_ids_arr, orders_arr = thinning(
            voxel_ids_arr, voxel_points_arr, cube_types_arr, cube_ids_arr, orders_arr, resolution
        )
    
    # Step 4: reconstruction
    # [S2, unofficial] Extracts mesh from voxels and cube types.
    # Implementation is a C++/CUDA binding.
    margin_voxels, vertices, faces = reconstruction(
        voxel_ids=voxel_ids_arr,
        voxel_points=voxel_points_arr,
        cube_ids=cube_ids_arr,
        cube_types=cube_types_arr,
        orders=orders_arr,
        res=resolution,
        pattern=0, # Default pattern
        enable_postprocessing=enable_postprocessing,
        dataname=dataname
    )
    
    # Deduplicate vertices and remap face indices [S1, unofficial]
    vertices_unique, indices = np.unique(vertices, axis=0, return_index=True)
    faces = faces[indices].reshape((-1, 3))
    
    # Step 5 & 6: filling_holes (Optional)
    if enable_supplementary_sampling:
        # [S2, unofficial] Prepare query points for hole filling.
        # Implementation is a C++/CUDA binding.
        query_points_arr = filling_holes_prepare(
            voxel_ids=voxel_ids_arr,
            voxel_points=voxel_points_arr,
            cube_types=cube_types_arr,
            resolution=resolution
        )
        
        # [S2, unofficial] Apply selected supplementary points to fill holes.
        # Implementation is a C++/CUDA binding.
        added_voxel_ids, added_voxel_points = filling_holes_apply(
            voxel_ids=voxel_ids_arr,
            voxel_points=voxel_points_arr,
            query_points=query_points_arr,
            resolution=resolution
        )
        
        # Merge added voxels into main voxel structure
        for vid, vpt in zip(added_voxel_ids, added_voxel_points):
            voxel_ids_arr = np.append(voxel_ids_arr, vid)
            voxel_points_arr = np.append(voxel_points_arr, vpt)
            
        # Re-run reconstruction to incorporate new geometry
        margin_voxels, vertices, faces = reconstruction(
            voxel_ids=voxel_ids_arr,
            voxel_points=voxel_points_arr,
            cube_ids=cube_ids_arr,
            cube_types=cube_types_arr,
            orders=orders_arr,
            res=resolution,
            pattern=0,
            enable_postprocessing=enable_postprocessing,
            dataname=dataname
        )
        vertices_unique, indices = np.unique(vertices, axis=0, return_index=True)
        faces = faces[indices].reshape((-1, 3))
        
    return vertices_unique, faces
```

**Parameters:**
*   `resolution`: Integer, voxel grid resolution per dimension. Default toy example uses 8 [S1, unofficial][S2, unofficial].
*   `enable_postprocessing`: Boolean, typically set to False during margin extraction and likely True for final extraction [S2, unofficial].
*   `args.bias_level`: Ratio controlling virtual cone removal extent, where cone radius `r = args.num_cells * args.bias_level` [S1, unofficial].
*   `args.noise`: Multiplier for Gaussian point noise standard deviation, scaled by `args.num_cells/2` [S1, unofficial].
*   `args.encoder_type`: Input modality selector, either `'point'` (point cloud) or `'voxel'` (TSDF/binary occupancy grid) [S1, unofficial].

**Data Preparation Formulas (for training/simulation):**
*   **Grid Creation:** `x_grids = np.arange(0, args.num_cells+1, args.len_cell)` (and identical for y and z axes) [S1, unofficial].
*   **Cone Removal Simulation:** Compute cone direction `cone_dir = np.random.uniform(-1, 1, (3, 1))`, normalize it, calculate projection `cone_dist = np.dot(pts_i, cone_dir)`, mask points with `cone_dist < 0`, calculate cone radius `cone_radius = (cone_dist/h) * r`, and mask points where `orth_dist > cone_radius` [S1, unofficial].
*   **Gaussian Noise:** Add noise if `args.noise > 0`: `pts_sampled = pts_sampled + np.random.normal(0, args.noise*args.num_cells/2, pts_sampled.shape)` [S1, unofficial].
*   **Vertex Deduplication:** `vertices_unique, indices = unique_rows(vertices)`, then `faces = faces[indices].reshape((-1, 3))` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If `num_vertices[0] == 0` after mesh extraction, the function returns early to prevent plotting or saving empty meshes [S1, unofficial].
*   The legacy `filling_holes()` function is disabled and throws a runtime error if called directly; users must use `filling_holes_prepare()` followed by `filling_holes_apply()` [S2, unofficial].
*   When `args.bias_level > 0.0`, removed points are padded to a fixed maximum length matrix `pts_removed_mtx` to maintain batch dimensions, filled with `-1` if fewer than max were removed [S1, unofficial].
*   The bounding box array must have exactly 6 elements; otherwise, a runtime error is thrown [S2, unofficial].
*   Precision issues may arise in vertex offset calculations if the grid resolution is too low or the object is very small relative to the grid [G].

## Subtopics

### [DCX Mesh Extraction](4478-dcx-mesh-extraction.md)
DCX Mesh Extraction is a method to extract a mesh from a point cloud using the DCX pipeline, with optional thinning and supplementary sampling to fill holes. It operates on a discrete voxel grid, allowing explicit control over topology via cube types and enabling supplementary sampling for geometric hole filling.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S2, unofficial].
*   **Measured Performance:** Specific performance metrics are not provided in the sources [S2, unofficial]. The use of C++/CUDA extensions for reconstruction suggests an optimization for speed compared to pure Python implementations [S1, unofficial][S2, unofficial].
*   **Scaling:** The method scales with the resolution `R`. Higher resolutions provide more detailed meshes but increase memory and computation time cubically [G].
*   **Strengths:** Explicit control over topology via cube types allows for better handling of complex geometries and topological changes compared to implicit methods [S1, unofficial]. The pipeline is modular, allowing for optional thinning and hole filling [S2, unofficial].
*   **Limitations:** The method requires a voxel grid, which may not be suitable for very large or unbounded scenes without careful bounding box selection [G]. The quality of the mesh depends on the accuracy of the predicted topology and offsets [S1, unofficial].
*   **When to Use:** Suitable for converting point clouds or volumetric data into explicit meshes where topological control is important [S1, unofficial][S2, unofficial].
*   **When Not to Use:** Not ideal for real-time applications with very high-resolution requirements due to the cubic scaling with resolution [G]. Not suitable for scenes without a well-defined bounding box [G].

## Background
The DCX Pipeline is based on the "Deep Marching Cubes: Learning Explicit Surface Representations" paper by Yiyi Liao, Simon Donn\'{e}, and Andreas Geiger, published at CVPR 2018 [S1, unofficial]. The original paper introduced the concept of learning explicit surface representations by predicting topology classes and vertex offsets [S1, unofficial]. The DCX Pipeline implementation, as seen in the GitHub repositories [S1, unofficial][S2, unofficial], extends this concept with a more structured data pipeline, including voxelization, thinning, and hole filling steps, exposed via Python bindings. It is used in research and applications requiring explicit mesh extraction from point clouds or volumetric data [S2, unofficial].

## Sources
[S1] yiyiliao/deep_marching_cubes, https://github.com/yiyiliao/deep_marching_cubes, fetched 2026-10-26 (unofficial)
[S2] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
