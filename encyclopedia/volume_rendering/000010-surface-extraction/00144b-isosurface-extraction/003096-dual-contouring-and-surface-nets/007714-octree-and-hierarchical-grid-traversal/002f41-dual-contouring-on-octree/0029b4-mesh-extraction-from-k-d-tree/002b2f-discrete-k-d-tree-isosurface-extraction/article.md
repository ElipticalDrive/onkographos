# Discrete k-d tree isosurface extraction

Also called: Direct Isosurface Extraction from Scattered Volume Data, KD-Tree Isosurface Extraction, KD-tree contour edge traversal

## Overview
This method extracts isosurfaces directly from scattered volume data (unstructured samples with no known connectivity) without resampling onto a structured grid or generating a global/local polyhedral mesh [S2]. It computes points on the isosurface by linearly interpolating between neighboring pairs of samples and renders the result as a point cloud [S2]. It sets itself apart from gridded methods like Marching Cubes by avoiding grid construction, and from Delaunay tetrahedrization by being approximately one order of magnitude faster [S2].

## How it works
**Inputs:** A set of scattered sample points $X = \{x_i\}$ in $\mathbb{R}^3$ with associated scalar values $f(x_i)$, and a real isovalue $v_{iso}$ [S2].
**Outputs:** A point cloud representing the isosurface $f(x) = v_{iso}$ [S2].

### Data Structures
1.  **K-D Tree:** Partitions the 3D domain of scattered sample positions into cells [S2].
2.  **Binary Index:** Each cell in the k-d tree is assigned a binary index to enable direct access without tree traversal [S2].
3.  **EdgeKd:** An array of 4 k-d tree node pointers, indexed 0..3, accessed as `nodes[i*2]` and `nodes[i*2+1]` for pairs [S3, unofficial].
4.  **AALine:** A struct containing a `PositionCode` (integer vector representing grid coordinates) and a `dir` (int) [S3, unofficial].
5.  **PositionCode:** A vector-like type with integer components accessible via `[dir]` [S3, unofficial].

### Algorithm Steps

1.  **Build K-D Tree:** Partition the 3D domain of scattered samples into cells using a k-d tree data structure [S2].
2.  **Index Cells:** Assign a binary index to each cell in the k-d tree [S2].
3.  **Neighbor Candidate Selection:** For each sample $x_i$:
    *   Determine potential neighbor candidates by performing bitwise operations on the binary indices of the cell representing $x_i$ and its neighbors [S2].
    *   Select appropriate neighbors from the small set of candidates using an angle criterion $\alpha$ [S2].
    *   *Note:* Using nearest neighbors instead of natural neighbors (estimated via k-d tree/angle criterion) fails in cases of varying density distributions [S2].
4.  **Isosurface Intersection Detection:** For each sample $x_i$ and each selected neighbor $x_j$:
    *   Examine if the neighbor is separated from the sample by the isosurface (i.e., $f(x_i) - v_{iso}$ and $f(x_j) - v_{iso}$ have opposite signs) [S2].
5.  **Isopoint Computation:** For each pair $(x_i, x_j)$ separated by the isosurface:
    *   Linearly interpolate between $x_i$ and $x_j$ to compute the exact position $p_k$ of the isopoint on the connecting edge:
        $$ p_k = x_i + \frac{v_{iso} - f(x_i)}{f(x_j) - f(x_i)} (x_j - x_i) $$
    *   Add $p_k$ to the output point cloud [S2].
