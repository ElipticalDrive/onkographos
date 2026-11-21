# Customized Quicksort for Visibility Sorting
Also called: Customized Quicksort

## Overview
This method implements a specialized quicksort algorithm to sort volume rendering cells (tetrahedra) for visibility determination. It addresses the performance bottleneck of generic sorting routines by optimizing data movement for the specific structure required by power distance sorting. It is distinguished from generic library sorts (like `qsort`) by its ability to handle the coupled data structures of tangential distance and cell index efficiently, yielding a 75% to 89% improvement in speed [S1, unofficial].

## How it works
**Inputs:**
- A list of $N$ tetrahedral cells.
- For each cell, the precomputed tangential squared distance to the circumscribing sphere, denoted as $d^2$.
- The tetrahedral index, denoted as $idx$.

**Outputs:**
- A sorted list of tetrahedral indices ordered by increasing tangential squared distance $d^2$.

**Data Structures:**
The algorithm operates on an array of structures or parallel arrays containing two fields per element:
1. `dist`: A `float` representing the tangential squared distance.
2. `index`: An integer representing the tetrahedral index.

**Algorithm Steps:**
1. **Initialization:** Load the list of tetrahedra with their associated tangential squared distances and indices.
2. **Partitioning:** Select a pivot element from the current sub-array. The pivot selection strategy is critical for performance on nearly sorted data (see View Coherence Optimization).
3. **Comparison:** Compare the `dist` field of elements against the pivot's `dist`.
4. **Swapping:** When swapping elements during partitioning, move both the `dist` (float) and the `index` (integer) together to maintain the association between the distance metric and the cell identifier.
5. **Recursion:** Recursively apply the sort to the sub-arrays created by the partition.
6. **Termination:** Stop when sub-arrays are of size 0 or 1.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def customized_quicksort(cells, low, high):
    """
    Sorts the cells array in-place based on the 'dist' field.
    cells: List of objects/tuples with attributes 'dist' (float) and 'index' (int).
    low: Start index of the sub-array.
    high: End index of the sub-array.
    """
    if low < high:
        # Partition the array and get the pivot index
        pivot_index = partition(cells, low, high)
        
        # Recursively sort elements before and after partition
        customized_quicksort(cells, low, pivot_index - 1)
        customized_quicksort(cells, pivot_index + 1, high)

def partition(cells, low, high):
    """
    Partitions the array such that elements with dist <= pivot.dist are on the left,
    and elements with dist > pivot.dist are on the right.
    """
    # Select pivot. For view coherence, the last element is often efficient if data is nearly sorted.
    pivot_dist = cells[high].dist
    pivot_index = high
    
    i = low - 1  # Index of smaller element
    
    for j in range(low, high):
        # Compare only the distance field
        if cells[j].dist <= pivot_dist:
            i = i + 1
            # Swap both dist and index to keep them coupled
            swap(cells, i, j)
            
    # Place pivot in correct position
    swap(cells, i + 1, high)
    return i + 1

def swap(cells, i, j):
    """
    Swaps two elements in the cells array.
    Moves both the tangential squared distance (float) and the tetrahedral index (int).
    """
    temp_dist = cells[i].dist
    temp_index = cells[i].index
    
    cells[i].dist = cells[j].dist
    cells[i].index = cells[j].index
    
    cells[j].dist = temp_dist
    cells[j].index = temp_index
```

**Parameters and Edge Cases:**
- **Pivot Selection:** The choice of pivot significantly impacts performance. Standard random or median-of-three pivots are common, but for view coherence, using the last element of the previous frame's sorted order is effective [S1, unofficial].
- **Precision:** The distance is stored as a `float`. Precision issues may arise if distances are extremely close, potentially affecting strict visibility ordering.
- **Degenerate Input:** If the input is already sorted or nearly sorted, a naive quicksort may degrade to $O(N^2)$ without proper pivot selection or hybridization (e.g., switching to insertion sort for small sub-arrays). The custom implementation exploits view coherence to mitigate this [S1, unofficial].

## Subtopics

### [View Coherence Optimization](14507-view-coherence-optimization.md)
Accelerates sorting by leveraging the similarity between consecutive camera viewpoints, using the previous frame's sorted order as a starting point.

## Operating characteristics
- **Time Complexity:** Average case $O(N \log N)$. Best case $O(N \log N)$ or better with view coherence on nearly sorted data. Worst case $O(N^2)$ if pivot selection is poor and data is structured adversarially, though mitigated by the custom implementation's design.
- **Memory Complexity:** $O(\log N)$ stack space for recursion. In-place sorting requires no additional memory for the data structures.
- **Performance:** Provides a 75% to 89% improvement over the C library utility `qsort()` [S1, unofficial]. The rate varies between 600,000 to 2 million cells per second depending on the hardware and scene complexity [S1, unofficial].
- **Strengths:** High efficiency due to optimized data movement (moving only two values: float and int) and avoidance of generic sorting overhead.
- **Limitations:** Not guaranteed to produce accurate visibility ordering unless the mesh is a Delaunay triangulation (inherited from the underlying Power Distance Sort method).

## Background
This method is described in the context of unstructured volume rendering, specifically for sorting tetrahedral cells based on tangential distances to circumscribing spheres, a technique originally proposed by Cignoni et al. and Karasick et al. [S1, unofficial]. The customized quicksort was developed to overcome the inefficiencies of generic sorting routines when handling the specific data structures required for this visibility sorting task.

## Sources
[S1] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
