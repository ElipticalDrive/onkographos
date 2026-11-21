# Interval Table for Isosurface Extraction

Also called: Fast Implementation, TetIndex

## Overview
The Interval Table is a pre-processing data structure that maps scalar value intervals to tetrahedron IDs, allowing the extraction algorithm to skip tetrahedra whose function span does not contain the isovalue [S1, unofficial]. It solves the problem of inefficiently iterating over the entire mesh during isosurface extraction by providing a fast lookup mechanism for candidate cells. Unlike the naive implementation which loops over all tetrahedra, or the standard implementation which visits neighbors but still processes many non-intersecting cells, the interval table allows skipping tetrahedra entirely based on their scalar range, drastically reducing the number of visited tetrahedra [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh with associated scalar values at vertices; a global scalar range $[F_{min}, F_{max}]$; a resolution parameter $R$ (number of intervals).
*   **Output:** A data structure (`TetIndex`) enabling $O(1)$ or $O(\log R)$ retrieval of tetrahedron IDs intersecting a given isovalue $v_{iso}$.

**Data Structures:**
The structure is implemented as a class `TetIndex` containing:
1.  `bounds`: A pair of floats $(F_{min}, F_{max})$ representing the global minimum and maximum of the scalar field.
2.  `resolution`: An integer $R$, the number of intervals in the table.
3.  `table`: A vector of vectors of integers (`std::vector<std::vector<int>>`). The outer vector has size $R$. Each inner vector stores the IDs of tetrahedra whose scalar span overlaps with the corresponding interval.

**Numbered Steps:**

1.  **Initialization:** Call `setBounds(F_min, F_max)` to store the global scalar range. Call `setResolution(R)` to allocate the `table` with $R$ empty vectors. A reasonable default for $R$ is 500 [S1, unofficial].
2.  **Pre-processing (Population):** Iterate over all tetrahedra $T$ in the input mesh.
    a. Compute the function span $[t_{min}, t_{max}]$ for tetrahedron $T$, where $t_{min} = \min_{v \in T} f(v)$ and $t_{max} = \max_{v \in T} f(v)$.
    b. Determine which intervals in the table overlap with $[t_{min}, t_{max}]$.
    c. For each overlapping interval index $i$, append the tetrahedron ID to `table[i]`.
3.  **Query (Runtime):** Given an isovalue $v_{iso}$:
    a. Calculate the interval index $i$ corresponding to $v_{iso}$ using the formula:
       $$ i = \left\lfloor \frac{v_{iso} - F_{min}}{F_{max} - F_{min}} \times R \right\rfloor $$
       Clamp $i$ to $[0, R-1]$ if necessary.
    b. Call `getCandidates(v_{iso})` to retrieve a pointer to `table[i]`.
4.  **Extraction:** Iterate only over the tetrahedron IDs returned by `getCandidates()` in the extraction loop (e.g., `IsoSurfacer::FastExtraction()`), skipping all other tetrahedra.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class TetIndex:
    def __init__(self):
        self.F_min = 0.0
        self.F_max = 0.0
        self.R = 500
        self.table = [] # Vector of vectors of ints

    def setBounds(self, F_min, F_max):
        self.F_min = F_min
        self.F_max = F_max
        # Ensure non-zero range to avoid division by zero
        if self.F_max == self.F_min:
            self.F_max = self.F_min + 1.0

    def setResolution(self, R):
        self.R = R
        self.table = [[] for _ in range(R)]

    def addTet(self, tet_id, t_min, t_max):
        # Map t_min and t_max to interval indices
        # Index calculation: (value - F_min) / (F_max - F_min) * R
        idx_min = int((t_min - self.F_min) / (self.F_max - self.F_min) * self.R)
        idx_max = int((t_max - self.F_min) / (self.F_max - self.F_min) * self.R)
        
        # Clamp indices to valid range [0, R-1]
        idx_min = max(0, min(idx_min, self.R - 1))
        idx_max = max(0, min(idx_max, self.R - 1))
        
        # Add tet_id to all overlapping intervals
        for i in range(idx_min, idx_max + 1):
            self.table[i].append(tet_id)

    def getCandidates(self, v_iso):
        idx = int((v_iso - self.F_min) / (self.F_max - self.F_min) * self.R)
        idx = max(0, min(idx, self.R - 1))
        return self.table[idx]

# Usage in IsoSurfacer::FastExtraction
def FastExtraction(tet_index, v_iso):
    candidates = tet_index.getCandidates(v_iso)
    for tet_id in candidates:
        process_tetrahedron(tet_id)
```

**Parameters:**
*   **Resolution ($R$):** The number of intervals in the table. A reasonable value is 500 [S1, unofficial]. Higher values increase precision but also memory usage and construction time.

**Edge Cases and Pitfalls:**
*   **Zero Range:** If $F_{min} == F_{max}$, division by zero occurs in index calculation. The implementation must handle this, e.g., by setting $F_{max} = F_{min} + \epsilon$ [G].
*   **Boundary Clamping:** The calculated index must be clamped to $[0, R-1]$ to handle isovalue precision errors or values slightly outside the computed global bounds [G].
*   **Duplicate IDs:** A tetrahedron may be added to multiple intervals if its span covers multiple bins. This is correct behavior but increases memory usage.

## Subtopics

### Tetrahedron Range Skip Optimization
An optimization to reduce the number of tetrahedra traversed during isosurface extraction by skipping tetrahedra that cannot contain the isovalue [S2]. It stores the range $[v_{T,min}, v_{T,max}]$ of function values for each tetrahedron $T$ [S2]. Before subdivision, it checks if the isovalue $v_{iso}$ falls into the range $[v_{T,min}, v_{T,max}]$ [S2]. If $v_{iso} \notin [v_{T,min}, v_{T,max}]$, the tetrahedron is skipped regardless of error metrics [S2]. This requires storing two additional values ($v_{T,min}, v_{T,max}$) per tetrahedron but typically reduces the number of tetrahedra traversed by up to 50% [S2].

## Operating characteristics
*   **Time Complexity:** Pre-processing is $O(N \cdot K)$ where $N$ is the number of tetrahedra and $K$ is the average number of intervals a tetrahedron spans. Query time is $O(1)$ to find the interval and $O(k)$ to iterate over candidates, where $k$ is the number of tetrahedra intersecting the isovalue.
*   **Memory Complexity:** $O(N \cdot K + R)$, where $R$ is the resolution. The table stores references to tetrahedra, potentially duplicating IDs across intervals.
*   **Measured Performance:** The method drastically reduces the number of visited tetrahedra compared to naive or standard neighbor-visiting approaches, yielding faster extraction times [S1, unofficial]. In specific benchmarks, it has been shown to beat VTK implementations [S1, unofficial].
*   **Strengths:** Significant speedup for isosurface extraction by avoiding unnecessary geometric processing of non-intersecting cells.
*   **Limitations:** Requires pre-processing time and additional memory. Performance depends on the distribution of scalar values; if the isovalue is common to many tetrahedra, the candidate list may still be large.
*   **When to use:** When extracting isosurfaces from large tetrahedral meshes where the isovalue is known or fixed, and pre-processing time is acceptable.
*   **When not to use:** For dynamic meshes where the scalar field changes frequently, as the table would need constant rebuilding.

## Background
The Interval Table was introduced as part of a visualization exercise for isosurface extraction, aiming to provide a faster alternative to standard VTK implementations [S1, unofficial]. It is based on the concept of mapping scalar ranges to data structures to optimize search operations. The related Tetrahedron Range Skip Optimization was described in the context of adaptive multi-valued volume data visualization using data-dependent error metrics, highlighting the benefit of range checks in reducing traversal counts [S2].

## Sources
[S1] Julien Tierny - Visualization Exercise - Isosurface Extraction, https://julien-tierny.github.io/visualizationExerciseIso.html, fetched 2026-10-30 (unofficial)
[S2] Adaptive Multi-Valued Volume Data Visualization using Data-Dependent Error Metrics, https://doi.org/10.2316/journal.205.2005.2.205-4291, fetched 2026-10-26
