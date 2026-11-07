# Joint Contour Net

Also called: Discrete Reeb space

## Overview
The Joint Contour Net (JCN) is a discrete graph-based representation of the Reeb space for multivariate scalar data, approximating the topology of $n$ variables defined over an $m$-dimensional domain [S1, unofficial]. It solves the problem of visualizing and analyzing the topological relationships between multiple scalar fields by partitioning the domain into connected regions called joint contour slabs [S1, unofficial]. Unlike the [Reeb Graph](9315-reeb-graph.md), which handles single-variable topology, the JCN extends this to multivariate inputs; it differs from the [Multi-Dimensional Reeb Graph](9382-multi-dimensional-reeb-graph.md) by using a quantization-based slab construction rather than critical node extraction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** $n$ scalar functions $f_1, \dots, f_n$ defined on an $m$-dimensional domain $\mathbb{R}^m$ (typically a grid or mesh).
*   **Output:** An undirected graph $G = (V, E)$.
    *   If $n < m$, the output is an approximation of the Reeb space.
    *   If $n \ge m$, the output is a subdivision of the input geometry over the $n$ variables [S1, unofficial].

**Data Structures:**
*   **Graph:** Undirected graph.
*   **Vertices ($V$):** Represent joint contour slabs. Each vertex stores the barycentric spatial coordinates of the slab.
*   **Edges ($E$):** Represent adjacency between joint contour slabs.

**Algorithm Steps:**
1.  **Quantization:** Define quantization intervals for each of the $n$ input scalar functions.
2.  **Overlay:** Overlay the quantization intervals to construct the bivariate/multivariate field. This partitions the domain into regions where the isovalue tuples of all $n$ functions remain constant within their respective intervals.
3.  **Slab Identification:** Identify connected regions formed by the overlay. These regions are the joint contour slabs. Each slab is a connected component of the domain where the quantized values of all input functions are identical [S1, unofficial].
4.  **Vertex Placement:** For each identified joint contour slab, compute its barycenter (centroid) in the spatial domain. Place a vertex in the graph at this coordinate.
5.  **Edge Construction:** Mark adjacency between regions. If two joint contour slabs share a boundary in the domain, add an undirected edge between their corresponding vertices in the graph [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_joint_contour_net(scalar_fields, quantization_intervals):
    """
    scalar_fields: List of n scalar functions (arrays or grids).
    quantization_intervals: List of n tuples (min, max, step) for each field.
    Returns: Graph with vertices at slab barycenters and edges for adjacency.
    """
    # Step 1 & 2: Quantize and Overlay
    # Create a grid of quantized values for each field
    quantized_fields = []
    for i, field in enumerate(scalar_fields):
        min_val, max_val, step = quantization_intervals[i]
        # Quantize field values to discrete bins
        bins = np.arange(min_val, max_val + step, step)
        quantized = np.digitize(field, bins) - 1
        quantized_fields.append(quantized)

    # Create a combined key for each voxel/cell
    # The key is the tuple of quantized values for all n fields
    num_voxels = scalar_fields[0].size
    combined_keys = []
    for idx in range(num_voxels):
        key = tuple(q[idx] for q in quantized_fields)
        combined_keys.append(key)

    # Step 3: Identify Slabs (Connected Components)
    # Group voxels by their combined key
    slab_groups = {}
    for idx, key in enumerate(combined_keys):
        if key not in slab_groups:
            slab_groups[key] = []
        slab_groups[key].append(idx)

    # For each group, determine if it is connected (spatially)
    # This requires knowing the spatial connectivity of the input domain
    # Assuming a regular grid for simplicity
    vertices = []
    edges = []
    vertex_map = {} # Maps slab_key to vertex_index

    for slab_key, voxel_indices in slab_groups.items():
        # Compute barycenter of the slab
        spatial_coords = get_spatial_coordinates(voxel_indices)
        barycenter = np.mean(spatial_coords, axis=0)
        
        vertex_index = len(vertices)
        vertices.append({'barycenter': barycenter, 'slab_key': slab_key})
        vertex_map[slab_key] = vertex_index

    # Step 5: Mark Adjacency
    # Check neighbors of each voxel in each slab
    # If a neighbor belongs to a different slab, add an edge
    for slab_key, voxel_indices in slab_groups.items():
        current_vertex_idx = vertex_map[slab_key]
        
        for voxel_idx in voxel_indices:
            neighbors = get_spatial_neighbors(voxel_idx)
            for neighbor_idx in neighbors:
                neighbor_key = combined_keys[neighbor_idx]
                if neighbor_key != slab_key:
                    neighbor_vertex_idx = vertex_map[neighbor_key]
                    # Add edge if not already present
                    if not edge_exists(current_vertex_idx, neighbor_vertex_idx):
                        edges.append((current_vertex_idx, neighbor_vertex_idx))

    return Graph(vertices, edges)
```

**Parameters:**
*   **Quantization Intervals:** The choice of intervals affects the resolution of the JCN. Finer intervals yield more slabs and a more detailed graph, but increase computational cost. Typical values depend on the range and distribution of the scalar fields [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Parallelization Merge:** In distributed memory implementations, multiple sub-JCNs are constructed in parallel. The merge step, which combines these sub-JCNs into a single output, is the limiting factor for speedup [S1, unofficial].
*   **High Dimensionality:** When $n \ge m$, the output is not a Reeb space approximation but a geometric subdivision. The algorithm still computes, but the topological interpretation changes [S1, unofficial].

## Subtopics

### Jacobi Structure
An improvement over the Jacobi set that relates the Jacobi set directly to the Reeb space by capturing the exact location of topological change [S1, unofficial]. It includes 'regular sheets' connecting critical nodes, allowing it to capture topological elements (e.g., forking in multi-field topology) that the standard Jacobi set cannot represent [S1, unofficial]. It is extracted as the set of critical nodes in the [Multi-Dimensional Reeb Graph](9382-multi-dimensional-reeb-graph.md) (MDRG) [S1, unofficial].

### Layered Reeb Graph
An alternative approach for presenting the Reeb space of multiple scalar functions [S1, unofficial]. It differs from the MDRG by working directly with Jacobi sets rather than the Jacobi structure (which includes regular sheets) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, the algorithm involves quantization, grouping, and connectivity checks, which are generally linear in the number of voxels for regular grids, but connectivity checks can be expensive.
*   **Memory Complexity:** Not given in the sources. Depends on the number of joint contour slabs and the size of the input domain.
*   **Performance:** The JCN is better suited to parallelization than contour tree algorithms because each joint contour slab is constructed from smaller independent regions called fragments [S1, unofficial].
*   **Limitations:** The merge step in parallel implementations is the limiting factor for speedup [S1, unofficial].
*   **When to Use:** Suitable for visualizing and analyzing multivariate data where topological relationships between multiple scalar fields are of interest, such as in nuclear physics (scission datasets) or meteorology (hurricane data) [S1, unofficial].
*   **When Not to Use:** When a single-variable topological summary is sufficient (use [Reeb Graph](9315-reeb-graph.md) or [Contour Tree](9315-reeb-graph.md)).

## Background
The Joint Contour Net was first presented as a discrete representation of the Reeb space by Carr et al. [53] [S1, unofficial]. It has been used in nuclear physics to visualize and analyze scission datasets, identifying the splitting of atomic nuclei into multiple parts using proton and neutron density fields [S1, unofficial]. It was also used to visually analyze data from Hurricane Isabel, mapping vertices to barycentric spatial coordinates to capture topological changes [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
