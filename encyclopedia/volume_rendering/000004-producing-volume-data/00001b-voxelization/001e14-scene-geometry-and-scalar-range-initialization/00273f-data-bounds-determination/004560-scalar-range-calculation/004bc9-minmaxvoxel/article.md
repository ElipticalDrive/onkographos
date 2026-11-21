# MinMaxVoxel

Also called: Voxel Grid Bounding Box Calculation

## Overview
MinMaxVoxel computes the scalar value range (minimum and maximum) for non-regular volume data structures, specifically Adaptive Mesh Refinement (AMR) trees, to establish the global scalar range for visualization [S1, unofficial]. It iterates over active leaf nodes to find the extremal values, serving as the foundational step for transfer function mapping and normalization [S1, unofficial]. A related variant, Voxel Grid Bounding Box Calculation, determines the spatial extent (minimum and maximum voxel coordinates) of a dataset rather than its scalar values [S2, unofficial].

## How it works
The method operates on two distinct data types: scalar values within an AMR tree (primary method) and voxel coordinates (variant).

### Primary Method: Scalar Range Calculation
**Inputs and Outputs:**
*   Input: A reference to a `LeafArray` (specifically `openvdb::tree::LeafManager<TreeType>`) containing the active leaves of the volume tree [S1, unofficial].
*   Output: Two values, `mMin` and `mMax`, representing the global minimum and maximum scalar values found in the active voxels [S1, unofficial].

**Data Structures:**
The core state is held in a functor class with the following fields [S1, unofficial]:
*   `LeafArray& mLeafArray`: A reference to the leaf manager. Type: `openvdb::tree::LeafManager<TreeType>`.
*   `ValueType mMin`: The current minimum value. Type: `typename TreeType::ValueType`.
*   `ValueType mMax`: The current maximum value. Type: `typename TreeType::ValueType`.

**Algorithm Steps:**
1.  **Initialization:** Set `mMin` to the maximum possible value of `ValueType` (`std::numeric_limits<ValueType>::max()`) and `mMax` to the lowest possible value (`std::numeric_limits<ValueType>::lowest()`) [S1, unofficial].
2.  **Parallel Execution:** If parallel processing is enabled, invoke `tbb::parallel_reduce` over the range of leaves provided by `mLeafArray.getRange()` [S1, unofficial]. This splits the work into tasks.
3.  **Serial Execution:** If parallel processing is disabled, directly call the operator `(*this)(mLeafArray.getRange())` [S1, unofficial].
4.  **Leaf Iteration:** The operator iterates through each leaf index `n` from `range.begin()` to `range.end()` [S1, unofficial].
5.  **Value Iteration:** For each leaf `n`, obtain a constant iterator `iter` starting from the first active value using `mLeafArray.leaf(n).cbeginValueOn()` [S1, unofficial].
6.  **Value Processing:** Loop while `iter` is valid. Fetch the current value using `iter.getValue()` [S1, unofficial].
7.  **Update Extremes:** Update the local minimum and maximum:
    $$ mMin = \min(mMin, value) $$
    $$ mMax = \max(mMax, value) $$
    [S1, unofficial].
