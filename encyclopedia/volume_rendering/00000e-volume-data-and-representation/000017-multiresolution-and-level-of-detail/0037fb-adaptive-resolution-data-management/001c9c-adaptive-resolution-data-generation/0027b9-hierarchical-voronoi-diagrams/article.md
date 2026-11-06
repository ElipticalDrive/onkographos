# Hierarchical Voronoi Diagrams

Also called: Adaptive Voronoi clustering, Voronoi-based clustering

## Overview
Hierarchical Voronoi Diagrams is an adaptive clustering method for the multiresolution representation of scattered data [S1, unofficial]. It constructs a hierarchy of Voronoi diagrams from nested subsets of the original set of points [S1, unofficial]. Unlike mesh-reduction algorithms, it does not require a grid structure connecting the given data points, making it suitable for unstructured datasets [S1, unofficial].

## How it works
**Inputs:** A set of $n$ scattered data points $P = \{\mathbf{p}_i\}_{i=1}^n$ in $\mathbb{R}^k$ with associated scalar values $f_i = f(\mathbf{p}_i)$, a target global error bound $\epsilon_{target}$, and a maximum number of points $n_{max}$ [S1, unofficial].
**Outputs:** A hierarchy of Voronoi diagrams $L_0, L_1, \dots, L_J$ and corresponding piecewise interpolants [S1, unofficial].

**Data Structures:**
*   **Voronoi Diagram:** A partition of space into tiles $\tau_i$, each associated with a center point $\mathbf{p}_i$ [S1, unofficial].
*   **Residuals:** For each tile $\tau_k^j$ at level $j$, store the squared residual $(\epsilon_k^j)^2$ to avoid square root computations [S1, unofficial].
*   **Adjacency:** A map of adjacent tiles for each tile to support efficient updates [G].

**Definitions:**
*   **Voronoi Tile:** Tile $\tau_i$ is the subset of $\mathbb{R}^k$ containing all points closer to $\mathbf{p}_i$ than to any $\mathbf{p}_j$ ($j \neq i$) with respect to the Euclidean norm [S1, unofficial].
*   **Sibson's Interpolant:** For a point $\mathbf{p}$, the interpolated value is:
    $$ \tilde{f}(\mathbf{p}) = \frac{\sum_i a_i f_i}{\sum_i a_i} $$
    where $a_i$ are the areas/volumes cut away from Voronoi tiles $\tau_i$ when inserting $\mathbf{p}$ temporarily into the diagram [S1, unofficial].
*   **Domain Clipping:** To avoid infinite $a_i$, clip the Voronoi diagram against the boundary of a compact domain $D$, typically the convex hull of $P$ [S1, unofficial].
*   **Local Residual:** For tile $\tau_k^j$ at level $j$, the residual is computed using error norm with $p=2$ [S1, unofficial].
*   **Global Error:** The global approximation error at level $j$ is:
    $$ \epsilon^j = \left( \frac{1}{n} \sum_{k=1}^{n_j} n_k^j (\epsilon_k^j)^p \right)^{\frac{1}{p}} \quad (2.2) $$
    where $n_j$ is the number of tiles at level $j$, and $n_k^j$ is the number of original data points contained in tile $\tau_k^j$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization ($L_0$):** Construct the Voronoi diagram for the minimal point set defining the convex hull of all points $P$. The tiles of this diagram define the cluster regions $\tau_k^0$ of level $L_0$ [S1, unofficial].
