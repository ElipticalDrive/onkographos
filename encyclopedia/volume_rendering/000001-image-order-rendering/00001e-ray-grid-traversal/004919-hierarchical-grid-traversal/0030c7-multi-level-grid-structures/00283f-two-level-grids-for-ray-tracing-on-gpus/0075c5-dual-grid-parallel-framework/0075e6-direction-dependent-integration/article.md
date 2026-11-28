# Direction-Dependent Integration

Also called: Chunk-wise Direction-Dependent Integration, Raytheia Integration Algorithm

## Overview
Direction-Dependent Integration is a parallel algorithm for computing direction-dependent integrals by decoupling the source locations from the integration domain using a dual-grid structure [S1, unofficial]. It solves the problem of efficiently evaluating radiative transfer or similar integrals in large-scale astronomical simulations where the source distribution and the medium properties differ in resolution or topology [S1, unofficial]. Unlike monolithic ray-tracing methods, it partitions the contribution grid into chunks and processes them sequentially across MPI ranks, allowing each rank to accumulate partial contributions from all chunks [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A source grid defining ray-emitting elements (cells) and a contribution grid (AMR Cartesian grid) containing the quantity to be integrated $f_k$ [S1, unofficial].
- **Input:** $N_{ranks}$: Number of MPI ranks [S1, unofficial].
- **Input:** $N_s$: Number of source elements within each MPI rank that lie inside the molecular cloud [S1, unofficial].
- **Input:** $N_{rays}$: Number of rays emanated from each source element [S1, unofficial].
- **Output:** The direction-dependent integral $I(s, r)$ for each source element $s$ and ray $r$ [S1, unofficial].

**Data Structures:**
- **Source Grid:** Stores the locations of ray-emitting elements. The algorithm assumes a partitioning where each MPI rank holds a subset of source elements [S1, unofficial].
- **Contribution Grid:** An AMR Cartesian grid built from the source grid, providing a traversal-friendly representation [S1, unofficial]. It is partitioned into $N_{ranks}$ chunks [S1, unofficial].
- **Integration Variable:** $f_k$ represents the quantity to be integrated for contribution grid cell $k$ [S1, unofficial].
- **Path Length:** $r_{cell,k}$ is the path length of the ray through cell $k$ [S1, unofficial].

**Algorithm Steps:**
The algorithm operates on each MPI rank independently, accumulating the integral $I(s, r)$ over all contribution chunks [S1, unofficial].

1.  **Chunk Traversal:** Iterate over all contribution chunks $c$ from $0$ to $N_{ranks}-1$ [S1, unofficial].
2.  **Data Broadcast:** Broadcast the AMR data (spatial boundaries and integration variables) of contribution chunk $c$ to all MPI ranks [S1, unofficial].
3.  **Source Iteration:** Iterate over all source elements $s$ from $1$ to $N_s$ [S1, unofficial].
4.  **Validity Check:** Check if source element cell $s$ is inside the molecular cloud [S1, unofficial].
    -   If **not** inside: Set $I(s, r) = 0$ for all rays $r$ [S1, unofficial].
    -   If **inside**: Proceed to ray emission.
5.  **Ray Emission:** Iterate over all rays $r$ from $1$ to $N_{rays}$ [S1, unofficial]. Rays are emitted from the cell center of source element $s$ [S1, unofficial].
6.  **Path Determination:** Determine the ray-walking path within the current contribution chunk using a parametric traversal method [S1, unofficial].
7.  **Path Length Computation:** Compute the path length $r_{cell,k}$ of the ray through each cell $k$ in the chunk using a slab method [S1, unofficial].
8.  **Accumulation:** Update the integral:
    $$ I(s, r) = I(s, r) + f_k \cdot r_{cell,k} $$
    This accumulates the partial contribution of the current chunk [S1, unofficial].
9.  **Completion:** After traversing all chunks, $I(s, r)$ holds the complete direction-dependent integral [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize integral array I[s][r] to 0 for all s in 1..Ns and r in 1..Nrays

# Loop over all contribution chunks
for c in range(0, Nranks):
    # Broadcast AMR data of contribution chunk c to all MPI ranks
    # Data includes spatial boundaries (min/max coordinates) and integration variables f_k
    chunk_data = MPI_BCAST(chunk_c_data)
    
    # Loop over all source elements in this rank
    for s in range(1, Ns + 1):
        # Check if source element cell s is inside the molecular cloud
        if is_inside_molecular_cloud(source_cell[s]):
            # Loop over all rays emanated from the cell center
            for r in range(1, Nrays + 1):
                # Determine ray-walking path within the contribution chunk
                # Uses parametric traversal method
                path_cells = determine_ray_path(ray_origin[source_cell[s]], ray_direction[r], chunk_data)
                
                # Compute path length of the ray through each cell k in the path
                # Uses slab method
                for k in path_cells:
                    r_cell_k = compute_slab_path_length(ray, chunk_data.cell[k])
                    
                    # Accumulate partial contribution
                    # f_k is the quantity to be integrated of contribution grid cell k
                    I[s][r] = I[s][r] + chunk_data.f[k] * r_cell_k
        else:
            # If cell is not inside the molecular cloud, set integral to 0
            for r in range(1, Nrays + 1):
                I[s][r] = 0
```

**Parameters:**
- $N_{ranks}$: Number of MPI ranks [S1, unofficial].
- $N_s$: Number of source elements within each MPI rank inside the cloud [S1, unofficial].
- $N_{rays}$: Number of rays per source element [S1, unofficial].
- $f_k$: Quantity to be integrated for cell $k$ [S1, unofficial].
- $r_{cell,k}$: Path length through cell $k$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Outside Cloud:** Source elements outside the molecular cloud must explicitly set $I(s, r) = 0$ to avoid garbage values [S1, unofficial].
- **Chunk Boundaries:** The parametric traversal must correctly handle rays entering and exiting the contribution chunk boundaries [S1, unofficial].
- **Precision:** Accumulating partial contributions across chunks requires careful handling of floating-point precision if high accuracy is needed, though the algorithm assumes standard accumulation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The algorithm performs $N_{ranks}$ broadcasts and, for each rank, iterates over $N_s$ sources and $N_{rays}$ rays. The cost per ray depends on the number of cells traversed in the AMR grid. Total work is proportional to $N_{ranks} \times N_s \times N_{rays} \times (\text{avg cells per ray})$ [S1, unofficial].
- **Memory Complexity:** Each rank stores the source elements assigned to it and the current contribution chunk data [S1, unofficial]. The AMR data for one chunk is broadcast, so memory usage per rank is dominated by the source grid and the current chunk [S1, unofficial].
- **Performance:** The method is designed for high-performance ray-tracing in astronomical simulations, leveraging MPI for parallelism [S1, unofficial].
- **Strengths:** Decouples source and integration domains; efficient for AMR grids; scalable with MPI [S1, unofficial].
- **Limitations:** Requires broadcasting chunk data; sequential chunk processing per rank may limit parallelism if $N_{ranks}$ is large relative to work per chunk [S1, unofficial].

## Background
The method is part of the RAYTHEIA algorithm, a high-performance ray-tracing framework for three-dimensional direction-dependent equations in astronomical simulations [S1, unofficial]. It was introduced to handle the complexity of direction-dependent integrals in large-scale simulations where traditional methods struggle with performance and memory [S1, unofficial].

## Sources
[S1] RAYTHEIA: A high-performance ray-tracing algorithm for three-dimensional direction-dependent equations in astronomical simulations, https://arxiv.org/pdf/2605.09882, fetched 2026-10-24 (unofficial)
