# Graph Partitioning for Cell Clusterization
Also called: Graph Partitioning, METIS Graph Partitioning

## Overview
Graph Partitioning for Cell Clusterization assigns tetrahedral volume cells to processor-specific clusters to balance rendering load and minimize inter-cluster communication [S1, unofficial]. It sets itself apart from heuristic clustering methods by using the METIS tool to solve a constrained optimization problem, ensuring geometric stability and balanced computational costs across different visualizations [S1, unofficial].

## How it works
**Inputs:**
*   A tetrahedral mesh defined by cells (vertices in the graph) and shared faces (edges in the graph).
*   The number of partitions $P$ (typically equal to the number of processors $N$).
*   Vertex weights $W_v$ representing rendering cost.
*   Edge weights $W_e$ representing interaction cost.

**Outputs:**
*   A partition $C = \{C_1, C_2, \dots, C_P\}$ of the vertex set $V$, where each $C_i$ is a disjoint, non-empty subset of cells assigned to processor $i$ [S1, unofficial].

**Data Structures:**
*   **Graph $G = (V, E, W_v, W_e)$**: An undirected graph without loops or multiple edges [S1, unofficial].
    *   $V$: Set of tetrahedral cells.
    *   $E$: Set of faces connecting cells. Cell $V_i$ is connected to $V_j$ if they share a common face [S1, unofficial].
    *   $W_v$: Vertex weights corresponding to cell rendering costs.
    *   $W_e$: Edge weights corresponding to the amount of interaction between cells [S1, unofficial].

**Steps:**
1.  **Construct Graph**: Build $G$ where each node represents a tetrahedral cell and each edge represents a shared face between two cells [S1, unofficial].
2.  **Assign Vertex Weights**: Calculate $W_v$ for each cell. The rendering cost of a cluster $C_i$ is approximated by the sum of the areas of the triangular faces within that cluster [S1, unofficial]:
    $$ \text{Load}(C_i) = \sum_{f \in C_i} \text{Area}(f) $$
    where $\text{Area}(f)$ is the area of triangular face $f$ [S1, unofficial].
3.  **Assign Edge Weights**: Define $W_e$ based on the interaction cost between adjacent cells. The specific weighting scheme (e.g., geometric stability vs. cutnet metric) determines these values [S1, unofficial].
4.  **Partition**: Use the METIS graph partitioning tool to divide $V$ into $P$ subsets [S1, unofficial].
5.  **Enforce Constraints**: The partitioning must satisfy:
    *   **Balance**: The sum of vertex weights in each cluster $C_i$ should be approximately equal, ensuring nearly equal rendering costs [S1, unofficial].
    *   **Minimization**: The sum of edge weights $W_e$ connecting nodes in different partitions $C_i$ and $C_j$ should be minimized, reducing total interaction between clusters [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Graph Partitioning for Cell Clusterization
# Note: METIS is the specific tool cited; standard graph partitioning logic is implied.

def graph_partitioning_cell_clusterization(cells, faces, num_partitions):
    """
    Assigns tetrahedral cells to partitions using graph partitioning.
    
    Args:
        cells: List of tetrahedral cell objects.
        faces: List of face objects, each containing references to two adjacent cells.
        num_partitions: Number of clusters P (typically number of processors).
        
    Returns:
        partitions: List of P lists, where each list contains cell IDs.
    """
    
    # Step 1: Construct the graph G = (V, E, Wv, We)
    # V is the set of cells. E is the set of shared faces.
    graph = Graph()
    
    # Add vertices (cells)
    for cell in cells:
        graph.add_vertex(cell.id)
        
    # Add edges (shared faces) and calculate weights
    for face in faces:
        cell_u = face.cell_u
        cell_v = face.cell_v
        
        # Edge exists if cells share a face
        if cell_u != cell_v:
            # Calculate vertex weight Wv: Sum of areas of faces in the cluster
            # This is an approximation of rendering cost
            area_u = face.area # Assuming face area contributes to load
            # Note: In practice, Wv is pre-calculated per cell based on all its faces
            # Wv[cell_u] += area_u
            # Wv[cell_v] += area_v
            
            # Calculate edge weight We: Interaction cost
            # This depends on the specific weighting scheme (e.g., Cutnet Metric)
            weight_e = calculate_edge_weight(face)
            
            graph.add_edge(cell_u.id, cell_v.id, weight=weight_e)
            
    # Step 2: Set vertex weights
    # The constraint is that sum of Wv in each cluster should be approximately equal
    for cell in cells:
        # Wv is the rendering cost, approximated by sum of face areas
        # This must be calculated before calling METIS
        graph.set_vertex_weight(cell.id, cell.rendering_cost)
        
    # Step 3: Partition the graph using METIS
    # METIS minimizes edge cuts while balancing vertex weights
    partitions = metis_partition(graph, num_partitions)
    
    return partitions

def calculate_edge_weight(face):
    """
    Placeholder for edge weight calculation.
    The specific metric (e.g., Cutnet Metric) determines this value.
    [S1, unofficial]
    """
    # Implementation depends on the chosen weighting scheme
    return 1.0 # Default placeholder
```

**Edge Cases and Pitfalls:**
*   **Precision**: The load approximation using face area is an estimation; actual rendering cost may vary based on projection and screen pixel distribution [S1, unofficial].
*   **Boundaries**: The number of clusters $P$ can be chosen between the number of processors $N$ and the number of cells, but must be balanced against communication overhead [S1, unofficial].
*   **Degenerate Input**: The graph must not have loops or multiple edges [S1, unofficial].

## Subtopics

### [Weighting Scheme for Clusterization Graph](23487-weighting-scheme-for-clusterization-graph.md)
Defines vertex and edge weights to balance load and minimize communication, prioritizing geometric stability over cell volume or area metrics.

### [Cutnet Metric](23492-cutnet-metric.md)
A partitioning objective function where the cost is the sum of the weights of external nets.

## Operating characteristics
*   **Time Complexity**: Depends on the METIS algorithm used, typically sub-linear or linear with respect to the number of cells for large graphs.
*   **Memory Complexity**: Requires storing the graph structure (cells, faces, weights).
*   **Performance**: Eliminates the cost of global clustering by allowing processors to work on initially assigned data in parallel [S1, unofficial].
*   **Strengths**: Balances rendering costs and minimizes interaction; produces spherical clusters with stable rendering loads [S1, unofficial].
*   **Limitations**: Requires pre-computation of graph weights; may not adapt dynamically to view-dependent changes without remapping [S1, unofficial].
*   **When to Use**: When static partitioning is sufficient and load balancing is critical for parallel volume rendering [S1, unofficial].
*   **When Not to Use**: When dynamic, view-dependent clustering is required without significant remapping overhead [S1, unofficial].

## Background
Graph partitioning is a technique for assigning tasks to partitions to balance load and minimize interaction, arising in VLSI design, telephone network design, and sparse Gaussian elimination [S1, unofficial]. In volume rendering, it is used to form cell clusters for parallel processing [S1, unofficial]. The method uses the METIS graph partitioning tool [S1, unofficial].

## Sources
[S1] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
