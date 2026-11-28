# Weighting Scheme for Clusterization Graph

Also called: Vertex and Edge Weighting for Graph Partitioning

## Overview
This method defines the vertex and edge weights for the clusterization graph used in graph-partitioning-based volume rendering [S1, unofficial]. It solves the problem of balancing computational load and minimizing inter-cluster communication by assigning specific metrics to graph nodes and edges [S1, unofficial]. It sets apart from alternatives like the Cutnet Metric by prioritizing geometric stability; specifically, it produces spherical clusters with stable rendering loads across different visualizations, avoiding the huge variations in cell numbers found in Cell Volume (CV) or Cell Area (CA) schemes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** The clusterization graph $G=(V, E)$, where $V$ represents cells (or clusters) and $E$ represents adjacency between them.
*   **Output:** Weighted graph $G'$ with vertex weights $w_v$ and edge weights $w_e$.

**Data Structures**
The graph is represented as an adjacency structure. Each vertex $v \in V$ stores a scalar weight $w_v$. Each edge $e \in E$ connecting vertices $u$ and $v$ stores a scalar weight $w_e$.

**Formulas and Definitions**
Let $CV$ denote Cell Volume, $CA$ denote Cell Area, and $FA$ denote Face Area.
The weighting scheme is defined as:
$$ w_v = 1 \quad \forall v \in V $$
$$ w_e = FA \quad \forall e \in E $$

**Parameters**
*   **Vertex Weight Scheme:** Unit cost (Value: 1).
*   **Edge Weight Scheme:** Face Area (Value: Geometric area of the shared face).

**Steps**
1.  Initialize the clusterization graph $G$ with all cells as vertices and shared faces as edges.
2.  For every vertex $v$ in $V$, set the vertex weight $w_v$ to 1. This ensures equal cell size and uniform communication cost [S1, unofficial].
3.  For every edge $e$ in $E$, calculate the Face Area ($FA$) of the geometric interface between the two adjacent cells.
4.  Set the edge weight $w_e$ to the calculated $FA$. This represents the cluster's interaction with other clusters and the screen [S1, unofficial].
5.  Pass the weighted graph $G'$ to the partitioning tool (e.g., METIS) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_weighting_scheme(graph):
    """
    Assigns weights to the clusterization graph for volume rendering partitioning.
    
    Args:
        graph: A graph object containing nodes (cells) and edges (shared faces).
               Each edge must have a method or attribute to retrieve the geometric 
               face area between adjacent cells.
               
    Returns:
        graph: The same graph object with updated vertex and edge weights.
    """
    # Step 2: Set all vertex weights to 1 (Unit cost scheme)
    for vertex in graph.vertices:
        vertex.weight = 1.0
        
    # Step 3 & 4: Set edge weights to Face Area (FA)
    for edge in graph.edges:
        # Calculate the geometric area of the shared face between the two 
        # cells connected by this edge.
        face_area = calculate_face_area(edge.cell_a, edge.cell_b)
        edge.weight = face_area
        
    return graph

def calculate_face_area(cell_a, cell_b):
    """
    [G] Helper to compute the area of the shared boundary between two adjacent 
    volume cells. The exact implementation depends on the cell geometry 
    (e.g., hexahedral, tetrahedral).
    """
    # Implementation detail: Compute the intersection polygon/area of the 
    # two cells' boundaries.
    shared_boundary = compute_intersection(cell_a.geometry, cell_b.geometry)
    return compute_area(shared_boundary)
```

**Edge Cases and Pitfalls**
*   **Precision:** Face area calculations must be precise to avoid load imbalance. Degenerate faces (zero area) should be handled, though in regular grids this is rare.
*   **Boundary Conditions:** Edges on the domain boundary or cluster boundaries interact with the screen. The FA metric inherently accounts for this interaction probability [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Strengths:** Produces spherical clusters that do not change rendering loads significantly at different visualizations [S1, unofficial]. Avoids huge variations in the number of cells per cluster, which occurs in CV and CA schemes [S1, unofficial].
*   **Limitations:** Relies on geometric face area, which may not directly correlate with computational cost if cell sizes vary significantly (though unit vertex weights assume uniform cell cost).
*   **When to use:** When stable rendering performance across varying camera angles and visualizations is required, and when minimizing communication cost variance is prioritized over strict computational load balancing based on cell volume.

## Background
This weighting scheme originates from research on hypergraph-partitioning based remapping models for image-space parallel volume rendering [S1, unofficial]. It was selected in experimental studies (Chapter 7 of the source) to outperform other schemes like CV (Cell Volume) and CA (Cell Area) in terms of load stability and cluster shape [S1, unofficial].

## Sources
[S1] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
