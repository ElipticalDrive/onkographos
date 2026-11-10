# Volume-Based Topology Simplification via Augmented Reeb Graph

Also called: Augmented Reeb Graph Handle Detection, Augmented Reeb Graph Method, Intraribbon handle identification, Reeb graph cycle detection, Sweep-based Topology Simplification

## Overview
This method removes topological noise (specifically handles) from isosurfaces extracted from volumetric scalar data by sweeping through the volume, encoding topology in an augmented Reeb graph, isolating handles, measuring their size, and removing small ones [S1]. It differs from polygon-by-polygon traversal methods by using an axis-aligned sweep that allows for out-of-core processing and produces a less dense Reeb graph, and from cut-graph methods by searching for nonseparating cuts locally rather than analyzing the entire surface [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A regularly sampled 3D grid of scalar values $F(x)$. The interior is defined as $F(x) < 0$ and the exterior as $F(x) \ge 0$ [S1].
*   **Output:** A topologically simplified isosurface mesh with small handles removed [S1].

**Data Structures:**
*   **Grid Cube:** Bounded by 8 grid data points. Contains up to 4 polygons from the isosurface generation algorithm [S1].
*   **Slice:** The set of grid cubes between two adjacent data planes (z-intervals) [S1].
*   **Contour:** Oriented closed polylines formed by the intersection of the isosurface with a planar cross-section (data plane) [S1].
*   **Ribbon:** A connected component of the surface within a slice. Its boundaries consist of one or more contours in the two adjacent planes [S1].
*   **Augmented Reeb Graph:** A graph storing geometric information for each contour and ribbon. Nodes correspond to contours and ribbons; edges represent adjacency. Cycles in this graph correspond to handles [S1].
*   **Disjoint-Set Data Structure:** Used to manage component labels for ribbons and contours [S1].

**Connectivity Rule:**
The method uses Lachaud [1996] connectivity rules for isosurface generation to ensure a closed oriented surface without singularities or self-intersections [S1].

**Algorithm Steps:**

1.  **Sweep and Encode Topology:**
    Sweep through the volume along parallel data planes (z-intervals). Within each slice, identify ribbons and contours using breadth-first search (BFS) [S1].
    *   **Contour Construction:** Search from an arbitrary edge in the plane until the contour is closed [S1].
    *   **Ribbon Construction:** Run a BFS in the slice starting with polygons adjacent to one contour and ending with polygons adjacent to the previous contour [S1].
    *   **Graph Construction:** Create nodes in the Reeb graph for ribbons and contours. Record adjacency as graph edges [S1]. Associate a label with every ribbon and contour node to identify the connected component [S1].

2.  **Detect Intraribbon Handles (Optional/Conditional):**
    Standard z-interval traversal may miss handles entirely contained within a single ribbon (slice). These occur in 1–10% of slices [S1].
    *   Compute the Euler characteristic $\chi$ of the isosurface within each z-interval: $\chi = |V| - |E| + |F|$ [S1].
    *   If the slice has nonzero genus ($g = (2-\chi)/2 > 0$), it contains handles [S1].
    *   For such slices, analyze the distance function on a polygon-by-polygon basis using a BFS over polygons in the current slice [S1].
    *   Modify contour construction to be defined by this polygon BFS so the Reeb graph encodes the intra-ribbon handle topology [S1].
    *   **Final Check:** Confirm overall correspondence by ensuring a match between the Euler characteristic of the surface traversed thus far and the number of cycles in the Reeb graph [S1].

3.  **Isolate Handles:**
    Detect handles as cycles in the Reeb graph formed when two contour nodes with the same connected component label are merged into a single ribbon node [S1].
    *   When adding a ribbon node $r$ and its adjacent previous contours $C$, check if any pair of contours $c_1, c_2 \in C$ have the same label [S1].
    *   If $label(c_1) = label(c_2)$, a Reeb cycle is formed [S1].
    *   Isolate the handle by performing a BFS through the graph to find the shortest graph cycle between $c_1$ and $c_2$ [S1].
    *   The Reeb cycle path consists of alternating ribbon and contour nodes and defines the handle [S1].
    *   If a ribbon has $k$ child contours with the same component labels ($k \ge 2$), consider all pairs of child contours to test reasonable ways to isolate a handle [S1].

4.  **Measure and Remove:**
    *   For each handle found, measure its size [S1].
    *   If the size is sufficiently small, remove the handle [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
function AddRibbonToReebGraph(ribbon r, ReebGraph G):
    # Add ribbon r as node in graph G
    G.addNode(r)
    
    # Assign a unique label to the new ribbon
    label(r) = generateUniqueLabel()
    
    # Identify all previous contours C adjacent to r on the surface
    C = getAdjacentContours(r)
    
    # Check for cycles (handles) among pairs of adjacent contours
    for each pair (c1, c2) in combinations(C, 2):
        if label(c1) == label(c2):
            # A cycle is detected: two contours of the same component merge
            # Find the shortest path between c1 and c2 in the existing graph
            P = shortestPath(c1, c2, G)
            
            # Report the Reeb cycle. The cycle consists of the path P,
            # the edge from c1 to r, and the edge from r to c2.
            # Note: The source defines the cycle as (c2, r) + (r, c1) + P
            cycle = (c2, r) + (r, c1) + P
            reportReebCycle(cycle)
            
            # Isolate the handle using the cycle path
            isolateHandle(cycle)
    
    # Add edges from all adjacent contours to the new ribbon
    for each contour c in C:
        G.addEdge(c, r)
        
        # Unify labels of contour c and ribbon r to reflect merging
        unifyLabels(c, r)

function unifyLabels(node1, node2):
    # Implemented using a Union-Find algorithm on a disjoint-set data structure
    root1 = findSet(node1)
    root2 = findSet(node2)
    if root1 != root2:
        unionSets(root1, root2)
```

**Edge Cases and Pitfalls:**
*   **Intraribbon Handles:** Handles entirely contained within a single ribbon (slice) are not detected by the standard z-interval traversal. They must be detected by computing the Euler characteristic and performing a polygon-by-polygon BFS within that slice [S1].
*   **Multiple Saddles:** A ribbon may have $k \ge 2$ child contours with the same component label. This corresponds to a region with a degenerate critical point (multiple saddle). The algorithm must consider all pairs of child contours to find the best nonseparating cut [S1].
*   **Precision/Boundaries:** The method relies on the connectivity rules of Lachaud [1996] to ensure a closed oriented surface. Deviations from these rules may result in singularities or self-intersections [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Supports out-of-core processing due to ordered traversal through slices, keeping surface access local [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The z-interval sweep allows out-of-core processing, unlike polygon-by-polygon traversal which requires irregular data access [S1].
*   **Strengths:** Out-of-core capability; less dense Reeb graph than polygon-by-polygon traversal; fast approximations of nonseparating cuts [S1].
*   **Limitations:** Not guaranteed to find globally minimum cuts (local search only) [S1].
*   **When to use:** When processing large volumes that do not fit in memory, or when a fast approximation of topology simplification is sufficient [S1].
*   **When not to use:** When globally minimal cuts are required, or when the surface topology is simple enough that local simplification is unnecessary [S1].

## Background
The method was introduced by [S1] (Wood et al., 2004) as a way to remove excess topology from isosurfaces. It builds on the concept of Reeb graphs for topology analysis [S1]. The approach contrasts with cut-graph methods (e.g., [S1]) which analyze the entire surface to find nearly-minimal cuts, whereas this method searches for nonseparating cuts locally, one handle at a time [S1].

## Sources
[S1] Removing excess topology from isosurfaces, https://doi.org/10.1145/990002.990007, fetched 2026-11-03