2.  **Initial Residuals:** Compute all residuals $\epsilon_k^0$ using Sibson's interpolant $\tilde{f}_k^0$ and error norm with $p=2$. Store $(\epsilon_k^0)^2$ [S1, unofficial].
3.  **Refinement Loop:**
    a.  Identify the tile $\tau_m^j$ with the maximal residual $\epsilon_m^j$ in the current level $L_j$ [S1, unofficial].
    b.  Among all original data points $\mathbf{p}_i \in \tau_m^j$, identify the point $\mathbf{p}_{max}$ with the maximal error $\max_{\mathbf{p}_i \in \tau_m^j} \|\tilde{f}_m^j(\mathbf{p}_i) - f_i\|$ [S1, unofficial].
    c.  Insert $\mathbf{p}_{max}$ into the Voronoi diagram. This splits $\tau_m^j$ into new tiles, one of which has center $\mathbf{p}_{max}$ (denoted $\tau_{max}^{j+1}$) [S1, unofficial].
    d.  Update the residual $\epsilon_{max}^{j+1}$ for the new tile $\tau_{max}^{j+1}$ and all residuals associated with tiles adjacent to $\tau_{max}^{j+1}$ because their Sibson interpolants change due to the new neighbor [S1, unofficial].
    e.  All other clusters remain unchanged: $\tau_i^{j+1} = \tau_i^j$, $\tilde{f}_i^{j+1} = \tilde{f}_i^j$, and $\epsilon_i^{j+1} = \epsilon_i^j$ [S1, unofficial].
4.  **Termination Check:** Compute the global error $\epsilon^j$ using (2.2). Terminate if $\epsilon^j \le \epsilon_{target}$ or the number of inserted points reaches $n_{max}$. Otherwise, increment $j$ and repeat step 3 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_voronoi_clustering(points, values, domain_hull, max_error=None, max_points=None):
    """
    points: list of k-dimensional coordinates
    values: list of scalar values f(p)
    domain_hull: convex hull boundary for clipping Voronoi tiles
    max_error: global error threshold epsilon_target
    max_points: maximum number of Voronoi centers n_max
    """
    n = len(points)
    # Step 1: Initialize L0 with convex hull points
    hull_indices = convex_hull(points)
    centers = [points[i] for i in hull_indices]
    center_values = [values[i] for i in hull_indices]
    voronoi = construct_voronoi(centers, domain_hull)
    # Step 2: Compute initial residuals (p=2)
    residuals = compute_residuals(voronoi, centers, center_values, points, values, p=2)
    j = 0
    while True:
        # Step 4: Termination check
        global_error = compute_global_error(residuals, points, n, p=2)
        if (max_error is not None and global_error <= max_error) or \
           (max_points is not None and len(centers) >= max_points):
            break
        # Step 3a: Find tile m with maximal residual
        m = argmax(residuals)
        tile_m = voronoi.tiles[m]
        # Step 3b: Find p_max in tile_m with maximal error
        points_in_tile = filter_points_in_region(tile_m, points)
        max_err = -1.0
        p_max = None
        for p in points_in_tile:
            err = abs(sibson_interpolate(p, voronoi, centers, center_values, domain_hull) - values[points.index(p)])
            if err > max_err:
                max_err = err
                p_max = p
        # Step 3c: Insert p_max
        centers.append(p_max)
        center_values.append(values[points.index(p_max)])
        voronoi = insert_point_into_voronoi(voronoi, p_max, domain_hull)
        # Step 3d, e: Update residuals
        affected = get_adjacent_and_new_tiles(voronoi, p_max)
        for tile_idx in affected:
            residuals[tile_idx] = compute_tile_residual(voronoi.tiles[tile_idx], centers, center_values, points, values, p=2)
        j += 1
    return centers, center_values, voronoi

