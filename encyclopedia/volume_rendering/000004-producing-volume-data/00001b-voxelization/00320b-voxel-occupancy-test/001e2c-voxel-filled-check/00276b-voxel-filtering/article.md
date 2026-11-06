# Voxel Filtering

Also called: Voxel Predicate Filter

## Overview
Voxel Filtering iterates over a voxel grid to identify and return the linear indices of voxels that satisfy a specific user-defined condition [S1, unofficial]. It serves as a general-purpose selection mechanism within the voxelization pipeline, acting as a post-processing step for [Voxel Filled Check](1705-voxel-filled-check.md) to refine occupancy data [S1, unofficial]. Unlike [Voxel Material Classification](3645-voxel-material-classification.md), which categorizes voxels by material type, Voxel Filtering selects voxels based on arbitrary boolean predicates, enabling tasks such as density thresholding or invalid value exclusion [S1, unofficial].

## How it works
**Inputs:**
- A voxel data array (linear or multi-dimensional) of size $N$.
- A predicate function $P(v)$ that accepts a voxel value $v$ and returns a boolean.

**Outputs:**
- A list (or vector) of linear indices $i$ where $P(v_i)$ is true.

**Data Structures:**
- **Input Array:** A contiguous memory block of voxel values. The layout is assumed to be linearized (e.g., row-major or column-major) such that index $i$ corresponds to a unique voxel.
- **Output Vector:** A dynamic array of integers storing the valid linear indices.

**Steps:**
1. Initialize an empty output vector $L$.
2. Iterate through the input data array from index $i = 0$ to $N-1$.
3. For each index $i$, retrieve the voxel value $v_i$.
4. Evaluate the predicate function: $result = P(v_i)$.
5. If $result$ is true, append $i$ to $L$.
6. After the loop completes, return $L$.

**Parameters:**
- `func`: A callable predicate that takes a single voxel (CellType) as input and returns a boolean indicating whether the voxel satisfies the condition [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_filter(data_array, predicate_func):
    """
    Filters a voxel grid based on a predicate function.
    
    Args:
        data_array: List or array of voxel values.
        predicate_func: Function that takes a voxel value and returns a boolean.
        
    Returns:
        List of linear indices where predicate_func returns True.
    """
    output_indices = []
    num_voxels = len(data_array)
    
    for i in range(num_voxels):
        voxel_value = data_array[i]
        if predicate_func(voxel_value):
            output_indices.append(i)
            
    return output_indices
```

**Edge Cases and Pitfalls:**
- **Empty Result:** If no voxels satisfy the predicate, the output list is empty. The caller must handle this to avoid rendering errors or division by zero in subsequent steps.
- **Predicate Cost:** If the predicate function is computationally expensive, this linear scan becomes a bottleneck. [G] Consider caching predicate results if the voxel data does not change between frames.
- **Boundary Conditions:** The method assumes the input array is fully populated. If the grid has padding or sparse regions, the linear index mapping must be consistent with the coordinate system used by the predicate.

## Subtopics

### [Sample Uniform and Occupied Cells](4651-sample-uniform-and-occupied-cells.md)
This variant generates a mixed set of voxel indices for volume rendering by combining uniformly sampled cells with cells known to be occupied by geometry. It specifically filters out cells with invalid density values (such as -1) to ensure valid rendering data. This approach balances coverage of empty space with detail in occupied regions.

### Volume Subset Extraction
This method extracts a subset of voxels based on a transfer function evaluation. For each voxel, it computes a transfer value $t = F(V, p)$, where $V$ is the gradient and $p$ is the density of the voxel [S2, unofficial]. The voxel is included in the subset if $t$ exceeds a user-defined threshold [S2, unofficial]. The extracted subset is stored as a 2D array of splat rows, where each row contains the extracted voxels [S2, unofficial]. Each voxel in the subset stores its location and normal vector, as rows may contain different numbers of voxels and voxels are not necessarily contiguous [S2, unofficial]. The data structure supports traversal in either back-to-front or front-to-back order [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the total number of voxels in the grid, as every voxel must be visited to evaluate the predicate [S1, unofficial].
**Memory Complexity:** $O(K)$, where $K$ is the number of voxels satisfying the predicate. In the worst case ($K=N$), this is $O(N)$.
**Performance:** Performance is dominated by the cost of the predicate function and memory bandwidth for reading the input array. [G] For large grids, parallelization (e.g., using GPU kernels or multi-threading) is recommended to reduce latency.
**Strengths:** Simple implementation, flexible predicate definition, and efficient linear scan.
**Limitations:** Requires full traversal of the grid; does not exploit spatial coherence or hierarchical structures (like octrees) for early culling.

## Background
Voxel Filtering is a fundamental operation in voxel-based rendering pipelines, often used in conjunction with densification and pruning strategies in 3D Gaussian Splatting variants [S1, unofficial]. It allows for dynamic adjustment of voxel sets based on optimization iterations or user-defined criteria [S1, unofficial]. The concept of extracting subsets based on transfer functions is also found in volume splatting techniques, where antialiasing and detail preservation are managed by filtering voxels based on gradient and density properties [S2, unofficial].

## Sources
[S1] 3DGS-FLoD/flod, https://github.com/3DGS-FLoD/flod, fetched 2026-10-27 (unofficial)
[S2] DTIC ADA606764: Splatting Errors and Antialiasing, https://archive.org/details/DTIC_ADA606764, fetched 2026-10-26 (unofficial)
