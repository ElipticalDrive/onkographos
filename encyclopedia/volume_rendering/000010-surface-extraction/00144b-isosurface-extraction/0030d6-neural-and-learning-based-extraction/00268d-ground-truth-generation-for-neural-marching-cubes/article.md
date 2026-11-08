# Ground Truth Generation for Neural Marching Cubes

Also called: LOD ground truth generation, NMC GT preprocessing

## Overview
This method generates multi-resolution ground truth data for training Neural Marching Cubes (NMC) by computing per-grid-cell labels (case IDs, vertex positions, and plane parameters) from high-resolution voxel and surface intersection data [S1, unofficial]. It solves the problem of creating accurate training targets for a Convolutional Neural Network (CNN) that predicts Marching Cubes configurations, enabling the network to learn the mapping from input fields to mesh primitives [S1, unofficial]. It differs from gradient-based methods by explicitly using computed surface intersection points and their normals for least-squares fitting to determine plane parameters, rather than relying solely on voxel occupancy or field gradients [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** High-resolution voxel grid (e.g., $1024^3$) and a list of surface intersection points with normals derived from a high-resolution mesh or SDF.
*   **Output:** HDF5 datasets containing ground truth labels for target grid resolutions (e.g., $32^3$ or $64^3$), including case IDs, integer parameters (vertex counts), and float parameters (vertex coordinates and plane equations).

**Data Structures:**
The method relies on specific lookup tables and arrays to map voxel configurations to mesh outputs:
1.  **LUT Configs (`configs`):** Shape $32768 \times 3$. Stores `[case_id, #insideCC, #outsideCC]` for all $2^{15}$ possible voxel occupancy patterns (assuming 15-bit connectivity or specific variant).
2.  **LUT Tessellations (`tessellations`):** Shape $32768 \times \text{max\_num\_of\_triangles} \times 3$. Stores triangle vertex indices for each case.
3.  **Integer Parameters (`_int`):** Per cell, stores `num_of_int_params - 1` integers. `num_of_int_params = 1 + 1 + 3 + 1` (Case ID, Corner Vertex Count, Face Vertex Count, Internal Vertex Count).
4.  **Float Parameters (`_float`):** Per cell, stores `num_of_float_params` floats. `num_of_float_params = 8 \times 3 + 3 \times 1 + 12 \times 2` (8 Corner Vertices $\times$ 3 coords, 3 Face Vertices $\times$ 1 coord, 12 Edge Plane Params $\times$ 2 coords).

**Numbered Steps:**
1.  **Downsampling:** Filter high-resolution intersection points where coordinates are multiples of `downscale = 1024 // voxel_size`. Downsample the high-resolution voxels to the target grid size (32 or 64) by aggregating or sampling the original $1024^3$ grid.
2.  **Intersection Aggregation:** Compute `LOD_intersection_pointer` and `LOD_intersection_data` arrays. Use a utility function (e.g., `cutils.get_intersection_points_in_cells`) to map the filtered intersection points to their corresponding cells in the target grid.
3.  **Ground Truth Computation:** Call `cutils.get_gt` with the downsampled voxels, intersection data, LUT configs, and LUT tessellations. This function computes:
    *   Integer parameters: Case ID, and counts of corner, face, and internal vertices.
    *   Float parameters: Positions of corner and face vertices, and plane parameters for edges/faces.
4.  **Storage:** Save the results into HDF5 datasets with the layout: `_case_id` ($(\text{grid\_size}+1)^3$ int32), `_int` ($(\text{grid\_size}+1)^3 \times (\text{num\_of\_int\_params}-1)$ int32), `_float` ($(\text{grid\_size}+1)^3 \times \text{num\_of\_float\_params}$ float32), `_sdf` ($(\text{grid\_size}+1)^3$ float32), and `_voxel` ($(\text{grid\_size}+1)^3$ uint8).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import h5py

def generate_ground_truth(high_res_voxels, high_res_intersections, target_grid_size):
    """
    Generate ground truth for Neural Marching Cubes.
    
    Args:
        high_res_voxels: np.array of shape (1024, 1024, 1024), uint8.
        high_res_intersections: list of tuples (point, normal).
        target_grid_size: int, e.g., 32 or 64.
        
    Returns:
        dict: Ground truth data ready for HDF5 storage.
    """
    # Constants
    base_res = 1024
    downscale = base_res // target_grid_size
    
    # Step 1: Downsample voxels
    # Aggregate voxels to target grid. Assuming simple sampling or mean pooling.
    # Reshape to (target_grid_size, downscale, target_grid_size, downscale, target_grid_size, downscale)
    # and reduce.
    voxels_reshaped = high_res_voxels.reshape(
        target_grid_size, downscale, 
        target_grid_size, downscale, 
        target_grid_size, downscale
    )
    # Example: take the value of the first voxel in each block, or average.
    # The source implies filtering points, but for voxels, it says "Downsample... by filtering points where coordinates are multiples".
    # For voxels, we assume a standard downsampling operation.
    low_res_voxels = voxels_reshaped[::downscale, ::downscale, ::downscale] # Placeholder for actual aggregation logic
    
    # Step 2: Filter and map intersections
    # Filter intersections: keep only those where point coordinates are multiples of downscale in the high-res grid
    filtered_intersections = []
    for point, normal in high_res_intersections:
        if (point[0] % downscale == 0 and 
            point[1] % downscale == 0 and 
            point[2] % downscale == 0):
            filtered_intersections.append((point, normal))
            
    # Map to target grid cells
    # LOD_intersection_pointer and LOD_intersection_data are computed via cutils
    # Here we simulate the mapping: cell_index = (point // downscale)
    cell_indices = []
    for point, normal in filtered_intersections:
        cell_idx = (point // downscale).astype(int)
        # Ensure cell_idx is within bounds
        if np.all(cell_idx < target_grid_size):
            cell_indices.append((cell_idx, normal))
            
    # Step 3: Compute GT using cutils.get_gt
    # This function is a black box in the source, but we define its inputs/outputs
    # Inputs: low_res_voxels, cell_indices (intersection data), configs, tessellations
    # Outputs: case_id, int_params, float_params
    
    # Initialize output arrays
    # Grid size for GT is (target_grid_size + 1)^3 cells? Or target_grid_size^3?
    # Source says: _case_id shape (grid_size+1)^3. This implies the grid of cells includes boundaries.
    num_cells = (target_grid_size + 1)**3
    
    case_id = np.zeros(num_cells, dtype=np.int32)
    int_params = np.zeros((num_cells, 1 + 3 + 1), dtype=np.int32) # Case ID is separate in _case_id? 
    # Source: _int shape (grid_size+1)^3 x (num_of_int_params-1). 
    # num_of_int_params = 1 (case) + 1 (corner) + 3 (face) + 1 (internal) = 6.
    # So _int stores 5 ints per cell.
    int_params = np.zeros((num_cells, 5), dtype=np.int32)
    float_params = np.zeros((num_cells, 8*3 + 3*1 + 12*2), dtype=np.float32)
    
    # Simulate cutils.get_gt call
    # In practice, this iterates over all cells, determines the case ID from voxels,
    # and fills vertex positions and plane parameters from intersections.
    for z in range(target_grid_size + 1):
        for y in range(target_grid_size + 1):
            for x in range(target_grid_size + 1):
                cell_idx = z * (target_grid_size + 1)**2 + y * (target_grid_size + 1) + x
                
                # Determine case ID from low_res_voxels
                # Assuming low_res_voxels is indexed by cell corner or center.
                # Marching Cubes typically uses 8 corners.
                # This is a simplification; actual logic depends on voxel-to-cell mapping.
                voxel_val = low_res_voxels[z, y, x] # Placeholder
                case_id[cell_idx] = determine_case_id(voxel_val) # Placeholder for actual LUT lookup
                
                # Fill parameters based on case ID and intersections
                # This part is complex and handled by cutils
                pass
                
    return {
        'case_id': case_id,
        'int': int_params,
        'float': float_params,
        'sdf': np.zeros(num_cells, dtype=np.float32), # Placeholder
        'voxel': np.zeros(num_cells, dtype=np.uint8) # Placeholder
    }

def determine_case_id(voxel_val):
    # Placeholder for actual Marching Cubes case ID determination
    return 0
```

**Parameters:**
*   `cell_size = 8`: Ratio between grid cell size and base voxel size.
*   `grid_size_list = [32, 64]`: Target grid resolutions for ground truth generation.
*   `configs`: Shape $32768 \times 3$, stores `[case_id, #insideCC, #outsideCC]`.
*   `tessellations`: Shape $32768 \times \text{max\_num\_of\_triangles} \times 3$, stores triangle vertex indices.

**Edge Cases and Pitfalls:**
*   **Alignment:** Voxels and Marching Cubes cells are not naturally aligned. This is handled by padding or offsetting during voxel sampling via `cutils.get_input_voxel` [S1, unofficial].
*   **Precision:** Floating-point precision in vertex positions and plane parameters must be maintained to ensure mesh quality.
*   **Degenerate Input:** Empty grids or grids with no surface intersections must be handled gracefully, likely resulting in zero-filled parameters.

## Subtopics

### [NDC Ground Truth LOD Generation via Least-Squares Plane Fitting](4398-ndc-ground-truth-lod-generation-via-least-squares-plane-fitt.md)
This method generates multi-resolution ground truth data for training Neural Distance Fields (NDC) by fitting planes to surface intersection points within grid cells. It solves the problem of creating accurate per-cell labels (signs and plane parameters) from high-resolution voxel and intersection data, which are then used to train a CNN to predict Marching Cubes configurations. It differs from gradient-based methods by explicitly using computed intersection points and their normals for least-squares fitting.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of high-resolution voxels and intersection points for downsampling and filtering. The ground truth computation per cell is constant time, dependent on the number of intersections per cell.
*   **Memory Complexity:** Proportional to the target grid size cubed, $(\text{grid\_size}+1)^3$, for storing the ground truth datasets.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Provides accurate training data for CNNs by explicitly using surface geometry. Supports multiple resolutions.
*   **Limitations:** Requires high-resolution input data. Computationally intensive for very large grids.
*   **When to use:** When training Neural Marching Cubes or similar neural surface extraction methods.
*   **When not to use:** When real-time ground truth generation is required, or when high-resolution input data is unavailable.

## Background
The method originates from the Neural Marching Cubes (NMC) paper by Zhiqin Chen and Hao Zhang, published in ACM Transactions on Graphics (SIGGRAPH Asia) in 2021 [S1, unofficial]. It was developed to enable the training of neural networks for predicting Marching Cubes configurations, allowing for more flexible and high-quality surface extraction from implicit fields. The supplementary material and code repository provide the implementation details [S1, unofficial].

## Sources
[S1] czq142857/NMC, https://github.com/czq142857/NMC, fetched 2026-10-26 (unofficial)