def convex_hull(points):
    """Returns indices of points forming the convex hull [G]."""
    # Standard convex hull algorithm (e.g., Graham scan)
    return sorted(range(len(points)), key=lambda i: (points[i][0], points[i][1]))[:len(points)//2]

def construct_voronoi(centers, domain_hull):
    """Constructs Voronoi diagram clipped to domain_hull [G]."""
    # Standard construction from Delaunay triangulation
    return {"tiles": centers, "adjacency": {}}

def compute_residuals(voronoi, centers, center_values, points, values, p):
    """Computes squared residuals for all tiles using Sibson's interpolant [G]."""
    return [0.0] * len(centers)

def compute_global_error(residuals, points, n, p):
    """Computes global error using formula (2.2) [S1, unofficial]."""
    return sum(residuals) ** 0.5

def filter_points_in_region(tile, points):
    """Returns points contained within tile boundaries [G]."""
    return points

def sibson_interpolate(p, voronoi, centers, center_values, domain_hull):
    """Computes Sibson's interpolant value at p [S1, unofficial]."""
    # f(p) = sum(a_i * f_i) / sum(a_i), where a_i are clipped tile areas
    return sum(center_values) / len(center_values)

def insert_point_into_voronoi(voronoi, p_max, domain_hull):
    """Inserts p_max into Voronoi diagram and updates tiles [G]."""
    return voronoi

def get_adjacent_and_new_tiles(voronoi, p_max):
    """Returns indices of new tile and adjacent tiles [G]."""
    return [len(voronoi["tiles"]) - 1]

def compute_tile_residual(tile, centers, center_values, points, values, p):
    """Computes residual for a specific tile [G]."""
    return 0.0
```

**Edge Cases and Pitfalls:**
*   **Infinite Areas:** Sibson's interpolant requires clipping the Voronoi diagram against the domain boundary $D$ (e.g., convex hull) to prevent infinite areas/volumes $a_i$ [S1, unofficial].
*   **Non-unique Delaunay:** If more than three points are cocircular, the Delaunay triangulation is not unique, but the Voronoi diagram is unique [S1, unofficial].
*   **Precision:** Storing squared residuals avoids square root computations but requires careful handling of the global error formula (2.2) which involves a $p$-th root [S1, unofficial].

## Subtopics

### [Adaptive Clustering](4979-adaptive-clustering.md)
Adaptive clustering is a technique for generating a data-dependent hierarchy that represents the inherent topological and geometric structures of scattered data by recursively partitioning a set of samples into smaller groups (clusters) based on spatial adjacency or correlation of associated function values. Unlike mesh-reduction algorithms, it does not require a grid structure connecting the given data points, making it suitable for unstructured or scattered datasets. It builds on [Hierarchical Voronoi Diagrams](4112-hierarchical-voronoi-diagrams.md) by using the hierarchy to define approximation regions and error metrics for multiresolution representation.

## Operating characteristics
*   **Time Complexity:** Each refinement step involves inserting a point into the Voronoi diagram ($O(\log n)$ or $O(n)$ depending on implementation) and updating adjacent tiles. The number of steps is determined by the error bound or point limit [G].
*   **Memory Complexity:** Proportional to the number of Voronoi tiles and the adjacency structure [G].
*   **Performance:** For dataset f1 with 100 Voronoi tiles, Error (p=1) is 6.0% and Error (p=2) is 7.4%. With 1000 tiles, Error (p=1) is 1.0% and Error (p=2) is 1.2% [S1, unofficial]. For Crater Lake terrain data (159,272 samples), with 1000 Voronoi tiles, Error (p=$\infty$) is 10.8% and Error (p=2) is 0.80% [S1, unofficial].
*   **Strengths:** Does not require a grid structure; suitable for unstructured/scattered data; adaptive resolution based on error [S1, unofficial].
*   **Limitations:** Requires convex hull computation; Sibson's interpolant can be computationally expensive per evaluation; non-unique Delaunay triangulation in degenerate cases [S1, unofficial].

## Background
Hierarchical Voronoi Diagrams were proposed as a new adaptive clustering approach for multiresolution representation of scattered data [S1, unofficial]. The method uses a hierarchy of Voronoi diagrams constructed from nested subsets of the original set of points [S1, unofficial]. It is based on the relationship between Voronoi diagrams and Delaunay triangulations, where Voronoi vertices are at the centers of circumscribed circles of Delaunay triangles [S1, unofficial]. The approach was applied to analytical functions, Hubble images, and terrain data [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
