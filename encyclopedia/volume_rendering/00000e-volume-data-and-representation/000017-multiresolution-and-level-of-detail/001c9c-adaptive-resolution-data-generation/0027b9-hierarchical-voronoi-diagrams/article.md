# Hierarchical Voronoi Diagrams

Also called: Adaptive Voronoi clustering, Voronoi-based clustering

## Overview
Hierarchical Voronoi Diagrams is an adaptive clustering method for the multiresolution representation of scattered data [S1, unofficial]. It constructs a hierarchy of Voronoi diagrams from nested subsets of points to approximate scalar fields, building on [Adaptive Resolution Data Generation](1331-adaptive-resolution-data-generation.md) by defining approximation regions and error metrics for multiresolution representation. It differs from mesh-reduction algorithms by not requiring a grid structure, making it suitable for unstructured datasets [S1, unofficial].

## How it works
**Inputs:** A set of $n$ scattered data points $P = \{\mathbf{p}_i\}_{i=1}^n$ in $\mathbb{R}^k$ with associated scalar values $f_i = f(\mathbf{p}_i)$. A target global error bound $\epsilon_{target}$ or a maximum number of points $n_{max}$.
**Outputs:** A hierarchy of Voronoi diagrams $L_0, L_1, \dots, L_J$ and corresponding piecewise interpolants.

**Data Structures:**
*   **Voronoi Diagram:** A partition of space into tiles $\tau_i$. Each tile $\tau_i$ is associated with a center point $\mathbf{p}_i$.
*   **Residuals:** For each tile $\tau_k^j$ at level $j$, store the squared residual $(\epsilon_k^j)^2$ to avoid square root computations.
*   **Adjacency:** A map of adjacent tiles for each tile to support efficient updates.

**Definitions:**
*   **Voronoi Tile:** Tile $\tau_i$ is the subset of $\mathbb{R}^k$ containing all points closer to $\mathbf{p}_i$ than to any $\mathbf{p}_j$ ($j \neq i$) with respect to the Euclidean norm [S1, unofficial].
*   **Sibson's Interpolant:** For a point $\mathbf{p}$, the interpolated value is:
    $$ \tilde{f}(\mathbf{p}) = \frac{\sum_i a_i f_i}{\sum_i a_i} $$
    where $a_i$ are the areas/volumes cut away from Voronoi tiles $\tau_i$ when inserting $\mathbf{p}$ temporarily into the diagram [S1, unofficial].
*   **Domain Clipping:** To avoid infinite $a_i$, clip the Voronoi diagram against the boundary of a compact domain $D$, typically the convex hull of $P$ [S1, unofficial].
*   **Local Residual:** For tile $\tau_k^j$ at level $j$, the residual is computed using error norm (2.1) with $p=2$ [S1, unofficial].
*   **Global Error:** The global approximation error at level $j$ is:
    $$ \epsilon^j = \left( \frac{1}{n} \sum_{k=1}^{n_j} n_k^j (\epsilon_k^j)^p \right)^{\frac{1}{p}} \quad (2.2) $$
    where $n_j$ is the number of tiles at level $j$, and $n_k^j$ is the number of original data points contained in tile $\tau_k^j$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization ($L_0$):** Construct the Voronoi diagram for the minimal point set defining the convex hull of all points $P$. The tiles of this diagram define the cluster regions $\tau_k^0$ of level $L_0$.
2.  **Initial Residuals:** Compute all residuals $\epsilon_k^0$ using Sibson's interpolant $\tilde{f}_k^0$ and error norm (2.1) with $p=2$. Store $(\epsilon_k^0)^2$.
3.  **Refinement Loop:**
    a.  Identify the tile $\tau_m^j$ with the maximal residual $\epsilon_m^j$ in the current level $L_j$.
    b.  Among all original data points $\mathbf{p}_i \in \tau_m^j$, identify the point $\mathbf{p}_{max}$ with the maximal error $\|\tilde{f}_m^j(\mathbf{p}_i) - f_i\|$.
    c.  Insert $\mathbf{p}_{max}$ into the Voronoi diagram. This splits $\tau_m^j$ into new tiles, one of which has center $\mathbf{p}_{max}$ (denoted $\tau_{max}^{j+1}$).
    d.  Update the residual $\epsilon_{max}^{j+1}$ for the new tile $\tau_{max}^{j+1}$.
    e.  Update residuals for all tiles adjacent to $\tau_{max}^{j+1}$ because their Sibson interpolants change due to the new neighbor.
    f.  All other clusters remain unchanged: $\tau_i^{j+1} = \tau_i^j$, $\tilde{f}_i^{j+1} = \tilde{f}_i^j$, and $\epsilon_i^{j+1} = \epsilon_i^j$.
