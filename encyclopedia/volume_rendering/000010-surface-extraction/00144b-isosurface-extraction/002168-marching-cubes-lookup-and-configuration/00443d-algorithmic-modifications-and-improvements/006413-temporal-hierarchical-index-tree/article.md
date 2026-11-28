# Temporal Hierarchical Index Tree


## Overview
The Temporal Hierarchical Index Tree (THIT) is a search index structure designed for efficient isosurface extraction in large-scale time-varying scalar fields [S1, unofficial]. It solves the problem of overwhelming storage overhead and disk I/O associated with maintaining separate search indices for every time step in a simulation [S1, unofficial]. Unlike 4D octrees that couple spatial and temporal dimensions awkwardly, THIT uses a value-partition paradigm (Span Space) to adaptively coalesce cell extreme values over time, achieving significant disk-space savings while maintaining nearly optimal extraction performance [S1, unofficial].

## How it works
**Inputs:** A time-varying scalar field defined on a steady grid (structured or unstructured) with $T$ time steps. An isovalue $V_{iso}$.
**Outputs:** A set of triangles representing the isosurface at a specific query time step $t$.

**Data Structures:**
The core data structure is the **Temporal Hierarchical Index Tree**.
*   **Tree Structure:** A hierarchical tree where each node represents a specific time interval $[i, j]$ and a subset of cells.
*   **Node Content:** Each node stores the *temporal extreme values* for the cells it covers. For a cell $c$ in a node covering time interval $[i, j]$, the node stores:
    *   $min_{ij}(c) = \min_{t \in [i, j]} (min_t(c))$
    *   $max_{ij}(c) = \max_{t \in [i, j]} (max_t(c))$
    *   Where $min_t(c)$ and $max_t(c)$ are the minimum and maximum scalar values of cell $c$ at time step $t$.
*   **Leaf/Intermediate Organization:** Cells within a node are organized using a data structure suitable for steady-state isosurface extraction (e.g., a spatial index or simple list, depending on the underlying steady-state algorithm used [S1, unofficial]).

**Algorithm Steps:**

1.  **Characterize Cells:** For each cell in the field, determine its extreme values ($min_t, max_t$) at every time step $t$.
2.  **Classify Variation:** Analyze the variation of these extreme values over time.
    *   Cells with **small variation** (high temporal coherence) are candidates for long time intervals.
    *   Cells with **large variation** (low temporal coherence) require shorter time intervals to maintain accuracy.
3.  **Build Tree:** Construct the THIT by recursively partitioning cells based on their temporal variation.
    *   If a cell's variation over the current time interval $[i, j]$ is small, it is placed in a single node covering $[i, j]$.
    *   If the variation is large, the interval is split (e.g., $[i, k]$ and $[k+1, j]$), and the cell is placed in nodes covering the shorter sub-intervals.
    *   This process continues until a base level of variation or time resolution is reached.
4.  **Query (Extraction at time $t$):**
    *   Traverse the THIT to find all nodes that cover the specific time step $t$.
    *   Collect all cells stored in these nodes.
    *   For each collected cell, retrieve the *actual* scalar data at time $t$.
    *   Check if the cell is an isosurface cell: $min_t(c) < V_{iso} < max_t(c)$.
    *   For qualifying cells, compute the isosurface geometry (e.g., using Marching Cubes) using the actual scalar values at time $t$.

**Formulas:**
The temporal extreme values for a cell over a time interval $[i, j]$ are defined as [S1, unofficial]:
$$ min_{ij} = \min_{t=i..j} (min_t) $$
$$ max_{ij} = \max_{t=i..j} (max_t) $$

The approximation guarantees finding all isosurface cells because [S1, unofficial]:
If $t \in [i, j]$ and $min_t < V_{iso}$ and $max_t > V_{iso}$, then:
$$ min_{ij} < V_{iso} \quad \text{and} \quad max_{ij} > V_{iso} $$

**Parameters:**
*   $V_{iso}$: The isovalue defining the surface.
*   Time interval split threshold: Not explicitly defined in the sources, but implied as a parameter controlling the trade-off between storage and false positives [S1, unofficial].

