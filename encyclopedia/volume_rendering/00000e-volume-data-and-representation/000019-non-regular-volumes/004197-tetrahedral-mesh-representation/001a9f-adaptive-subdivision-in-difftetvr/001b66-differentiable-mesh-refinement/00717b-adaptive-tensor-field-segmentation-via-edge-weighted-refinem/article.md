# Adaptive Tensor Field Segmentation via Edge-Weighted Refinement

Also called: Tensor Field Segmentation via Edge-Weighted Coarsening and Subdivision

## Overview
A topology-based segmentation method for visualizing 2D symmetric tensor fields that delineates cells of similar or dissimilar behavior based on directional and eigenvalue characteristics [S1, book]. It refines an initial topological pre-segmentation by merging (coarsening) or splitting (subdividing) cells using edge-weighted priority queues to maintain mesh validity and avoid hanging nodes [S1, book]. Unlike static segmentation, it offers focus+context visualization by adaptively adjusting resolution and accuracy thresholds [S1, book].

## How it works
**Inputs:** A 2D symmetric tensor field defined on a domain; an initial topological pre-segmentation consisting of curvilinear cells bounded by tensorlines [S1, book].
**Outputs:** A refined mesh of cells where each cell exhibits similar eigenvalue and eigenvector behavior [S1, book].

### Data Structures
*   **Mesh:** A planar subdivision where cells are bounded by edges (polylines). Edges may be shared by two cells or be boundary edges.
*   **Priority Queue (Coarsening):** Stores pairs of adjacent cells `(C1, C2)` eligible for merging. Sorted by geometric and scalar edge weights.
*   **Priority Queue (Subdivision):** Stores edges `e` eligible for subdivision. Implemented as a FIFO queue sorted by edge weights.
*   **Edge Weights:** Scalar values computed for each edge based on geometric properties and scalar field invariants derived from the tensor field.

### Scalar Field Derivation
Before refinement, scalar fields rendering eigenvalue behavior are derived from the tensor field [S1, book]. Common choices include:
1.  **Maximum Shear Stress:** $S = |\lambda - \mu|$, where $\lambda, \mu$ are eigenvalues [S1, book].
2.  **Generalized Fractional Anisotropy:** $FA^* = \sqrt{\frac{(\lambda - \mu)^2}{\lambda^2 + \mu^2 + A^2}}$, where $A$ is a positive constant to eliminate discontinuity near zero [S1, book]. Values range from 0 to 1.

### Edge Weight Definitions
Let $e$ be an edge consisting of $k$ segments with points $x_i$ ($i=1 \dots k+1$). Let $s$ be a scalar function along the edge, with $s_i = s(x_i)$ and $\bar{s}$ being the mean of $s$ along $e$. Let $v_i$ be the major eigenvector at $x_i$.

*   **Variance of scalar values:** $w_v(e) = \frac{\sum_{i=1}^{k} (s_i - \bar{s})^2}{\sum_{i=1}^{k} |x_{i+1} - x_i|}$ [S1, book].
*   **Absolute difference:** $w_d(e) = \text{abs}(s_{\min} - s_{\max})$, where $s_{\min}, s_{\max}$ are min/max scalar values on $e$ [S1, book].
*   **Edge length:** $w_l(e) = \sum_{i=1}^{k} |x_{i+1} - x_i|$ [S1, book].
*   **Change of eigenvector direction:** $w_c(e) = \sum_{i=1}^{k} |\angle(v_i, v_{i+1})|$ [S1, book].

### Thresholds
*   $\phi_l$: Minimum edge length threshold (e.g., 1% of displayed domain range) [S1, book].
*   $\phi_d$: Scalar difference threshold (e.g., 0.1 for $FA^*$) [S1, book].
*   $\phi_v$: Variance threshold (e.g., 0.001) [S1, book].
*   $\phi$: Anisotropy threshold for degenerate regions (0.05) [S1, book].

### Algorithm Steps

#### 1. Pre-segmentation
Extract the integral topological skeleton using major and minor eigenvector fields. Delineate regions of equivalent eigenvector behavior bounded by tensorlines to form curvilinear cells [S1, book].

#### 2. Coarsening (Merge)
Merges adjacent cells lacking structural information.
*   **Prerequisite:** Two cells share a common edge that can be deleted. Edges containing hanging nodes cannot be common edges [S1, book].
*   **Priority:** Pairs are sorted by minimum edge length of involved edges, then by maximum edge length of edges to be merged (for basic workflow) [S1, book].
*   **Process:**
    1.  Identify all adjacent cell pairs that can be merged.
    2.  Sort pairs into the coarsening priority queue based on chosen edge weights.
    3.  While queue is not empty:
        a. Pop the highest priority pair $(C_1, C_2)$.
        b. Merge $C_1$ and $C_2$ by deleting the common edge and creating a new cell from the union of edges.
        c. Update the mesh data structure.
        d. Update the priority queue: Add new potential merge pairs involving the new cell. If contiguous pairs in the direction of the deleted edge are candidates, move them to the top of the queue to ensure recursive merging [S1, book].