4.  **Termination Check:** Compute the global error $\epsilon^j$ using (2.2). Terminate if $\epsilon^j \le \epsilon_{target}$ or the number of inserted points reaches $n_{max}$. Otherwise, increment $j$ and repeat step 3.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_voronoi_clustering(points, values, domain_hull, max_error=None, max_points=None):
    """
    points: list of (x, y, z) coordinates
    values: list of scalar values f(p)
    domain_hull: convex hull of points for clipping
    max_error: global error threshold
    max_points: maximum number of Voronoi centers
    """
    # Step 1: Initialize with convex hull points
    hull_points = get_convex_hull(points)
    hull_values = get_values_at(hull_points, points, values)
    
    # Voronoi diagram state
    centers = hull_points
    center_values = hull_values
    voronoi = build_voronoi(centers, domain_hull)
    
    # Compute initial residuals
    residuals = compute_residuals(voronoi, centers, center_values, points, values, p=2)
    
    j = 0
    while True:
        # Step 4: Check termination
        global_error = compute_global_error(residuals, points, p=2)
        if max_error is not None and global_error <= max_error:
            break
        if max_points is not None and len(centers) >= max_points:
            break
            
        # Step 3a: Find max residual tile
        max_tile_idx = argmax(residuals)
        tile = voronoi.tiles[max_tile_idx]
        
        # Step 3b: Find point with max error in tile
        points_in_tile = get_points_in_tile(tile, points)
        max_error_point_idx = argmax([abs(sibson_interpolate(p, tile, centers, center_values, domain_hull) - values[i]) 
                                      for i, p in enumerate(points_in_tile)])
        p_max = points_in_tile[max_error_point_idx]
        f_max = values[points.index(p_max)]
        
        # Step 3c: Insert p_max
        centers.append(p_max)
        center_values.append(f_max)
        voronoi = update_voronoi(voronoi, p_max, domain_hull)
        
        # Step 3d, e: Update residuals
        # Identify affected tiles (new tile and neighbors)
        affected_tiles = get_affected_tiles(voronoi, p_max)
        
        for tile_idx in affected_tiles:
            tile = voronoi.tiles[tile_idx]
            # Recompute Sibson interpolant for this tile's region
            # Recompute residual for this tile
            residuals[tile_idx] = compute_tile_residual(tile, centers, center_values, points, values, p=2)
            
        j += 1
        
    return centers, center_values, voronoi
```

**Edge Cases and Pitfalls:**
*   **Infinite Areas:** Sibson's interpolant requires clipping the Voronoi diagram against the domain boundary $D$ (e.g., convex hull) to prevent infinite areas/volumes $a_i$ [S1, unofficial].
*   **Non-unique Delaunay:** If more than three points are cocircular, the Delaunay triangulation is not unique, but the Voronoi diagram is unique [S1, unofficial].
*   **Precision:** Storing squared residuals avoids square root computations but requires careful handling of the global error formula (2.2) which involves a $p$-th root [S1, unofficial].

## Subtopics

### [Adaptive Clustering](4979-adaptive-clustering.md)
Adaptive clustering is a technique for generating a data-dependent hierarchy that represents the inherent topological and geometric structures of scattered data by recursively partitioning a set of samples into smaller groups (clusters) based on spatial adjacency or correlation of associated function values. Unlike mesh-reduction algorithms, it does not require a grid structure connecting the given data points, making it suitable for unstructured or scattered datasets. It builds on [Hierarchical Voronoi Diagrams](4112-hierarchical-voronoi-diagrams.md) by using the hierarchy to define approximation regions and error metrics for multiresolution representation.

## Operating characteristics
*   **Time Complexity:** Each refinement step involves inserting a point into the Voronoi diagram ($O(\log n)$ or $O(n)$ depending on implementation) and updating adjacent tiles. The number of steps is determined by the error bound or point limit.
*   **Memory Complexity:** Proportional to the number of Voronoi tiles and the adjacency structure.
*   **Performance:** For dataset f1 with 100 Voronoi tiles, Error (p=1) is 6.0% and Error (p=2) is 7.4%. With 1000 tiles, Error (p=1) is 1.0% and Error (p=2) is 1.2% [S1, unofficial]. For Crater Lake terrain data (159,272 samples), with 1000 Voronoi tiles, Error (p=$\infty$) is 10.8% and Error (p=2) is 0.80% [S1, unofficial].
*   **Strengths:** Does not require a grid structure; suitable for unstructured/scattered data; adaptive resolution based on error.
*   **Limitations:** Requires convex hull computation; Sibson's interpolant can be computationally expensive per evaluation; non-unique Delaunay triangulation in degenerate cases.

## Background
Hierarchical Voronoi Diagrams were proposed as a new adaptive clustering approach for multiresolution representation of scattered data [S1, unofficial]. The method uses a hierarchy of Voronoi diagrams constructed from nested subsets of the original set of points [S1, unofficial]. It is based on the relationship between Voronoi diagrams and Delaunay triangulations, where Voronoi vertices are at the centers of circumscribed circles of Delaunay triangles [S1, unofficial]. The approach was applied to analytical functions, Hubble images, and terrain data [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
