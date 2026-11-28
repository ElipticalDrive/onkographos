# Customized Quicksort for Visibility Sorting
Also called: Customized Quicksort

## Overview
This method implements a specialized quicksort algorithm to sort volume rendering cells (tetrahedra) for visibility determination [S1, unofficial]. It addresses the performance bottleneck of generic sorting routines by optimizing data movement for the specific coupled data structures required by the visibility sorting task [S1, unofficial]. It is distinguished from generic library sorts (like `qsort`) by its ability to handle these structures efficiently, yielding a 75% to 89% improvement in speed [S1, unofficial].

## How it works
**Inputs:**
- A list of $N$ tetrahedral cells [S1, unofficial].
- For each cell, the precomputed tangential squared distance to the circumscribing sphere, denoted as $d^2$ [S1, unofficial].
- The tetrahedral index, denoted as $idx$ [S1, unofficial].

**Outputs:**
- A sorted list of tetrahedral indices ordered by increasing tangential squared distance $d^2$ [S1, unofficial].

**Data Structures:**
The algorithm operates on an array of structures or parallel arrays containing two fields per element:
1. `dist`: A `float` representing the tangential squared distance [S1, unofficial].
2. `index`: An integer representing the tetrahedral index [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Load the list of tetrahedra with their associated tangential squared distances and indices [S1, unofficial].
2. **Partitioning:** Select a pivot element from the current sub-array [S1, unofficial].
3. **Comparison:** Compare the `dist` field of elements against the pivot's `dist` [S1, unofficial].
4. **Swapping:** When swapping elements during partitioning, move both the `dist` (float) and the `index` (integer) together to maintain the association between the distance metric and the cell identifier [S1, unofficial].
5. **Recursion:** Recursively apply the sort to the sub-arrays created by the partition [S1, unofficial].
6. **Termination:** Stop when sub-arrays are of size 0 or 1 [S1, unofficial].

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
        pivot_index = partition(cells, low, high)
        customized_quicksort(cells, low, pivot_index - 1)
        customized_quicksort(cells, pivot_index + 1, high)

def partition(cells, low, high):
    """
    Partitions the array such that elements with dist <= pivot.dist are on the left,
    and elements with dist > pivot.dist are on the right.
    """
    pivot_dist = cells[high].dist
    pivot_index = high
    
    i = low - 1
    
    for j in range(low, high):
        if cells[j].dist <= pivot_dist:
            i = i + 1
            swap(cells, i, j)
            
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
- **Pivot Selection:** The proper choice of pivots gives an efficient sorting of sorted and nearly sorted lists [S1, unofficial]. Using the sorted values from the previous view speeds up the sorting, exploiting view coherence [S1, unofficial].
- **Precision:** Not given in the sources.
- **Degenerate Input:** Not given in the sources.

## Subtopics

### [View Coherence Optimization](14507-view-coherence-optimization.md)
Accelerates sorting by leveraging the similarity between consecutive camera viewpoints, using the previous frame's sorted order as a starting point. It further optimizes the sorting loop by employing a fast, parallel order-checking mechanism to exit early if the data is already sorted. This method builds on [Customized Quicksort for Visibility Sorting](14497-customized-quicksort-for-visibility-sorting.md) by reducing the workload that the quicksort must perform.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Provides a 75% to 89% improvement over the C library utility `qsort()` [S1, unofficial]. The sorting rate varies between 600,000 to 2 million cells per second depending on the hardware and scene complexity [S1, unofficial].
- **Strengths:** High efficiency due to optimized data movement (moving only two values: float and int) and avoidance of generic sorting overhead [S1, unofficial].
- **Limitations:** Not guaranteed to produce accurate visibility ordering unless the mesh is a Delaunay triangulation (inherited from the underlying Power Distance Sort method).

## Background
This method is described in the context of unstructured volume rendering, specifically for sorting tetrahedral cells based on tangential distances to circumscribing spheres, a technique originally proposed by Cignoni et al. and Karasick et al. [S1, unofficial]. The customized quicksort was developed to overcome the inefficiencies of generic sorting routines when handling the specific data structures required for this visibility sorting task [S1, unofficial].

## Sources
[S1] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