#### 3. Subdivision (Split)
Refines cells exceeding similarity criteria.
*   **Priority:** Edges are sorted by maximum edge length, then by maximum scalar edge weight (for basic workflow) [S1, book].
*   **Start Point:**
    *   Option A: Midpoint of the edge (favors uniform cells) [S1, book].
    *   Option B: Point between extrema of scalar values on the edge (guarantees decrease in edge weight, more data-adapted) [S1, book].
*   **Process:**
    1.  Sort edges into the subdivision priority queue.
    2.  While queue is not empty:
        a. Pop the top edge $e$.
        b. Start a subdividing tensor line from the chosen start point on $e$, with opposite color (direction) to the boundary tensorlines.
        c. Integrate the tensor line until termination:
            i. It intersects an edge not in the priority queue.
            ii. It intersects an edge where subdivision would violate fixed thresholds (e.g., minimum edge length).
            iii. It meets other specific termination conditions (e.g., reaching a degenerate entity or boundary) [S1, book].
        d. Subdivide all cells intersected by the new tensor line. Update the mesh by adding new edges along the tensor line and generating new cells.
        e. Update the priority queue: Remove the original edge $e$. Add newly generated edges if they are candidates for further subdivision, sorted by weights [S1, book].

#### 4. Basic Segmentation Workflow
1.  **First Coarsening:** Merge similar cells to clean up pre-segmentation. Priority: min edge length, then max edge length [S1, book].
2.  **Subdivision:** Refine cells to pre-defined accuracy unless violating resolution. Start point: between extrema. Priority: max edge length, then max scalar weight [S1, book].
3.  **Final Coarsening:** Merge cells with highest similarity. Priority: min edge length, then ascending scalar edge weight $w_d$ [S1, book].

#### 5. Focus + Context
For a chosen focus region, repeat steps 2 and 3 with an adjusted geometric edge weight (minimum edge length), while keeping the accuracy edge weight unchanged. Cells exceeding the accuracy threshold can be highlighted [S1, book].

#### 6. Degenerate Region Handling
Degenerate entities (where tensors are isotropic) are handled via two subdivision passes using $FA^*$ as the scalar field [S1, book].
1.  **First Subdivision:** Consider only edges emerging from a degenerate entity (degenerate edges). Edge weight is the maximum anisotropy on the edge. Start point: point on the edge with anisotropy $< \phi$ (0.05) and maximum distance from the degenerate entity. Priority: least anisotropy value of start points [S1, book].
2.  **Second Subdivision:** Consider all edges participating in intermediate degenerate regions from the first pass, as the first pass does not guarantee intersection at low anisotropy points [S1, book].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_tensor_segmentation(tensor_field, initial_cells):
    """
    Main workflow for adaptive tensor field segmentation.
    """
    # 1. Pre-segmentation (Topology-based)
    cells = extract_topological_skeleton(tensor_field)
    
    # 2. Derive Scalar Field (e.g., FA*)
    scalar_field = derive_scalar_field(tensor_field, method='FA', constant_A=1.0)
    
    # 3. First Coarsening
    coarsen_cells(cells, scalar_field, 
                  weight_type='geometric', 
                  sort_keys=['min_edge_length', 'max_edge_length'])
    
    # 4. Subdivision
    subdivide_cells(cells, scalar_field, 
                    weight_type='scalar', 
                    start_point='extrema', 
                    sort_keys=['max_edge_length', 'max_scalar_weight'])
    
    # 5. Final Coarsening
    coarsen_cells(cells, scalar_field, 
                  weight_type='scalar', 
                  sort_keys=['min_edge_length', 'asc_scalar_weight'])
    
    return cells

def coarsen_cells(cells, scalar_field, weight_type, sort_keys):
    """
    Merges adjacent cells based on edge weights.
    """
    queue = PriorityQueue()
    
    # Initialize queue with all mergeable pairs
    for c1, c2 in get_adjacent_pairs(cells):
        if can_merge(c1, c2):
            weight = compute_merge_priority(c1, c2, scalar_field, weight_type)
            queue.push((weight, c1, c2))
            
    while not queue.is_empty():
        weight, c1, c2 = queue.pop()
        
        # Check if still valid (might have been merged/modified)
        if not are_adjacent(c1, c2) or not can_merge(c1, c2):
            continue
            
        # Merge
        new_cell = merge(c1, c2)
        update_mesh(cells, new_cell, c1, c2)
        
        # Update Queue: Add new pairs involving new_cell
        for neighbor in get_neighbors(new_cell):
            if can_merge(new_cell, neighbor):
                new_weight = compute_merge_priority(new_cell, neighbor, scalar_field, weight_type)
                # Recursive strategy: push to top if it continues a chain
                queue.push((new_weight, new_cell, neighbor), priority_boost=True)

