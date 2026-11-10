# Grid-based DBSCAN

Also called: DBSCAN 2D, Gunawan and Berg DBSCAN, O(n log n) DBSCAN for 2D

## Overview
Grid-based DBSCAN is an optimized variant of the DBSCAN clustering algorithm designed specifically for 2D spaces [S1, unofficial]. It solves the problem that standard DBSCAN implementations often fail to achieve optimal time complexity or produce discrepant results compared to the original algorithm due to the limitations of general index structures like VP-trees and KD-trees [S1, unofficial]. It sets itself apart from its siblings by mapping the dataset to a 2D grid and using graph connectivity to identify clusters, guaranteeing $O(n \log n)$ time complexity for 2D data, whereas [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md) relies on tree structures that may not guarantee this complexity or consistency [S1, unofficial].

## How it works
The method consists of four main steps: grid construction, core point determination, cluster combination, and border/noise assignment.

**Inputs and Outputs**
*   **Input:** A set of 2D points $P$, neighborhood radius $\varepsilon$, and minimum points parameter $minPts$.
*   **Output:** A clustering of $P$ where each point is labeled as a core-point, border-point, or noise, and assigned to a cluster ID.

**Data Structures**
*   **Grid $G$:** A 2D array or sparse hash table of dimensions $nRows \times nCols$. Each cell contains a list of points.
*   **Cell Side Length:** $cellSide = \varepsilon / \sqrt{2}$.
*   **Dimensions:**
    *   $nRows = \lfloor (maxX - minX) / cellSide \rfloor + 1$
    *   $nCols = \lfloor (maxY - minY) / cellSide \rfloor + 1$
    *   Where $minX, maxX, minY, maxY$ are the bounding box coordinates of $P$.

**Key Definitions**
*   **Definition 9:** For all points $p_a, p_b$ in cell $c$, $dist(p_a, p_b) \le \varepsilon$ [S1, unofficial]. This property ensures that any two points within the same cell are within the neighborhood distance $\varepsilon$.
*   **Definition 11:** Cell $c_i$ is connected to cell $c_j$ if $dist(p_a, p_b) \le \varepsilon$, where $p_a$ is a core-point in $c_i$ and $p_b$ is a core-point in $c_j$ [S1, unofficial].
*   **Definition 12:** Every connected component in the grid graph is a cluster [S1, unofficial].

**Parameters**
*   $\varepsilon$: The maximum distance for neighborhood queries. Used to determine cell side length and connectivity.
*   $minPts$: The minimum number of points required to form a dense region (core point) [S1, unofficial].

**Algorithm Steps**

1.  **Construct 2D Grid:**
    Initialize $minX, maxX, minY, maxY$ by scanning $P$. Calculate $cellSide$, $nRows$, and $nCols$. Initialize empty grid $G$. For each point $p \in P$, add $p$ to cell $G[\lfloor (p.x - minX) / cellSide \rfloor][\lfloor (p.y - minY) / cellSide \rfloor]$ [S1, unofficial].

2.  **Determine Core Points:**
    For each non-empty cell $c$ in $G$:
    *   If $|c| \ge minPts$, label all points in $c$ as core-points (based on Definition 9) [S1, unofficial].
    *   Else, for each point $p \in c$, count reachable points $nPoints$ by brute-forcing neighbors in the neighborhood $V_\varepsilon(c)$ (the 3x3 block of cells centered at $c$). If $nPoints \ge minPts$, label $p$ as core-point [S1, unofficial].