6.  **Rendering:** Render the resulting set of isopoints using point-based rendering techniques, specifically splatting [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface(samples, v_iso, angle_alpha):
    """
    samples: list of (position: Vec3, value: float)
    v_iso: float, isovalue
    angle_alpha: float, angle criterion in degrees
    returns: list of Vec3 (isopoints)
    """
    # Step 1 & 2: Partition domain and assign binary indices
    kdtree = _build_kd_tree(samples)
    cell_indices = _assign_binary_indices(kdtree)

    isopoints = []

    # Step 3 & 4: Neighbor selection
    for sample in samples:
        current_cell_idx = _get_cell_index(sample.position, kdtree)
        # Bitwise neighbor lookup on binary indices [S2]
        candidate_cells = _bitwise_neighbor_lookup(current_cell_idx, cell_indices)

        selected_neighbors = []
        for candidate_cell in candidate_cells:
            for cand_sample in _get_samples_in_cell(candidate_cell, kdtree):
                if cand_sample is sample:
                    continue
                # Angle criterion check [S2]
                if _check_angle(sample.position, cand_sample.position, angle_alpha):
                    selected_neighbors.append(cand_sample)

        # Step 5 & 6: Isosurface intersection and interpolation
        for neighbor in selected_neighbors:
            if _crosses_isosurface(sample.value, neighbor.value, v_iso):
                t = (v_iso - sample.value) / (neighbor.value - sample.value)
                isopoint = sample.position + t * (neighbor.position - sample.position)
                isopoints.append(isopoint)

    return isopoints

def render_point_cloud(isopoints):
    # Render using point-based rendering (splatting) [S2]
    for point in isopoints:
        _render_splat(point)

# Helper definitions (standard operations)
def _build_kd_tree(samples):
    # Standard kd-tree construction from point cloud
    return KDTree(samples)

def _assign_binary_indices(kdtree):
    # Assign binary index to each cell for direct access without traversal [S2]
    return {node.id: _compute_binary_index(node) for node in kdtree.nodes}

def _get_cell_index(position, kdtree):
    return kdtree.find_cell(position)

def _bitwise_neighbor_lookup(cell_idx, cell_indices):
    # Perform bitwise operations on binary indices to find potential neighbors [S2]
    neighbors = []
    for mask in _get_neighbor_masks():
        neighbor_idx = cell_idx ^ mask
        if neighbor_idx in cell_indices:
            neighbors.append(neighbor_idx)
    return neighbors

def _get_samples_in_cell(cell_idx, kdtree):
    return kdtree.get_samples(cell_idx)

def _check_angle(p1, p2, alpha):
    # Standard angle criterion check [G]
    vec = p2 - p1
    return vec.magnitude() <= alpha # Simplified for pseudocode clarity

def _crosses_isosurface(v1, v2, v_iso):
    return (v1 - v_iso) * (v2 - v_iso) < 0

def _render_splat(point):
    # Spherical splat rendering [S2]
    pass
```

### Parameters
*   **$v_{iso}$**: The real isovalue defining the isosurface $f(x) = v_{iso}$, within the range of the scalar function $f$ [S2].
*   **Angle criterion ($\alpha$)**: A parameter used to select appropriate neighbors. Values tested include $15^\circ$, $35^\circ$, $55^\circ$, and $80^\circ$ [S2].
*   **Error threshold**: Defaults to $10^{-7}$ [S3, unofficial].

### Edge Cases and Pitfalls
*   **Density Variation:** Using nearest neighbors instead of natural neighbors (estimated via k-d tree/angle criterion) fails in cases of varying density distributions of samples [S2].
*   **Boundary Handling:** During traversal (in related contouring implementations), if a node pointer becomes null (e.g., boundary of tree), the function returns immediately to prevent segfaults [S3, unofficial].
*   **Degenerate Edges:** In contouring leaf checks, if `minEnd[dir] >= maxEnd[dir]`, the edge is degenerate and the quad is skipped [S3, unofficial].
*   **Rendering Limitations:** The current implementation uses spherical splats instead of elliptical ones and cannot deal with sharp edges and corners yet [S2].

## Operating characteristics
*   **Time Complexity:** Approximately one order of magnitude faster than 3D mesh generation algorithms like Delaunay tetrahedrization, which has quadratic asymptotic time complexity [S2].
*   **Memory:** K-D trees require more memory than octrees because they store explicit cell positions, whereas octrees have implicit position information [S1].
*   **Quality:** The method gets very close to what a gridded isosurfacing technique like Marching Cubes generates in terms of deviation from the analytic surface [S2].
*   **Performance Data:**
    *   Synthetic sphere (8M samples, radius 70): Deviations from analytic sphere range from 0.0182 to 0.0425 depending on angle $\alpha$. Marching Cubes on 200x200x200 grid had a deviation of 0.0018 [S2].
    *   Sphere data set (8M samples):
        *   $\alpha=15^\circ$: 437,473 isopoints, deviation 0.0425.
        *   $\alpha=35^\circ$: 227,588 isopoints, deviation 0.0355.
        *   $\alpha=55^\circ$: 77,512 isopoints, deviation 0.0320.
        *   $\alpha=80^\circ$: 64,165 isopoints, deviation 0.0182 [S2].
    *   Boston Teapot (resampled 256x256x178): 243,238 isopoints extracted [S2].
    *   Engine (resampled 16M scattered points): 405,327 isopoints generated [S2].
*   **Strengths:** Fast extraction from scattered data; robust against varying density; no need for grid reconstruction or global meshing [S2].
*   **Limitations:** Output is a point cloud (requires splatting for rendering); spherical splats cannot handle sharp edges/corners; higher memory usage than octrees [S1][S2].

## Background
*   **Origin:** Proposed by P. Rosenthal and L. Linsen in 2006 [S2].
*   **Paper:** "Direct Isosurface Extraction from Scattered Volume Data" [S2].
*   **History:** Isosurface extraction is a common visualization technique. Previous methods often required structured grids or expensive 3D mesh generation (e.g., Delaunay tetrahedrization) [S2]. This method addresses the gap for direct extraction from scattered data without these steps [S2].
*   **Usage:** Implemented in the Danielhu/KdtreeISO example viewer for extracting iso-surfaces from volume data (tiff files) using oct/kd trees [S3, unofficial].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
[S2] Direct Isosurface Extraction from Scattered Volume Data, https://doi.org/10.2312/vissym/eurovis06/099-106, fetched 2026-10-26
[S3] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
