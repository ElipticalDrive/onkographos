# Round-Robin Data Distribution

Also called: Round-Robin Cell Distribution, Round-Robin Static Load Balancing, Round-robin distribution, Static Load Balanced Partitioning, Static Load Balancing via Round Robin

## Overview
Round-Robin Data Distribution is a static load balancing strategy for parallel volume rendering that assigns data cells to processors in a cyclic order to average out computational requirements [S2, unofficial][S3]. It solves the problem of load imbalance caused by variations in cell size, shape, and opacity by dispersing connected cells widely among processors, thereby avoiding the exacerbation of workload disparities that occurs when similar cells are grouped together [S2, unofficial]. Unlike methods that prioritize spatial locality or communication minimization, this approach prioritizes computational balance, relying on statistical averaging to achieve approximate load balance [S2, unofficial][S3].

## How it works
**Inputs and Outputs**
*   **Input**: A set of $N$ data cells (e.g., tetrahedral cells in an unstructured mesh) and $P$ processors.
*   **Output**: An assignment map where each cell is associated with a specific processor ID.

**Data Structures**
The method operates on the raw cell and node data. The specific storage layout required for the worst-case communication analysis is:
*   **Cell**: 16 bytes, containing four node indices.
*   **Node**: 16 bytes, containing three coordinates (floats) and one scalar value.

**Steps**
1.  **Cell Enumeration**: Assign a unique sequential index $i$ to each cell in the dataset, where $0 \le i < N$.
2.  **Processor Assignment**: For each cell $i$, determine the target processor $p$ using the modulo operation:
    $$ p = i \mod P $$
    where $P$ is the total number of processors.
3.  **Data Distribution**: Transfer the cell data and the necessary node data to the assigned processor.
    *   The cell data (16 bytes) is always transferred.
    *   The node data is transferred if not already present on the processor. In the worst-case scenario where no node information is shared between cells assigned to the same processor, all nodes for all cells must be transferred.

**Formulas**
*   **Assignment**: $p_i = i \mod P$
*   **Worst-Case Data Transfer Volume**: If no node sharing occurs, the maximum data transferred to a single processor for $n$ cells is:
    $$ V_{max} = n \times (\text{size}_{cell} + 4 \times \text{size}_{node}) $$
    Substituting the values from [S2, unofficial][S3]:
    $$ V_{max} = n \times (16 + 4 \times 16) = 80n \text{ bytes} $$

**Parameters**
*   **Cell Storage Size**: 16 bytes [S2, unofficial][S3].
*   **Node Storage Size**: 16 bytes [S2, unofficial][S3].
*   **Typical Values**: In an example with 1 million cells distributed across 128 processors, each processor receives approximately $1,000,000 / 128 \approx 7,812$ cells. The average data transferred per processor is $80 \times 7,812 \approx 624,960$ bytes, cited as 640,000 bytes in [S3] (likely accounting for rounding or specific dataset characteristics).