3.  **Combine Clusters:**
    Treat cells as nodes in a graph. Use a Connected Components algorithm (e.g., Tarjan's algorithm) to identify clusters. Two cells are connected if they contain core-points within distance $\varepsilon$ (Definition 11). Each connected component is a cluster [S1, unofficial].

4.  **Determine Border Points and Noise:**
    For each non-empty cell $c$ in $G$, for each point $p \in c$:
    *   If $p$ is not a core-point, find the nearest core-point in the neighborhood $V_\varepsilon(c)$.
    *   If no core-point is found within distance $\varepsilon$, label $p$ as noise.
    *   Otherwise, assign $p$ to the cluster of the nearest core-point.
    *   Cells with points $\ge minPts$ are skipped in this step as all their points are already core-points [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def grid_based_dbscan(P, epsilon, minPts):
    """
    P: List of (x, y) tuples
    epsilon: float, neighborhood radius
    minPts: int, minimum points for core point
    Returns: List of labels (cluster_id, -1 for noise)
    """
    if not P:
        return []

    # Step 1: Construct 2D Grid
    minX = min(p[0] for p in P)
    maxX = max(p[0] for p in P)
    minY = min(p[1] for p in P)
    maxY = max(p[1] for p in P)

    cellSide = epsilon / math.sqrt(2)
    nRows = math.floor((maxX - minX) / cellSide) + 1
    nCols = math.floor((maxY - minY) / cellSide) + 1

    # Initialize grid with empty lists
    grid = [[[] for _ in range(nCols)] for _ in range(nRows)]

    for p in P:
        col = math.floor((p[0] - minX) / cellSide)
        row = math.floor((p[1] - minY) / cellSide)
        # Handle boundary cases if point is exactly at maxX/maxY
        if col >= nCols: col = nCols - 1
        if row >= nRows: row = nRows - 1
        grid[row][col].append(p)

    # Helper to get neighbors in 3x3 neighborhood
    def get_cell_neighbors(row, col):
        neighbors = []
        for r in range(max(0, row - 1), min(nRows, row + 2)):
            for c in range(max(0, col - 1), min(nCols, col + 2)):
                neighbors.append((r, c))
        return neighbors

    # Step 2: Determine Core Points
    is_core = [[False for _ in range(nCols)] for _ in range(nRows)]
    # Store core points per cell for later use
    core_points_in_cell = [[[] for _ in range(nCols)] for _ in range(nRows)]

    for r in range(nRows):
        for c in range(nCols):
            cell_points = grid[r][c]
            if not cell_points:
                continue
            
            # Check if cell has enough points
            if len(cell_points) >= minPts:
                for p in cell_points:
                    is_core[r][c] = True
                    core_points_in_cell[r][c].append(p)
            else:
                # Brute force count neighbors in V_epsilon(c)
                nPoints = 0
                neighbors = get_cell_neighbors(r, c)
                for nr, nc in neighbors:
                    for q in grid[nr][nc]:
                        # Check distance for each point in p
                        pass # Logic below is per point p
                
                # Re-evaluate per point p as per Algorithm 6
                for p in cell_points:
                    nPoints = 0
                    neighbors = get_cell_neighbors(r, c)
                    for nr, nc in neighbors:
                        for q in grid[nr][nc]:
                            dist = math.sqrt((p[0]-q[0])**2 + (p[1]-q[1])**2)
                            if dist <= epsilon:
                                nPoints += 1
                    if nPoints >= minPts:
                        is_core[r][c] = True
                        core_points_in_cell[r][c].append(p)
                        break # Only need to know if p is core

    # Step 3: Combine Clusters using Connected Components on Cells
    # Build adjacency list for cells containing core points
    cell_graph = {}
    core_cells = []
    
    for r in range(nRows):
        for c in range(nCols):
            if is_core[r][c]:
                cell_graph[(r,c)] = []
                core_cells.append((r,c))

    # Connect cells if they have core points within epsilon
    # Optimization: Only check adjacent cells in grid for connectivity
    for r, c in core_cells:
        neighbors = get_cell_neighbors(r, c)
        for nr, nc in neighbors:
            if (nr, nc) in cell_graph and (nr, nc) != (r, c):
                # Check if there exists a pair of core points within epsilon
                connected = False
                for p in core_points_in_cell[r][c]:
                    for q in core_points_in_cell[nr][nc]:
                        dist = math.sqrt((p[0]-q[0])**2 + (p[1]-q[1])**2)
                        if dist <= epsilon:
                            connected = True
                            break
                    if connected:
                        break
                
                if connected:
                    cell_graph[(r,c)].append((nr, nc))
                    cell_graph[(nr, nc)].append((r, c))

    # Find Connected Components
    visited = set()
    components = [] # List of lists of cells
    
    def dfs(node, component):
        stack = [node]
        while stack:
            curr = stack.pop()
            if curr in visited:
                continue
            visited.add(curr)
            component.append(curr)
            for neighbor in cell_graph.get(curr, []):
                if neighbor not in visited:
                    stack.append(neighbor)

    for cell in core_cells:
        if cell not in visited:
            component = []
            dfs(cell, component)
            components.append(component)

    # Assign cluster IDs to core points
    point_to_cluster = {}
    for i, component in enumerate(components):
        cluster_id = i + 1
        for r, c in component:
            for p in core_points_in_cell[r][c]:
                point_to_cluster[p] = cluster_id

    # Step 4: Determine Border Points and Noise
    labels = {}
    
    # Initialize all points as unclassified
    for p in P:
        labels[p] = None

    # Assign core points
    for p, cid in point_to_cluster.items():
        labels[p] = cid

    # Assign border points and noise
    for r in range(nRows):
        for c in range(nCols):
            cell_points = grid[r][c]
            if len(cell_points) >= minPts:
                continue # Skip cells where all points are core points
            
            for p in cell_points:
                if labels[p] is not None:
                    continue # Already a core point
                
                nearest_core = None
                min_dist = float('inf')
                
                neighbors = get_cell_neighbors(r, c)
                for nr, nc in neighbors:
                    # Check all core points in neighbor cells
                    for q in core_points_in_cell[nr][nc]:
                        dist = math.sqrt((p[0]-q[0])**2 + (p[1]-q[1])**2)
                        if dist <= epsilon:
                            if dist < min_dist:
                                min_dist = dist
                                nearest_core = q
                
                if nearest_core is None:
                    labels[p] = -1 # Noise
                else:
                    labels[p] = point_to_cluster[nearest_core]

    # Return labels in original order if needed, or just the dict
    return labels
```

**Edge Cases and Pitfalls**
*   Standard index structures (VP-trees, KD-trees) do not guarantee $O(n \log n)$ execution time for DBSCAN or may produce discrepant results compared to the original algorithm [S1, unofficial].
*   For dimensions $d \ge 3$, the theoretical minimum complexity is $\Omega(n^{4/3})$, not $O(n \log n)$ [S1, unofficial].
*   Points exactly on the boundary of the bounding box must be handled correctly when calculating grid indices to avoid out-of-bounds errors [G].

## Operating characteristics
*   **Time Complexity:** $O(n \log n)$ for 2D spaces [S1, unofficial]. This is the theoretical minimum for $d \ge 3$ is $\Omega(n^{4/3})$ [S1, unofficial]. In practice, complexity can be $O(n^2)$ if parameters are not chosen appropriately [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of points and the grid size. The grid size depends on the bounding box and $\varepsilon$.
*   **Strengths:** Guarantees $O(n \log n)$ time complexity for 2D data, which is faster than standard DBSCAN implementations for large datasets. Accurately solves the original DBSCAN problem for 2D spaces [S1, unofficial].
*   **Limitations:** Only applicable to 2D spaces. For higher dimensions, the complexity advantage is lost or reduced.
*   **When to use:** When clustering 2D point clouds and performance is critical, and consistency with the original DBSCAN definition is required.
*   **When not to use:** For 3D or higher-dimensional data, or when memory is extremely constrained and the bounding box is very large relative to point density.

## Background
The original DBSCAN algorithm was proposed by Martin Ester, Hans-Peter Kriegel, Jörg Sander, and Xiaowei Xu in 1996 [S1, unofficial]. They suggested that index structures like VP-trees and KD-trees could reduce execution time to $O(n \log n)$ [S1, unofficial]. However, later research by Gunawan and Berg (2013), Gan and Tao (2015), and Schubert et al. (2017) demonstrated that this claim was flawed, as most strategies guaranteeing low execution time produce discrepant results or fail to achieve the claimed complexity [S1, unofficial]. Gunawan and Berg (2013) proposed the grid-based version for 2D spaces, which accurately solves the original problem in $O(n \log n)$ time [S1, unofficial]. This method is used in volume exploration approaches based on clustering and dimensional reduction to support the definition of multidimensional transfer functions [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