**Pitfalls:**
*   **False Positives:** Using only the temporal extreme values for the entire time span as a single index is inefficient. It is a necessary but not sufficient condition, causing many non-isosurface cells to be visited [S1, unofficial]. THIT mitigates this by adaptive coalescing.
*   **Grid Assumption:** The algorithm assumes a steady grid or a grid that is transformed in a known way [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class TemporalHierarchicalIndexTree:
    def __init__(self, field_data):
        """
        field_data: A 4D array or structure containing scalar values for 
                    cells over time steps t=0..T-1.
        """
        self.root = self._build_tree(field_data, 0, len(field_data.time_steps) - 1)

    def _build_tree(self, field_data, start_t, end_t):
        """
        Recursively builds the THIT.
        """
        # Identify cells that vary significantly in this time range
        varying_cells = []
        stable_cells = []
        
        for cell in field_data.all_cells:
            # Calculate temporal extremes for this cell in [start_t, end_t]
            min_val = min(field_data[cell, t] for t in range(start_t, end_t + 1))
            max_val = max(field_data[cell, t] for t in range(start_t, end_t + 1))
            
            # Calculate variation metric (e.g., range or variance)
            variation = max_val - min_val
            
            if variation < THRESHOLD: # Threshold determines stability
                stable_cells.append((cell, min_val, max_val))
            else:
                varying_cells.append(cell)
        
        # Create node for stable cells
        node = TreeNode(time_range=(start_t, end_t), stable_cells=stable_cells)
        
        # Recurse for varying cells
        if varying_cells and start_t < end_t:
            mid_t = (start_t + end_t) // 2
            node.left = self._build_tree(field_data, start_t, mid_t)
            node.right = self._build_tree(field_data, mid_t + 1, end_t)
        else:
            # Base case: store varying cells individually or with minimal grouping
            # This ensures all cells are covered
            for cell in varying_cells:
                min_val = min(field_data[cell, t] for t in range(start_t, end_t + 1))
                max_val = max(field_data[cell, t] for t in range(start_t, end_t + 1))
                node.stable_cells.append((cell, min_val, max_val))
                
        return node

    def extract_isosurface(self, time_step, isovalue):
        """
        Extracts isosurface for a specific time step.
        """
        # Traverse tree to find relevant nodes
        relevant_cells = self._traverse_tree(self.root, time_step)
        
        triangles = []
        for cell in relevant_cells:
            # Get actual values at the specific time step
            min_t = min(field_data[cell, time_step]) # Assuming cell has multiple points
            max_t = max(field_data[cell, time_step])
            
            # Check if cell intersects isosurface
            if min_t < isovalue < max_t:
                # Compute geometry using actual data (e.g., Marching Cubes)
                cell_triangles = marching_cubes(field_data[cell, time_step], isovalue)
                triangles.extend(cell_triangles)
                
        return triangles

    def _traverse_tree(self, node, time_step):
        """
        Returns list of cells from nodes covering time_step.
        """
        cells = []
        start_t, end_t = node.time_range
        
        if start_t <= time_step <= end_t:
            # Add cells stored in this node
            cells.extend([c for c, _, _ in node.stable_cells])
            
            # Recurse if children exist
            if node.left:
                cells.extend(self._traverse_tree(node.left, time_step))
            if node.right:
                cells.extend(self._traverse_tree(node.right, time_step))
                
        return cells

class TreeNode:
    def __init__(self, time_range, stable_cells=None):
        self.time_range = time_range
        self.stable_cells = stable_cells if stable_cells else []
        self.left = None
        self.right = None
```

## Subtopics

### Span Space Partitioning
A method for analyzing temporal variation by mapping cell extreme values to a 2D space (min vs max), allowing for efficient classification of cells into stable or varying categories [S1, unofficial].

### 4D Octrees
An alternative approach that couples spatial and temporal dimensions. THIT is preferred for its ability to handle unstructured grids and avoid the awkward coupling of dimensions inherent in 4D octrees [S1, unofficial].

## Operating characteristics
*   **Storage:** Achieves more than 80% disk-space savings compared to existing techniques that store indices for every time step [S1, unofficial].
*   **Performance:** Isosurface extraction time is nearly optimal, as it minimizes the number of cells visited by leveraging temporal coherence [S1, unofficial].
*   **I/O:** Substantially reduces disk I/O required to access indices at different time steps [S1, unofficial].
*   **Scalability:** Effective for large-scale time-varying CFD simulations [S1, unofficial].
*   **Limitations:** The algorithm assumes a steady grid or a grid with known transformations [S1, unofficial].

## Background
The Temporal Hierarchical Index Tree was proposed by Han-Wei Shen at MRJ Technology Solutions / NASA Ames Research Center [S1, unofficial]. It addresses the limitations of existing isosurface extraction algorithms when applied to large-scale time-varying fields, where storage overhead for search indices becomes overwhelming [S1, unofficial]. The work builds on the value-partition paradigm and Span Space concepts [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 20020073378: Isosurface Extraction in Time-Varying Fields Using a Temporal Hierarchical Index Tree, https://archive.org/details/NASA_NTRS_Archive_20020073378, fetched 2026-10-26 (unofficial)
