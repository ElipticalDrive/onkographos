# Tiny LOD
## Overview
Tiny LOD is a preprocessing method that computes a hierarchical level-of-detail (LOD) structure from volumetric data, generating a tree suitable for subsequent chunk-based storage and rendering [S1, unofficial]. It solves the problem of efficiently reducing the complexity of dense volumetric representations by aggregating voxels into coarser primitives while preserving structural detail [S1, unofficial].

## How it works
Tiny LOD operates as a preprocessing step that transforms raw volumetric input into a compressed hierarchical tree [S1, unofficial].

### Inputs and Outputs
*   **Input:** A volumetric data structure (referred to as "inner data") containing active voxels or splats [S1, unofficial].
*   **Parameters:**
    *   `lod_base` (f32): The base value for LOD computation [S1, unofficial].
    *   `merge_filter` (bool): A boolean flag controlling whether to apply a merge filter during LOD tree computation [S1, unofficial].
    *   `log_callback`: A function for logging progress or errors [S1, unofficial].
*   **Output:** A computed LOD tree structure, followed by encoded opacity values and a chunk tree representation [S1, unofficial].

### Data Structures
The method produces an intermediate LOD tree [S1, unofficial]. The specific internal layout of this tree is not detailed in the sources, but it serves as the basis for the subsequent chunk tree build step [S1, unofficial]. The final output involves encoding opacity values, implying a mapping from the tree nodes to scalar opacity fields [S1, unofficial].