**Edge Cases and Pitfalls**
*   **Loss of Spatial Locality**: The method destroys the spatial relationship between mesh cells, making an unstructured dataset appear more regular [S3]. This complicates optimizations that rely on spatial coherence, such as early ray termination or efficient neighbor lookups, unless a secondary spatial partitioning tree is used [S2, unofficial].
*   **Communication Overhead**: By discouraging data sharing, the method can lead to higher communication costs in the worst case (no node sharing) compared to spatially coherent distributions [S2, unofficial][S3].
*   **Approximate Balance**: Load balance is statistical and approximate. It relies on having a sufficiently large number of cells for the law of large numbers to smooth out local variations in cell size and opacity [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def round_robin_distribution(cells, num_processors):
    """
    Distributes cells to processors using round-robin assignment.
    
    Args:
        cells: List of cell objects, each having a unique index i and 
               a method get_node_indices() returning 4 node IDs.
        num_processors: Total number of processors P.
        
    Returns:
        processor_assignments: Dictionary mapping processor_id to list of cells.
        node_data: Dictionary mapping node_id to node data (coordinates, scalar).
    """
    processor_assignments = {p: [] for p in range(num_processors)}
    required_nodes = set()
    
    # Step 1 & 2: Assign cells to processors
    for i, cell in enumerate(cells):
        processor_id = i % num_processors
        processor_assignments[processor_id].append(cell)
        
        # Step 3: Identify required nodes for worst-case analysis
        # In a real implementation, nodes would be fetched and sent.
        # Here we just track which nodes are needed for the specific cells
        # assigned to each processor to calculate transfer volume.
        for node_id in cell.get_node_indices():
            required_nodes.add(node_id)
            
    return processor_assignments, required_nodes

def calculate_worst_case_transfer_volume(num_cells_per_processor):
    """
    Calculates the worst-case data transfer volume per processor.
    
    Args:
        num_cells_per_processor: Number of cells assigned to a single processor (n).
        
    Returns:
        Volume in bytes.
    """
    cell_size = 16  # bytes
    node_size = 16  # bytes
    nodes_per_cell = 4
    
    # V_max = n * (cell_size + nodes_per_cell * node_size)
    volume = num_cells_per_processor * (cell_size + nodes_per_cell * node_size)
    return volume
```

## Subtopics

### [Static Round-Robin Partitioning](2728-static-round-robin-partitioning.md)
A static load balancing strategy that partitions a data volume into many small subvolumes and distributes them round-robin across processors to average out projected area and element count differences.

### [Scattered Assignment](4204-scattered-assignment.md)
Scattered Assignment (SCA) is a data distribution scheme for parallel direct volume rendering on unstructured grids that assigns mesh-cells to processors in a scattered fashion to achieve load balancing during the Light Path Mapping (LPM) step. It sets itself apart from its siblings, Minimum-Communication Assignment (MCA) and Balanced-Load Minimized-Communication Assignment (BLMCA), by prioritizing computational load balance over communication minimization, relying on the assumption that neighboring cells have similar workloads due to similar views of the volume.

### [Randomized Data Decomposition](11592-randomized-data-decomposition.md)
A data distribution method where blocks are randomly assigned to processors with equal probability, relying on statistical theorems to guarantee load balance with high probability.

## Operating characteristics
*   **Time Complexity**: The assignment step is $O(N)$, where $N$ is the number of cells, as it involves a simple modulo operation per cell. No complex graph partitioning or iterative refinement is required, making the setup time negligible compared to dynamic methods [S2, unofficial].
*   **Memory Complexity**: $O(N)$ for storing the cell and node data. The distribution itself requires minimal additional memory for the assignment map.
*   **Measured Performance**: On an IBM SP2 supercomputer, a dataset of 500,000 tetrahedral cells achieved two frames per second for a 400x400-pixel image using 128 processors [S2, unofficial].
*   **Strengths**:
    *   **Simplicity**: Trivial to compute for any number of processors without expensive preprocessing [S2, unofficial].
    *   **Load Balance**: Effective at averaging out computational requirements when the number of cells is large, avoiding the severe load imbalances caused by grouping similar cells [S2, unofficial].
    *   **Flexibility**: Does not require connectivity data, only cell and node information [S2, unofficial][S3].
*   **Limitations**:
    *   **Communication**: Worst-case data transfer can be high if node sharing is minimal [S2, unofficial][S3].
    *   **Spatial Coherence**: Destroys spatial relationships, which can hinder optimizations like early ray termination or efficient compositing unless supplemented by a spatial partitioning tree [S2, unofficial][S3].
    *   **Approximate Balance**: Not guaranteed to be perfectly balanced for small datasets or highly heterogeneous cell properties [S2, unofficial].
*   **When to Use**: Suitable for large-scale unstructured datasets on massively parallel distributed-memory systems where computational balance is critical and spatial locality can be managed via secondary structures (e.g., a spatial tree) [S2, unofficial].
*   **When Not to Use**: Unsuitable for small datasets where statistical averaging fails, or when communication bandwidth is severely constrained and node sharing is high in the original mesh [S2, unofficial][S3].

## Background
Round-Robin Data Distribution was proposed as part of a scalable parallel cell-projection volume rendering algorithm for three-dimensional unstructured data [S2, unofficial]. It was implemented and tested on the IBM SP2 supercomputer [S2, unofficial]. The method serves as a baseline for static load balancing, often compared against dynamic approaches like Load-Based Cell (LBC) and Load-Based Adaptive Cell (LBAC) distribution to evaluate the trade-off between static assignment overhead and dynamic rebalancing costs [S1]. The approach was developed to address the limitations of ray-casting methods, which benefit from spatial coherence, by adopting a cell-projection method that allows for greater flexibility in data distribution [S2, unofficial].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
[S2] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S3] PDF A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