def subdivide_cells(cells, scalar_field, weight_type, start_point, sort_keys):
    """
    Subdivides cells by integrating tensor lines.
    """
    queue = PriorityQueue()
    
    # Initialize queue with all edges to be subdivided
    for cell in cells:
        for edge in cell.edges:
            if needs_subdivision(edge, scalar_field, weight_type):
                queue.push(edge, sort_keys=sort_keys)
                
    while not queue.is_empty():
        edge = queue.pop()
        
        # Determine start point
        if start_point == 'midpoint':
            start_pos = edge.midpoint()
        elif start_point == 'extrema':
            start_pos = edge.extrema_point(scalar_field)
            
        # Integrate tensor line
        tensor_line = integrate_tensor_line(start_pos, tensor_field, opposite_color=True)
        
        # Find intersected edges
        intersected_edges = []
        for e in cells.edges:
            if tensor_line.intersects(e) and e in queue:
                intersected_edges.append(e)
                
        # Subdivide intersected cells
        new_edges = []
        for e in intersected_edges:
            # Split cell containing e along tensor_line
            new_cells = split_cell(e, tensor_line)
            update_mesh(cells, new_cells, e)
            new_edges.extend(new_cells.edges)
            
        # Update Queue
        for e in intersected_edges:
            queue.remove(e)
        for e in new_edges:
            if needs_subdivision(e, scalar_field, weight_type):
                queue.push(e, sort_keys=sort_keys)

def compute_merge_priority(c1, c2, scalar_field, weight_type):
    """
    Computes the priority weight for merging two cells.
    """
    common_edge = get_common_edge(c1, c2)
    if weight_type == 'geometric':
        return (min_edge_length(c1.edges + c2.edges), max_edge_length(common_edge))
    elif weight_type == 'scalar':
        return (min_edge_length(c1.edges + c2.edges), scalar_weight(common_edge, scalar_field))
    return 0

def scalar_weight(edge, scalar_field):
    """
    Computes scalar edge weight (e.g., absolute difference).
    """
    s_vals = [scalar_field.sample(p) for p in edge.points]
    return abs(max(s_vals) - min(s_vals))
```

## Operating characteristics
*   **Time Complexity:** Not explicitly given, but depends on the number of cells, edges, and integration steps for tensor lines. Priority queue operations add logarithmic factors.
*   **Memory Complexity:** Linear with respect to the number of cells and edges in the final mesh.
*   **Performance:** On the 2PL dataset, variance weight ($w_v$) resulted in 2497 cells with error $5.039 \times 10^{-2}$, while basic workflow ($w_d$) resulted in 2830 cells with error $4.67 \times 10^{-2}$ [S1, book]. On the 1PL dataset, $w_v$ resulted in 1129 cells (error $3.64 \times 10^{-2}$) vs 1424 cells (error $3.38 \times 10^{-2}$) for basic [S1, book]. On the NB dataset, $w_v$ resulted in 285 cells (error $4.93 \times 10^{-2}$) vs 420 cells (error $4.02 \times 10^{-2}$) for basic [S1, book].
*   **Strengths:** Focus+context visualization; intuitive customization via edge weights; handles degenerate regions; maintains mesh validity.
*   **Limitations:** Edge weights only reflect scalar field behavior on edges, not necessarily the interior error of a cell; degenerate region handling may produce small, poorly shaped cells that are hard to merge later [S1, book].
*   **When to use:** When visualizing 2D symmetric tensor fields (e.g., stress simulations) requires adaptive resolution based on eigenvalue similarity and directional structures.
*   **When not to use:** When high accuracy in the interior of cells is required without edge-based approximation; when 3D tensor fields are involved (method is for 2D).

## Background
Originated in a 2011 paper by J. Nair, M. Zobel, and T. Hotz, presented in the *Digital Foundations* volume on Scientific Visualization [S1, book]. It addresses the insufficiency of scalar field analysis for understanding tensor fields by providing a topological segmentation that captures both directional (eigenvector) and scalar (eigenvalue) characteristics. The method is applied to tensor fields from numerical stress simulations in structural engineering, such as one-point load (1PL), two-point load (2PL), and notched block (NB) datasets [S1, book].

## Sources
[S1] DFU, Volume 2, Scientific Visualization: Interactions, Features, Metaphors, Complete Volume, https://doi.org/10.4230/dfu.vol2.sciviz.2011, fetched 2026-10-26 (book)
