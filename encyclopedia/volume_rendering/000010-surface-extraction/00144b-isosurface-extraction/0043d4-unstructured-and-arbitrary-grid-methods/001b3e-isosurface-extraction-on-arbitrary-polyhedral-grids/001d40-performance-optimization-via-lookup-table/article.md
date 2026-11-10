# Performance Optimization via Lookup Table

Also called: Pre-computed connectivity

## Overview
This method accelerates isosurface extraction on arbitrary polyhedral grids by replacing on-the-fly topological analysis with rapid table lookups. It solves the problem of high computational cost in general cell geometries by pre-computing vertex connectivity for all possible sign configurations. Unlike the standard [Marching Cubes](52-marching-cubes.md) which relies on a fixed 256-entry table for cubes, this approach is geometry-agnostic, allowing the same optimization strategy to be applied to arbitrary cell types [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on an arbitrary polyhedral grid; an isovalue $\phi$.
*   **Output:** A set of triangles representing the isosurface.
*   **Data Structure:** A lookup table mapping each possible vertex sign configuration to a pre-computed list of vertex indices (connectivity) that form the isosurface segments within that cell.

**Pre-computation Phase:**
1.  Identify the cell geometry (e.g., cube, tetrahedron).
2.  Determine the number of vertices $N$ per cell.
3.  Generate all $2^N$ possible sign patterns of vertex values relative to the isovalue.
4.  For each pattern, compute the exact connectivity (which vertices are connected by the isosurface) using the specific geometric constraints of the cell.
5.  Store these connectivity lists in a lookup table indexed by the sign pattern integer.

**Extraction Phase:**
1.  Iterate through each cell in the grid.
2.  Evaluate the scalar field at each vertex of the cell.
3.  Determine the sign of each vertex relative to the isovalue ($+1$ if $> \phi$, $-1$ if $< \phi$, $0$ if $= \phi$).
4.  Construct an integer index from these signs (e.g., treating signs as bits).
5.  Retrieve the pre-computed connectivity list from the lookup table using this index.
6.  Generate triangles using the vertex indices from the retrieved list.

**Formulas:**
The connectivity is determined by solving for the intersection of the isosurface with the cell edges. For a linear interpolation, the intersection point $P$ on an edge between vertices $V_i$ and $V_j$ with values $\phi_i$ and $\phi_j$ is:
$$ P = V_i + \frac{\phi - \phi_i}{\phi_j - \phi_i} (V_j - V_i) $$
The lookup table stores the indices of vertices involved in these intersections.

**Parameters:**
*   **Cell Geometry:** Defines the size of the lookup table ($2^N$ entries). For a cube, $N=8$, resulting in 256 entries [S1].
*   **Threads:** Parallel execution using OpenMP (`-fopenmp`) can significantly speed up the extraction phase [S1].

**Edge Cases:**
*   **Degenerate Cases:** Vertices lying exactly on the isosurface ($\phi_i = \phi$) require consistent handling to avoid gaps or duplicates. The pre-computation must define a deterministic rule for these cases [S1].
*   **Precision:** Floating-point precision errors may cause sign flips. A small epsilon tolerance is often used when comparing vertex values to the isovalue.

## Subtopics

### [Tetrahedral Marching Cubes (5-tetrahedra method)](4547-tetrahedral-marching-cubes-5-tetrahedra-method.md)
Decomposes cubes into 5 tetrahedra, using a lookup table for 81 sign patterns. Deterministic and topology-agnostic.

### [Tetrahedral Marching Cubes Neighbor Lookup](11620-tetrahedral-marching-cubes-neighbor-lookup.md)
A data structure optimization for neighbor tetrahedron offsets, enhancing the lookup process in tetrahedral decompositions.

## Operating characteristics
*   **Time Complexity:** Pre-computation is $O(2^N \cdot C)$ where $C$ is the cost of computing connectivity for one cell type. Extraction is $O(1)$ per cell for table lookup, plus $O(V)$ for triangle generation.
*   **Memory Complexity:** $O(2^N \cdot K)$ where $K$ is the average number of vertices per connectivity list. For cubes, this is small (256 entries).
*   **Performance:** On an iMac Pro (2017, 2.3 GHz Intel Xeon W), using OpenMP with all available threads achieves a $\times 6$ speedup compared to single-threaded execution [S1]. Without pre-computation, the method is on average 1.8 times slower than optimized Marching Cubes on cubes [S1].
*   **Strengths:** Geometry-agnostic; fast extraction; deterministic.
*   **Limitations:** Pre-computation cost for complex cells; memory usage grows exponentially with vertex count per cell.

## Background
Originated in the context of generalizing isosurface extraction to arbitrary grids. The method was proposed to address the computational inefficiency of on-the-fly topological analysis for non-standard cell geometries [S1]. It builds on the principles of [Marching Cubes](52-marching-cubes.md) but extends the lookup table concept to arbitrary polyhedral cells.

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
