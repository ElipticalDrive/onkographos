# Leaf Node Gaussian Approximation

Also called: Single Gaussian per Leaf, Spatial Clustering and Averaging

## Overview
Leaf Node Gaussian Approximation is a method to approximate VDB leaf nodes (typically 512 voxels) with a single Gaussian primitive to reduce rendering complexity while maintaining spatial coherence [S1, unofficial]. It serves as the lowest Level of Detail (LOD) in a hierarchical Gaussian representation, leveraging the inherent spatial clustering of VDB leaf nodes [S1, unofficial]. Unlike higher LOD variants that subdivide leaf nodes into smaller blocks, this method treats the entire leaf node as a single cluster, making it distinct from [Medium LOD Approximation](4128-voxel-representation-with-gaussian-lobes.md#medium-lod-approximation) and [High LOD Approximation](4128-voxel-representation-with-gaussian-lobes.md#high-lod-approximation).

## How it works
The method processes VDB leaf nodes to generate a set of axis-aligned Gaussian primitives. It distinguishes between dense and sparse leaf nodes to calculate position, opacity, and covariance.

### Inputs and Outputs
*   **Input:** A VDB grid with Tiled data storage. Leaf nodes are either dense (all voxels active) or sparse (subset of voxels active).
*   **Output:** A global array of Gaussian parameters (position, opacity, covariance) and metadata for OptiX Geometry Acceleration Structure (GAS) creation.

### Data Structures
The implementation uses thread-local buffers during parallel processing to avoid contention. Each thread maintains:
*   `thread_gaussian_positions`: List of 3D centers (world coordinates).
*   `thread_gaussian_opacities`: List of scalar opacity values.
*   `thread_gaussian_covariances`: Flat array of diagonal elements of a $3 \times 3$ covariance matrix (variances in X, Y, Z). Off-diagonal elements are zero (axis-aligned).
*   `localLeafData`: Metadata for Axis-Aligned Bounding Boxes (AABBs), including point offsets and counts for the global buffer [S1, unofficial].

### Steps
1.  **Verify Tiled Storage:** Ensure the VDB grid uses Tiled data storage. This guarantees no data is omitted. Skip leaf nodes that are excluded because they have a fixed value for every voxel; these are stored in the parent node's value mask [S1, unofficial].
2.  **Parallel Processing:** Process leaf nodes in parallel using Intel Thread Building Blocks (TBB). Each thread processes one leaf node, collecting data into its thread-local buffers [S1, unofficial].
3.  **Global Write:** Write collected Gaussian data to a global array using a mutex for safety [S1, unofficial].
4.  **Process Sparse Leaf Nodes:**
    *   **Bounding Box:** Traverse all active voxels. Initialize the bounding box to the dimensions of the starting voxel. Iteratively expand the bounding box. If no active voxels are found, exit early [S1, unofficial].
    *   **Coordinates:** Calculate the minimum and maximum coordinates of the active bounding box in VDB index space ($bboxMin$, $bboxMax$) [S1, unofficial].
    *   **Position:** Transform the centroid of the active voxel region into world space to serve as the Gaussian position. World coordinates are calculated via the grid transformation matrix $M$: $p_{world} = M \cdot p_{index}$ [S1, unofficial].
    *   **Opacity:** Average the opacity values of all active voxels to estimate overall opacity. Optionally scale the result to account for the Gaussian’s spatial extent [S1, unofficial].
    *   **Covariance:** Construct a diagonal covariance matrix from the radii. The radii vector $\sigma$ is calculated as:
        $$ \sigma = \frac{1}{2} \cdot (bboxMax - bboxMin) \cdot voxelSize $$
        where $\sigma$ and $voxelSize$ are vectors of 3 floating-point values representing radii and voxel dimensions in each axis, respectively. This defines an axis-aligned ellipsoid [S1, unofficial].
5.  **Process Dense Leaf Nodes:**
    *   **Position:** Calculate the Gaussian center in world coordinates through its Index space bounding boxes. For dense nodes, the bounding box corresponds to the entire leaf node dimensions [S1, unofficial].
    *   **Covariance:** If $voxelSize$ is isotropic, the resulting Gaussians are mostly spherical. The covariance is derived from the leaf node's spatial extent [S1, unofficial].
6.  **Store Parameters:** Store Gaussian parameters (position, opacity, covariance) and metadata for OptiX rendering in output buffers [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading

# Global state
global_gaussian_positions = []
global_gaussian_opacities = []
global_gaussian_covariances = []
global_mutex = threading.Lock()

def process_leaf_node(leaf_node, grid_transform_matrix, voxel_size):
    """
    Processes a single VDB leaf node and appends results to global buffers.
    """
    # Thread-local buffers
    local_positions = []
    local_opacities = []
    local_covariances = []
    
    # Check if leaf is excluded (fixed value stored in parent)
    if leaf_node.is_excluded_by_value_mask():
        return

    # Determine if dense or sparse
    if leaf_node.is_sparse():
        # Sparse Leaf Node Approximation
        active_voxels = leaf_node.get_active_voxels()
        
        if not active_voxels:
            return # No active voxels, skip

        # Calculate AABB in index space
        bbox_min_idx = active_voxels[0].index
        bbox_max_idx = active_voxels[0].index
        
        for voxel in active_voxels:
            idx = voxel.index
            bbox_min_idx = min(bbox_min_idx, idx)
            bbox_max_idx = max(bbox_max_idx, idx)
            
        # Calculate centroid in index space
        centroid_idx = (bbox_min_idx + bbox_max_idx) / 2.0
        
        # Transform to world space
        centroid_world = grid_transform_matrix.transform(centroid_idx)
        
        # Calculate Opacity (average of active voxels)
        total_opacity = sum(voxel.opacity for voxel in active_voxels)
        opacity = total_opacity / len(active_voxels)
        
        # Calculate Covariance (Radii)
        # sigma = 0.5 * (bboxMax - bboxMin) * voxelSize
        bbox_size_idx = bbox_max_idx - bbox_min_idx
        radii = 0.5 * bbox_size_idx * voxel_size
        
        # Covariance is diagonal matrix with radii^2? 
        # Note: Source defines sigma as radii. Standard Gaussian uses variance.
        # Source says "diagonal covariance matrix ... from the radii".
        # We store the diagonal elements. Assuming source implies variances are derived from radii.
        # However, source formula gives sigma (radii). 
        # Implementation detail: Store radii as diagonal elements if that's what the renderer expects,
        # or square them for variance. Source says "encoding variances in X, Y and Z".
        # So we must square the radii.
        variance = radii ** 2
        
        local_positions.append(centroid_world)
        local_opacities.append(opacity)
        local_covariances.extend(variance) # Flat array [var_x, var_y, var_z]
        
    else:
        # Dense Leaf Node Approximation
        # Dense nodes have consistent structure.
        # Center is geometric center of the leaf node in index space.
        leaf_bbox_min_idx = leaf_node.get_bbox_min_index()
        leaf_bbox_max_idx = leaf_node.get_bbox_max_index()
        
        centroid_idx = (leaf_bbox_min_idx + leaf_bbox_max_idx) / 2.0
        centroid_world = grid_transform_matrix.transform(centroid_idx)
        
        # Opacity: For dense, typically the value is uniform or averaged.
        # Source implies using the leaf's representative value.
        opacity = leaf_node.get_representative_opacity()
        
        # Covariance: Based on leaf node size.
        bbox_size_idx = leaf_bbox_max_idx - leaf_bbox_min_idx
        radii = 0.5 * bbox_size_idx * voxel_size
        variance = radii ** 2
        
        local_positions.append(centroid_world)
        local_opacities.append(opacity)
        local_covariances.extend(variance)

    # Write to global buffer with mutex
    with global_mutex:
        global_gaussian_positions.extend(local_positions)
        global_gaussian_opacities.extend(local_opacities)
        global_gaussian_covariances.extend(local_covariances)

def run_parallel_approximation(grid):
    """
    Main entry point to process all leaf nodes in parallel.
    """
    leaf_nodes = grid.get_leaf_nodes()
    grid_transform = grid.get_transform_matrix()
    voxel_size = grid.get_voxel_size()
    
    threads = []
    for leaf in leaf_nodes:
        t = threading.Thread(target=process_leaf_node, args=(leaf, grid_transform, voxel_size))
        threads.append(t)
        t.start()
        
    for t in threads:
        t.join()
        
    return global_gaussian_positions, global_gaussian_opacities, global_gaussian_covariances
```

### Parameters and Edge Cases
*   **Voxel Size:** Must be provided. If isotropic, Gaussians are spherical. If anisotropic, they are ellipsoidal [S1, unofficial].
*   **Sparse Nodes with No Active Voxels:** The algorithm exits early and emits no Gaussian [S1, unofficial].
*   **Tiled Storage:** Essential for correctness. Non-tiled grids may require different traversal logic [S1, unofficial].
*   **Precision:** Floating-point precision issues may arise when transforming index coordinates to world space for very large grids.

## Subtopics

### Medium LOD Approximation
A refinement level that subdivides leaf nodes into blocks to reduce detail while preserving volumetric distribution, using different algorithms for dense and sparse nodes [S1, unofficial]. For dense leaf nodes, it applies uniform spatial partitioning into non-overlapping $4 \times 4 \times 4$ blocks (64 voxels each), computing mean opacity and placing Gaussians at block centers [S1, unofficial]. For sparse leaf nodes, it attempts to form $2 \times 2 \times 2$ blocks of adjacent active voxels; if found, a Gaussian is emitted at the block centroid; otherwise, isolated active voxels are represented individually with half-voxel radii [S1, unofficial]. Choose this when higher fidelity than the single-leaf approximation is needed without the cost of High LOD.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of leaf nodes, $O(N_{leaves})$. Each leaf is processed once. Sparse nodes require traversal of active voxels, proportional to the number of active voxels in the leaf.
*   **Memory Complexity:** Linear with respect to the number of generated Gaussians. Each leaf node produces exactly one Gaussian primitive [S1, unofficial].
*   **Performance:** Reduces rendering complexity significantly by clustering 512 voxels into one primitive. Suitable for real-time rendering of large VDB datasets where spatial coherence is more important than fine detail [S1, unofficial].
*   **Strengths:** Simple implementation, low memory overhead, leverages existing VDB structure.
*   **Limitations:** Loses all internal detail of the leaf node. Not suitable for scenes requiring high-frequency volumetric features.
*   **When to Use:** Low LOD rendering, distant views, or when computational resources are severely constrained.
*   **When Not to Use:** High LOD requirements, close-up views, or when internal density variations within a leaf node are significant.

## Background
The method originates from the study of 3D Gaussian particle approximation of VDB datasets for scientific visualization [S1, unofficial]. It was developed to enable efficient rendering of large volumetric datasets using ray tracing acceleration structures like OptiX GAS [S1, unofficial]. The approach utilizes Intel Thread Building Blocks (TBB) for parallel processing, which is a default backend for multi-threading in the OpenVDB library ecosystem [S1, unofficial].

## Sources
[S1] 3D Gaussian Particle Approximation of VDB Datasets: A Study for Scientific Visualization, https://www.emergentmind.com/papers/2504.04857, fetched 2026-10-29 (unofficial)