### Steps
1.  Remove invalid splats from the inner data structure [S1, unofficial].
2.  Compute the level-of-detail tree using the `tiny_lod` library, passing the inner data, `lod_base`, `merge_filter`, and a log callback [S1, unofficial].
3.  Encode the opacity values for the computed LOD structure [S1, unofficial].
4.  Build the chunk tree starting at index 0 with a log callback [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_tiny_lod(inner_data, lod_base, merge_filter, log_callback):
    """
    Computes the Tiny LOD hierarchy from volumetric data.
    """
    # Step 1: Remove invalid splats from the inner data structure
    clean_data = remove_invalid_splats(inner_data)
    
    # Step 2: Compute the level-of-detail tree
    # Not given in the sources. The aggregation logic is encapsulated in the tiny_lod library.
    lod_tree = compute_lod_tree(clean_data, lod_base, merge_filter, log_callback)
    
    # Step 3: Encode the opacity values for the computed LOD structure
    # Not given in the sources. Standard implementation quantizes or compresses scalar values.
    encoded_opacities = encode_opacity(lod_tree)
    
    # Step 4: Build the chunk tree starting at index 0 with a log callback
    # Not given in the sources. Organizes the tree into chunks for efficient access.
    chunk_tree = build_chunk_tree(lod_tree, start_index=0, log=log_callback)
    
    return lod_tree, chunk_tree

def remove_invalid_splats(data):
    # Not given in the sources. Filters out voxels or splats with invalid attributes.
    return data

def compute_lod_tree(data, lod_base, merge_filter, log_callback):
    # Not given in the sources. Aggregates voxels into a hierarchical tree based on lod_base and merge_filter.
    return {}

def encode_opacity(tree):
    # Not given in the sources. Maps tree nodes to scalar opacity fields.
    return []

def build_chunk_tree(tree, start_index, log_callback):
    # Not given in the sources. Partitions the tree into fixed-size chunks starting at start_index.
    return []
```

### Edge Cases and Pitfalls
*   **Invalid Data:** The method explicitly requires the removal of invalid splats before processing [S1, unofficial].
*   **Precision:** The `lod_base` parameter is a 32-bit float; extreme values relative to the voxel grid scale may affect LOD granularity [S1, unofficial].
*   **Merge Filter:** If `merge_filter` is enabled, a merge filter is applied during LOD tree computation [S1, unofficial].

## Subtopics

### [Trilinear Evaluation on Compressed Trees](7074-trilinear-evaluation-on-compressed-trees.md)
This method evaluates a signal value at an arbitrary point $\mathbf{x}$ within a compressed tree structure, supporting trilinear filtering and adaptive traversal. It serves as the evaluation engine for the Compressed Random-Access Tree (CRAT) framework, enabling fine-grain random queries required by rendering algorithms. Unlike block-based decompression schemes (e.g., DXT1) that rely on specialized hardware, this method uses adaptive tree traversal to reconstruct data on the fly, trading raw decompression speed for spatial coherence and high-quality filtering.

### [LODTree Storage Layout](16362-lodtree-storage-layout.md)
The LODTree storage layout is a sparse, hierarchical data structure designed to store volumetric data with multiple levels of detail (LOD) in a compact format, enabling efficient dynamic isosurface extraction. It differs from sibling methods like [Trilinear Evaluation on Compressed Trees](7074-trilinear-evaluation-on-compressed-trees.md) by explicitly managing virtual voxel expansion across discrete LOD levels using a bit-mapped cell representation to minimize storage overhead.

### Bhatt LOD
A variant of the LOD computation that uses the `bhatt_lod` library instead of `tiny_lod` [S1, unofficial].
#### Steps
1.  Remove invalid splats from the inner data structure [S1, unofficial].
2.  Compute the level-of-detail tree using the `bhatt_lod` library, passing the inner data, `lod_base`, and a log callback [S1, unofficial].
3.  Encode the opacity values for the computed LOD structure [S1, unofficial].
4.  Build the chunk tree starting at index 0 with a log callback [S1, unofficial].
#### Parameters
*   `lod_base` (f32): The base value for LOD computation [S1, unofficial].

### High LOD Approximation
The highest LOD level that retains local detail by using smaller subdivision blocks for dense nodes and per-voxel mapping for sparse nodes [S2, unofficial].
#### Definition
High LOD approximates the volumetric field by aggregating dense regions into 2×2×2 Gaussian blocks while preserving individual voxels in sparse regions [S2, unofficial].
#### Steps
1.  **For dense leaf nodes:** Subdivide into 2×2×2 blocks, aggregating every 8 adjacent voxels into a single Gaussian [S2, unofficial].
2.  **For dense leaf nodes:** Average the scalar values of the 8 voxels to compute the Gaussian’s opacity [S2, unofficial].
3.  **For dense leaf nodes:** Transform the block center in index space (offset by `voxelSize`) into world space to serve as the Gaussian’s position [S2, unofficial].
4.  **For dense leaf nodes:** Set covariance to be axis-aligned and correspond to the full voxel size along each axis [S2, unofficial].
5.  **For non-dense leaf nodes:** Treat at per-voxel level, converting each active voxel individually into a Gaussian primitive [S2, unofficial].
6.  **For non-dense leaf nodes:** Use voxel center (offset by half the `voxelSize`) as the mean and opacity taken directly from the voxel value [S2, unofficial].
7.  **For non-dense leaf nodes:** Set Gaussian radii to half the voxel size [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Efficiently reduces data size for dense volumes; supports adaptive detail retention via `lod_base` [S1, unofficial].
*   **Limitations:** Relies on external libraries (`tiny_lod`, `bhatt_lod`); specific performance metrics are not provided [S1, unofficial].
*   **When to use:** When a compact hierarchical representation is needed for subsequent chunk-based rendering or storage [S1, unofficial].

## Background
Tiny LOD is associated with the `sparkjsdev/spark` project, an unofficial implementation or wrapper for volumetric rendering techniques [S1, unofficial]. The method is part of a broader effort to integrate sparse voxel structures with Gaussian splatting techniques, as seen in related works like "3D Gaussian Particle Approximation of VDB Datasets" [S2, unofficial]. The `bhatt_lod` variant suggests a connection to Bhatt's work on LOD computation, though specific citations are not provided in the sources [S1, unofficial].

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Particle Approximation of VDB Datasets: A Study for Scientific Visualization, https://www.emergentmind.com/papers/2504.04857, fetched 2026-10-29 (unofficial)
