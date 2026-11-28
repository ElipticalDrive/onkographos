# MinMaxVoxel


## Overview
MinMaxVoxel computes the minimum and maximum scalar values for active voxels in sparse volume trees, establishing the global scalar range for visualization [S1, unofficial]. It iterates over active leaf nodes to find extremal values, serving as a foundational step for transfer function mapping and normalization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A reference to a `LeafArray` (`openvdb::tree::LeafManager<TreeType>`) containing the active leaves of the volume tree [S1, unofficial].
*   Output: Two values, `mMin` and `mMax`, representing the global minimum and maximum scalar values found in the active voxels [S1, unofficial].

**Data Structures:**
The core state is held in a functor class with the following fields [S1, unofficial]:
*   `LeafArray& mLeafArray`: A reference to the leaf manager. Type: `openvdb::tree::LeafManager<TreeType>`.
*   `ValueType mMin`: The current minimum value. Type: `typename TreeType::ValueType`.
*   `ValueType mMax`: The current maximum value. Type: `typename TreeType::ValueType`.

**Algorithm Steps:**
1.  **Initialization:** Set `mMin` to `std::numeric_limits<ValueType>::max()` and `mMax` to `std::numeric_limits<ValueType>::lowest()` [S1, unofficial].
2.  **Dispatch:** Parallel execution invokes `tbb::parallel_reduce(mLeafArray.getRange(), *this)` [S1, unofficial]. Serial execution directly calls `(*this)(mLeafArray.getRange())` [S1, unofficial].
3.  **Leaf Iteration:** The operator iterates `n` from `range.begin()` to `range.end()` [S1, unofficial].
4.  **Value Iteration:** For each `n`, obtains `iter = mLeafArray.leaf(n).cbeginValueOn()` [S1, unofficial].
5.  **Value Processing:** Inner loop `for (; iter; ++iter)` fetches `value = iter.getValue()` [S1, unofficial].
6.  **Update Extremes:** Updates `mMin = std::min(mMin, value)` and `mMax = std::max(mMax, value)` [S1, unofficial].
7.  **Join (Parallel Only):** When parallel tasks complete, merges results: `mMin = std::min(mMin, rhs.mMin)` and `mMax = std::max(mMax, rhs.mMax)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class MinMaxVoxel:
    def __init__(self, leaf_array):
        self.mLeafArray = leaf_array
        self.mMin = math.inf
        self.mMax = -math.inf

    def __call__(self, range_obj):
        for n in range(range_obj.begin(), range_obj.end()):
            leaf = self.mLeafArray.leaf(n)
            iter = leaf.cbeginValueOn()
            while iter:
                value = iter.getValue()
                if value < self.mMin:
                    self.mMin = value
                if value > self.mMax:
                    self.mMax = value
                iter.increment()

    def split(self, amount):
        # TBB split constructor: copies reference and reinitializes extremes
        self.mMin = math.inf
        self.mMax = -math.inf

    def join(self, rhs):
        if rhs.mMin < self.mMin:
            self.mMin = rhs.mMin
        if rhs.mMax > self.mMax:
            self.mMax = rhs.mMax

def compute_scalar_range(leaf_array):
    finder = MinMaxVoxel(leaf_array)
    # Parallel execution invokes tbb.parallel_reduce(leaf_array.getRange(), finder)
    # Serial execution directly calls finder(leaf_array.getRange())
    finder(leaf_array.getRange())
    return finder.mMin, finder.mMax
```

**Parameters:**
*   `ValueType`: The scalar type stored in the tree (e.g., `float`, `double`). Typical values depend on the input data precision [S1, unofficial].
*   `LeafArray`: The container holding the tree leaves. Must support `leaf(n)` and `cbeginValueOn()` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Data:** If the volume contains no active voxels, `mMin` remains `inf` and `mMax` remains `-inf`. Downstream logic must handle this degenerate case, potentially defaulting to a zero range or raising an error [S1, unofficial].
*   **Precision:** Using `std::numeric_limits` ensures correct initialization for any floating-point type, avoiding hardcoded constants that might fail for `double` or `half` types [S1, unofficial].

## Subtopics

### [Voxel Grid Bounding Box Calculation](14718-voxel-grid-bounding-box-calculation.md)
Computes the minimum and maximum voxel coordinates occupied by a set of input voxels to determine the spatial extent of the dataset. It returns inclusive index bounds `[[xmin, ymin, zmin], [xmax, ymax, zmax]]` where indices are absolute and may be negative. This method serves as the foundational step for determining the active region of sparse voxel data, enabling efficient memory allocation and culling in downstream volume rendering pipelines. Choose it when spatial bounds are required rather than scalar ranges. See [Voxel Grid Bounding Box Calculation](14718-voxel-grid-bounding-box-calculation.md).

## Operating characteristics
**Time and Memory Complexity:**
*   Time: $O(N)$, where $N$ is the number of active leaves or values. The method iterates each active value exactly once [S1, unofficial].
*   Memory: $O(1)$ auxiliary space for storing min/max values [S1, unofficial]. The parallel version requires stack space for TBB tasks [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   The method supports parallel execution via TBB `parallel_reduce`, allowing linear scaling with the number of available cores for large leaf arrays [S1, unofficial].

**Strengths and Limitations:**
*   Strengths: Simple, robust, and parallelizable. Handles arbitrary scalar types via templates [S1, unofficial].
*   Limitations: Requires visiting all active voxels. Does not handle inactive (background) values, which are typically ignored in AMR contexts [S1, unofficial].

**When to Use:**
*   Use for initializing scalar ranges for transfer functions in volume rendering pipelines [S1, unofficial].

## Background
The primary MinMaxVoxel method is implemented in the OpenVDB library, developed by the Academy Software Foundation (ASWF) [S1, unofficial]. It is part of the core tree utilities for managing AMR data structures [S1, unofficial].

## Sources
[S1] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
