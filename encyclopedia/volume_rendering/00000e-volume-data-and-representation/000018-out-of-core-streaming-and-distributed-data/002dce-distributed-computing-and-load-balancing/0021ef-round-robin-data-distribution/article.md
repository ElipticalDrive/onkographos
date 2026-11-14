# Round-Robin Data Distribution

Also called: Round-Robin Cell Distribution, Round-Robin Static Load Balancing, Round-robin distribution, Static Load Balanced Partitioning, Static Load Balancing via Round Robin

## Overview
Round-Robin Data Distribution is a static load balancing strategy for distributed volume rendering that assigns data cells to processors in a cyclic order to average out computational requirements [S2, unofficial]. It solves the problem of load imbalance caused by variations in cell size, shape, and opacity by dispersing connected cells widely among processors, rather than grouping them spatially [S2, unofficial]. Unlike spatial partitioning methods that exploit locality, this method prioritizes computational balance over communication efficiency, serving as a baseline for static distribution in unstructured grid rendering [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A set of $N$ mesh cells (e.g., tetrahedra) and $P$ processors.
*   **Output**: An assignment map $A: \{0, \dots, N-1\} \rightarrow \{0, \dots, P-1\}$ mapping each cell index to a processor ID.

**Data Structures**
*   **Cell**: Stored as 16 bytes, containing four node indices [S2, unofficial][S3].
*   **Node**: Stored as 16 bytes, containing three coordinates (floats) and one scalar value (float) [S2, unofficial][S3].
*   **Assignment Array**: An array of size $N$ storing processor IDs, or computed on-the-fly.

**Steps**
1.  **Index Generation**: Generate a sequential list of cell indices $i = 0, 1, \dots, N-1$.
2.  **Cyclic Assignment**: For each cell $i$, assign it to processor $p$ using the modulo operation:
    $$ p = i \pmod P $$
    Where $P$ is the total number of processors [S2, unofficial].
3.  **Data Distribution**: Transfer the cell data (indices) and the required node data (coordinates and scalars) to the assigned processors. Connectivity data is not required for the distribution logic [S3].

**Formulas**
*   **Assignment**: $p_i = i \pmod P$
*   **Worst-Case Data Transfer Volume**: If no node information is shared between cells assigned to a processor, the maximum data transferred for $n$ cells is:
    $$ V_{max} = n \times (\text{size}_{cell} + 4 \times \text{size}_{node}) $$
    Substituting the values from [S2, unofficial][S3]:
    $$ V_{max} = n \times (16 + 4 \times 16) = 80n \text{ bytes} $$

**Parameters**
*   **Cell Storage Size**: 16 bytes [S2, unofficial][S3].
*   **Node Storage Size**: 16 bytes [S2, unofficial][S3].
*   **Typical Values**: In an example with $N=1,000,000$ cells and $P=128$ processors, each processor receives $n = 1,000,000 / 128 = 7,812.5$ cells (average). The average data transferred is $7,812.5 \times 80 = 625,000$ bytes, cited as 640,000 bytes in [S3] (likely accounting for integer division or specific dataset alignment).

**Edge Cases and Pitfalls**
*   **Precision**: The modulo operation is exact for integer indices.
*   **Boundaries**: If $N$ is not divisible by $P$, the first $N \pmod P$ processors receive $\lfloor N/P \rfloor + 1$ cells, and the remaining processors receive $\lfloor N/P \rfloor$ cells.
*   **Degenerate Input**: The method assumes valid cell indices. It does not handle empty cells or invalid topology.
*   **Communication Overhead**: The worst-case scenario assumes no node sharing. In practice, nodes are often shared by multiple cells, reducing the actual transfer volume. However, the round-robin distribution minimizes spatial locality, potentially increasing the number of unique nodes required per processor compared to spatial partitioning [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def round_robin_distribution(num_cells, num_processors):
    """
    Assigns cells to processors in a round-robin fashion.
    
    Args:
        num_cells (int): Total number of cells in the dataset.
        num_processors (int): Total number of processors.
        
    Returns:
        list: A list of processor IDs for each cell index.
    """
    assignments = []
    for i in range(num_cells):
        processor_id = i % num_processors
        assignments.append(processor_id)
    return assignments

def calculate_worst_case_transfer_volume(num_cells_per_processor):
    """
    Calculates the worst-case data transfer volume assuming no node sharing.
    
    Args:
        num_cells_per_processor (int): Number of cells assigned to a single processor.
        
    Returns:
        int: Volume in bytes.
    """
    cell_size = 16  # bytes
    node_size = 16  # bytes
    nodes_per_cell = 4
    volume = num_cells_per_processor * (cell_size + nodes_per_cell * node_size)
    return volume
```

## Subtopics

### [Static Round-Robin Partitioning](2728-static-round-robin-partitioning.md)
Partitions volume into small subvolumes distributed round-robin to average projected area and element count. Chosen when subvolume granularity is preferred over cell-level assignment.

### [Scattered Assignment](4204-scattered-assignment.md)
Assigns mesh cells in a scattered fashion for Light Path Mapping, prioritizing load balance over communication. Chosen for unstructured grids where neighboring cells have similar workloads.

### [Randomized Data Decomposition](11592-randomized-data-decomposition.md)
Assigns data blocks uniformly at random to achieve statistical load balance via the Law of Large Numbers. Chosen for out-of-core rendering where static load estimation is difficult.

## Operating characteristics
*   **Time Complexity**: $O(N)$ for the assignment phase, where $N$ is the number of cells. The assignment is trivial and requires no complex computation or sorting [S2, unofficial].
*   **Memory Complexity**: $O(N)$ to store the assignment map, or $O(1)$ if computed on-the-fly during data transfer.
*   **Measured Performance**: On an IBM SP2 with 128 processors, a dataset of 500,000 tetrahedral cells achieved 2 frames per second for a 400x400-pixel image [S2, unofficial].
*   **Scaling**: The method scales well with the number of processors because the assignment logic is independent of the data structure's complexity. However, communication overhead may increase if the lack of spatial locality leads to inefficient data sharing [S2, unofficial].
*   **Strengths**: Simple to implement; provides approximate load balance for large datasets; flexible for any number of processors without expensive rebalancing [S2, unofficial].
*   **Limitations**: Destroys spatial relationships between cells, which can hinder optimizations that rely on locality (e.g., early ray termination, efficient compositing) [S3]. Not suitable for small datasets where statistical averaging fails to balance load [S2, unofficial].
*   **When to Use**: When computational load varies significantly due to cell properties (size, opacity) and spatial grouping would exacerbate imbalance. Ideal for cell-projection rendering on unstructured grids [S2, unofficial].
*   **When Not to Use**: When spatial locality is critical for performance (e.g., ray-casting with heavy coherence) or when communication bandwidth is severely constrained and node sharing is high [S2, unofficial].

## Background
Round-Robin Data Distribution was proposed in the context of scalable parallel cell-projection volume rendering for three-dimensional unstructured data on massively parallel distributed-memory supercomputers, such as the IBM SP2 [S2, unofficial]. The method was introduced to address the limitations of spatial partitioning in unstructured grids, where connected cells often have similar sizes and opacities, leading to load imbalances if grouped together [S2, unofficial]. It serves as a static baseline for comparison against dynamic load balancing approaches like LBC and LBAC in isosurface extraction and volume rendering workflows [S1]. The approach relies on the principle that with a sufficiently large number of cells, the computational requirements for each processor will average out, producing an approximate load balance [S2, unofficial].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
[S2] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S3] PDF A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
