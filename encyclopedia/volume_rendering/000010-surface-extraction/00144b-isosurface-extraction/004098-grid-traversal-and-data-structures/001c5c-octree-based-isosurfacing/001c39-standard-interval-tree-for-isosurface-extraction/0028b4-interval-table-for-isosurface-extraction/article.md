# Interval Table for Isosurface Extraction

Also called: Fast Implementation, TetIndex

## Overview
The Interval Table is a pre-processing data structure that maps scalar value intervals to tetrahedron IDs, allowing the extraction algorithm to skip tetrahedra whose function span does not contain the isovalue [S1, unofficial]. It differs from the Standard Interval Tree by using a fixed-size array of buckets (intervals) rather than a dynamic tree structure [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A tetrahedral mesh with scalar values at vertices; global minimum ($F_{min}$) and maximum ($F_{max}$) of the scalar field; desired table resolution ($N$) [S1, unofficial].
*   **Output:** A data structure (`TetIndex`) enabling retrieval of candidate tetrahedra for a given isovalue [S1, unofficial].

**Data Structures**
The structure is implemented as a class `TetIndex` containing protected member variables for bounds (global min/max), resolution (table size), and the table itself (vector of vectors of ints) [S1, unofficial].

**Numbered Steps**
1.  **Initialization:** Call `setBounds(F_min, F_max)` to store the global scalar range [S1, unofficial]. Call `setResolution(N)` to allocate the table with $N$ empty vectors; a reasonable default for $N$ is 500 [S1, unofficial].
2.  **Pre-processing (Build):** Iterate over all tetrahedra $T_i$ in the mesh [S1, unofficial]. Compute the function span $[t_{min}, t_{max}]$ for $T_i$ [S1, unofficial]. Call `addTet(tet_id, t_min, t_max)` to insert the ID into all relevant intervals in the table [S1, unofficial].
3.  **Runtime (Query):** Given an isovalue $I$, call `getCandidates(I)` to retrieve a pointer to the vector of tetrahedron IDs that intersect the isovalue [S1, unofficial].
4.  **Extraction:** In `IsoSurfacer::FastExtraction()`, iterate only over the tetrahedra returned by `getCandidates()` instead of the entire mesh [S1, unofficial].

**Formulas**
*   Interval width: $\Delta = (F_{max} - F_{min}) / N$ [G].
*   Start index: $i_{start} = \lfloor (t_{min} - F_{min}) / \Delta \rfloor$ [G].
*   End index: $i_{end} = \lfloor (t_{max} - F_{min}) / \Delta \rfloor$ [G].
*   Query index: $idx = \lfloor (I - F_{min}) / \Delta \rfloor$ [G].
*   Indices are clamped to $[0, N-1]$ to prevent out-of-bounds access [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class TetIndex:
    def __init__(self):
        self.global_min = 0.0
        self.global_max = 0.0
        self.resolution = 500
        self.table = []

    def set_bounds(self, f_min, f_max):
        """Store the global function range."""
        self.global_min = f_min
        self.global_max = f_max

    def set_resolution(self, n):
        """Set the size of the table and allocate empty vectors."""
        self.resolution = n
        self.table = [[] for _ in range(self.resolution)]

    def add_tet(self, tet_id, t_min, t_max):
        """Store the tetrahedron ID in the table based on its function span."""
        if t_min > t_max:
            return
        if self.global_max == self.global_min:
            return
        delta = (self.global_max - self.global_min) / self.resolution
        start_idx = math.floor((t_min - self.global_min) / delta)
        end_idx = math.floor((t_max - self.global_min) / delta)
        start_idx = max(0, min(start_idx, self.resolution - 1))
        end_idx = max(0, min(end_idx, self.resolution - 1))
        for k in range(start_idx, end_idx + 1):
            self.table[k].append(tet_id)

    def get_candidates(self, isovalue):
        """Given an isovalue, return a pointer (reference) to the corresponding vector of tetrahedron IDs."""
        if self.global_max == self.global_min:
            return self.table[0]
        delta = (self.global_max - self.global_min) / self.resolution
        idx = math.floor((isovalue - self.global_min) / delta)
        idx = max(0, min(idx, self.resolution - 1))
        return self.table[idx]

def fast_extraction(tet_index, isovalue, mesh):
    """Extract isosurface using the interval table."""
    candidates = tet_index.get_candidates(isovalue)
    triangles = []
    for tet_id in candidates:
        tet = mesh.get_tetrahedron(tet_id)
        # Standard marching tetrahedra: interpolate vertex values,
        # determine active edges, and generate triangles [G]
        triangles.extend(marching_tetrahedra(tet, isovalue))
    return triangles
```

**Parameters**
*   **Resolution ($N$):** The number of intervals in the table. A reasonable value is 500 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** Floating-point division and floor operations may cause boundary errors; clamping indices prevents out-of-bounds access [G].
*   **Degenerate Input:** If $F_{min} == F_{max}$, division by zero occurs in $\Delta$; this is handled by checking for constant scalar fields and skipping extraction or returning all tetrahedra [G].

## Subtopics

### [Fixed-Sized Buckets for Isosurface Extraction](1252-fixed-sized-buckets-for-isosurface-extraction.md)
Uses fixed-size buckets but may differ in bucket sizing strategy or hashing. Choose when memory is strictly constrained.

### [Interval Tree with Cell Grouping](1342-interval-tree-with-cell-grouping.md)
Groups cells to reduce tree depth. Choose for highly non-uniform scalar distributions.

### [Span Space Isosurface Extraction](1750-span-space-isosurface-extraction.md)
Operates in span space rather than value space. Choose for complex topological changes.

### [Out-of-core Interval Tree Extraction](2146-out-of-core-interval-tree-extraction.md)
Designed for datasets larger than RAM. Choose when memory is insufficient for in-memory structures.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method drastically reduces the number of visited tetrahedra compared to naive iteration or standard neighbor-visiting approaches, yielding faster extraction times than VTK's standard implementation in typical exercises [S1, unofficial].
*   **Strengths:** Simple implementation; fast query time; effective for uniform scalar distributions [G].
*   **Limitations:** Performance degrades if scalar values are highly non-uniform (many tets per bucket); fixed resolution requires tuning [G].

## Background
The Interval Table, also referred to as the "Fast Implementation" or "TetIndex," was introduced in visualization exercises by Julien Tierny [S1, unofficial]. It is designed to optimize isosurface extraction by pre-filtering tetrahedra based on their scalar range, avoiding unnecessary processing of non-intersecting cells [S1, unofficial].

## Sources
[S1] Julien Tierny - Visualization Exercise - Isosurface Extraction, https://julien-tierny.github.io/visualizationExerciseIso.html, fetched 2026-10-30 (unofficial)