8.  **Join (Parallel Only):** When parallel tasks complete, merge results by comparing local extrema with the parent task's extrema:
    $$ mMin = \min(mMin, rhs.mMin) $$
    $$ mMax = \max(mMax, rhs.mMax) $$
    [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import tbb # Assuming TBB interface for parallel_reduce

class MinMaxVoxel:
    def __init__(self, leaf_array):
        """
        Initialize MinMaxVoxel for scalar range calculation.
        
        Args:
            leaf_array: openvdb::tree::LeafManager<TreeType> reference
        """
        self.mLeafArray = leaf_array
        # Initialize to extremes
        self.mMin = math.inf
        self.mMax = -math.inf

    def __call__(self, range_obj):
        """
        Serial execution of the min/max search over a range of leaf indices.
        
        Args:
            range_obj: Object with begin() and end() methods yielding leaf indices
        """
        for n in range(range_obj.begin(), range_obj.end()):
            # Get the leaf node
            leaf = self.mLeafArray.leaf(n)
            # Get iterator for active values
            iter = leaf.cbeginValueOn()
            
            # Iterate over active values
            while iter is not None:
                value = iter.getValue()
                if value < self.mMin:
                    self.mMin = value
                if value > self.mMax:
                    self.mMax = value
                iter.increment()

    def join(self, rhs):
        """
        Join results from a parallel task.
        
        Args:
            rhs: Another MinMaxVoxel instance from a parallel task
        """
        if rhs.mMin < self.mMin:
            self.mMin = rhs.mMin
        if rhs.mMax > self.mMax:
            self.mMax = rhs.mMax

def compute_scalar_range(leaf_array):
    """
    Compute the global scalar range for the volume.
    
    Args:
        leaf_array: openvdb::tree::LeafManager<TreeType>
        
    Returns:
        tuple: (min_value, max_value)
    """
    min_max_finder = MinMaxVoxel(leaf_array)
    
    # Check if parallel execution is desired/enabled
    # If parallel:
    #   tbb.parallel_reduce(leaf_array.getRange(), min_max_finder)
    # Else:
    min_max_finder(leaf_array.getRange())
    
    return (min_max_finder.mMin, min_max_finder.mMax)
```

**Parameters:**
*   `ValueType`: The scalar type stored in the tree (e.g., `float`, `double`). Typical values depend on the input data precision [S1, unofficial].
*   `LeafArray`: The container holding the tree leaves. Must support `leaf(n)` and `cbeginValueOn()` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Data:** If the volume contains no active voxels, `mMin` remains `inf` and `mMax` remains `-inf`. Downstream logic must handle this degenerate case, potentially defaulting to a zero range or raising an error [S1, unofficial].
*   **Precision:** Using `std::numeric_limits` ensures correct initialization for any floating-point type, avoiding hardcoded constants that might fail for `double` or `half` types [S1, unofficial].

### Variant: Voxel Grid Bounding Box Calculation
This variant computes the spatial bounding box (minimum and maximum voxel coordinates) rather than scalar values [S2, unofficial].

**Inputs and Outputs:**
*   Input: A slice/array of voxel coordinates (`IVec3`) [S2, unofficial].
*   Output: A tuple `(min_coord, max_coord)` of type `IVec3` [S2, unofficial].

**Algorithm Steps:**
1.  **Empty Check:** If the input voxel list is empty, return `(IVec3::ZERO, IVec3::ZERO)` [S2, unofficial].
2.  **Initialization:** Set `min` to `IVec3::MAX` and `max` to `IVec3::MIN` [S2, unofficial].
3.  **Iteration:** Loop over each voxel coordinate in the input list [S2, unofficial].
4.  **Update:** Update `min` and `max` using component-wise minimum and maximum operations:
    $$ min = \min(min, voxel) $$
    $$ max = \max(max, voxel) $$
    [S2, unofficial].
5.  **Return:** Return the final `(min, max)` tuple [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class IVec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
    
    def min(self, other):
        return IVec3(min(self.x, other.x), min(self.y, other.y), min(self.z, other.z))
    
    def max(self, other):
        return IVec3(max(self.x, other.x), max(self.y, other.y), max(self.z, other.z))

    @staticmethod
    def ZERO():
        return IVec3(0, 0, 0)

    @staticmethod
    def MAX():
        return IVec3(float('inf'), float('inf'), float('inf'))

    @staticmethod
    def MIN():
        return IVec3(float('-inf'), float('-inf'), float('-inf'))

def get_min_max(voxels):
    """
    Compute the bounding box of a set of voxel coordinates.
    
    Args:
        voxels: List of IVec3 objects
        
    Returns:
        tuple: (min_coord, max_coord)
    """
    if not voxels:
        return (IVec3.ZERO(), IVec3.ZERO())
    
    min_coord = IVec3.MAX()
    max_coord = IVec3.MIN()
    
    for voxel in voxels:
        min_coord = min_coord.min(voxel)
        max_coord = max_coord.max(voxel)
        
    return (min_coord, max_coord)
```

**Pitfalls:**
*   **Empty Input Handling:** Returning `(0,0,0)` for both min and max on empty input can cause downstream bounds calculations to produce a size of 1 instead of 0 if not handled separately [S2, unofficial].

## Subtopics

### Voxel Grid Bounding Box Calculation
Computes the minimum and maximum voxel coordinates occupied by a set of input voxels to determine the spatial extent of the dataset. It is chosen when the spatial bounds of a voxel set are needed, distinct from scalar range calculation. It handles empty inputs by returning `(0,0,0)` for both min and max, which requires careful downstream handling to avoid incorrect size calculations. See [Voxel Grid Bounding Box Calculation](11488-voxel-grid-bounding-box-calculation.md) for potential dedicated article if it exists.

## Operating characteristics
**Time and Memory Complexity:**
*   Time: $O(N)$, where $N$ is the number of active voxels (or leaves, depending on implementation granularity). The primary method iterates each active value exactly once [S1, unofficial]. The variant iterates each voxel coordinate once [S2, unofficial].
*   Memory: $O(1)$ auxiliary space for the primary method (storing only min/max) and the variant [S1, S2, unofficial]. The parallel version requires stack space for TBB tasks [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   The primary method supports parallel execution via TBB `parallel_reduce`, allowing linear scaling with the number of available cores for large leaf arrays [S1, unofficial].

**Strengths and Limitations:**
*   Strengths: Simple, robust, and parallelizable. Handles arbitrary scalar types via templates [S1, unofficial].
*   Limitations: Requires all active voxels to be visited. Does not handle inactive (background) values, which are typically ignored in AMR contexts [S1, unofficial].

**When to Use:**
*   Use for initializing scalar ranges for transfer functions in volume rendering pipelines [S1, unofficial].
*   Use the variant for spatial culling or bounding box calculations in voxel-based path tracing or ray marching [S2, unofficial].

## Background
The primary MinMaxVoxel method is implemented in the OpenVDB library, developed by the Academy Software Foundation (ASWF) [S1, unofficial]. It is part of the core tree utilities for managing AMR data structures [S1, unofficial]. The variant, Voxel Grid Bounding Box Calculation, is found in the `svo-rs` project, a Rust implementation of Sparse Voxel Octrees [S2, unofficial].

## Sources
[S1] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
[S2] mkmarek/svo-rs, https://github.com/mkmarek/svo-rs, fetched 2026-10-26 (unofficial)
