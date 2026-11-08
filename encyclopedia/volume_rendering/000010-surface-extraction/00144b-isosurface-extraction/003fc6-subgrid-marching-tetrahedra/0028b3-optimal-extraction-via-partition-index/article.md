# Optimal Extraction via Partition Index

Also called: OptimalExtraction, Saddle-based Partitioning

## Overview
Optimal Extraction via Partition Index is a method for extracting isosurfaces from a tetrahedral grid that guarantees visiting only the tetrahedra actually intersected by the level set, thereby achieving optimal runtime complexity for the extraction phase [S1, unofficial]. It distinguishes itself from standard marching tetrahedra by pre-processing the mesh into partitions separated by saddle contours, allowing the algorithm to jump directly to connected components of the isosurface via a partition index [S1, unofficial]. The method builds on [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) to guarantee intersection-free output.

## How it works
**Inputs and Outputs**
*   **Input:** A tetrahedral mesh with a scalar field defined on vertices; an isovalue ($v_{\text{iso}}$) [S1, unofficial].
*   **Output:** A set of triangles forming the isosurface, representing connected components without self-intersections [S1, unofficial].

**Data Structures**
1.  **Partition:** A collection of vertices belonging to a region where the isosurface forms a single connected component [S1, unofficial].
    *   **Layout:** `std::set<std::pair<VertexID, ScalarValue>>` [S1, unofficial].
    *   **Ordering:** Ordered by `ScalarValue` in ascending order [S1, unofficial].
2.  **PartitionIndex:** An interval table mapping scalar value ranges to the partitions intersecting those ranges [S1, unofficial].
    *   **Layout:** A sorted list of entries $(v_{\min}, v_{\max}, \text{Partition})$ where $v_{\min}$ and $v_{\max}$ are the minimum and maximum scalar values in the partition [S1, unofficial].

**Definitions**
*   **Regular Vertex:** A vertex where the set of direct neighbors with function values lower than the vertex forms a single connected component AND the set of direct neighbors with function values higher than the vertex forms a single connected component [S1, unofficial].
*   **Saddle Vertex:** Any vertex that is not regular, including minima, maxima, and vertices where the lower or higher neighbor groups are disconnected [S1, unofficial].
*   **Saddle Contour:** The topological boundary separating regions of regular vertices, passing through saddle vertices [S1, unofficial].

**Algorithm Steps**
1.  **Identify Saddles:** Iterate over all vertices in the mesh. For each vertex, check the connectivity of its neighbors with values strictly lower and strictly higher than the vertex's value. If either group is not a single connected component, mark the vertex as a saddle [S1, unofficial].
2.  **Partition the Mesh:** Initialize an empty list of partitions and a `visited` set for vertices. Iterate through all regular vertices. If a vertex is not visited, create a new partition and perform a flood fill starting from this vertex. Add a neighbor to the current partition if and only if the edge connecting the current vertex and the neighbor does not cross a saddle contour. Mark all added vertices as visited and add them to the partition. Repeat until all regular vertices are assigned to a partition [S1, unofficial].
3.  **Build PartitionIndex:** For each partition, determine the minimum and maximum scalar values among its vertices. Insert an entry into the `PartitionIndex` interval table mapping the interval $[v_{\min}, v_{\max}]$ to the partition [S1, unofficial].
4.  **Query Intersected Partitions:** Given $v_{\text{iso}}$, query the `PartitionIndex` to retrieve all partitions whose scalar range contains $v_{\text{iso}}$ [S1, unofficial].
5.  **Find Seed Vertices and Tetrahedra:** For each intersected partition, use `std::set::lower_bound()` on the partition's vertex set to find the vertex with the smallest function value $\ge v_{\text{iso}}$. This is the seed vertex [S1, unofficial]. Identify all tetrahedra containing the seed vertex. Check each tetrahedron using `IsCellOnLevelSet()` (i.e., does the tetrahedron's minimum and maximum scalar values bracket $v_{\text{iso}}$?). Select one such tetrahedron as the seed tetrahedron [S1, unofficial].
6.  **Extract Contour:** Call `ExtractContour(seed_tetrahedron)` for each seed tetrahedron found [S1, unofficial]. `ExtractContour` initializes a queue with the seed tetrahedron ID. While the queue is not empty, pop tetrahedron $T$, compute its intersection with the level set, and add unvisited neighbors whose function span intersects $v_{\text{iso}}$ to the queue. Repeat until the queue is empty [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def OptimalExtraction(mesh, scalar_field, isovalue):
    # Step 1: Identify Saddles
    saddles = set()
    for v in mesh.vertices:
        lower = [n for n in mesh.neighbors(v) if scalar_field[n] < scalar_field[v]]
        higher = [n for n in mesh.neighbors(v) if scalar_field[n] > scalar_field[v]]
        if not is_connected(lower, mesh.edges) or not is_connected(higher, mesh.edges):
            saddles.add(v)

    # Step 2: Partition Mesh
    partitions = []
    visited = set()
    for v in mesh.vertices:
        if v in saddles or v in visited:
            continue
        partition = []
        q = [v]
        visited.add(v)
        while q:
            curr = q.pop(0)
            partition.append((curr, scalar_field[curr]))
            for n in mesh.neighbors(curr):
                if n in visited or n in saddles:
                    continue
                visited.add(n)
                q.append(n)
        partition.sort(key=lambda x: x[1])  # Ordered by function value [S1, unofficial]
        partitions.append(partition)

    # Step 3: Build PartitionIndex (Interval Table)
    index = []
    for part in partitions:
        min_val = part[0][1]
        max_val = part[-1][1]
        index.append((min_val, max_val, part))
    index.sort(key=lambda x: x[0])

    # Step 4: Query Intersected Partitions
    intersected = []
    for min_v, max_v, part in index:
        if min_v <= isovalue <= max_v:
            intersected.append(part)

    # Step 5: Find Seed Vertices and Tetrahedra
    seed_tets = []
    for part in intersected:
        # lower_bound: first element with value >= isovalue [S1, unofficial]
        seed_v = None
        for vid, val in part:
            if val >= isovalue:
                seed_v = vid
                break
        if seed_v is None:
            seed_v = part[0][0]

        # Find intersected tetrahedron containing seed_v [S1, unofficial]
        for tet in mesh.tetrahedra_containing_vertex(seed_v):
            if scalar_field[tet.min_vertex] <= isovalue <= scalar_field[tet.max_vertex]:
                seed_tets.append(tet)
                break

    # Step 6: Extract Contours
    all_triangles = []
    visited_tets = set()
    for tet in seed_tets:
        all_triangles.extend(ExtractContour(tet, mesh, scalar_field, isovalue, visited_tets))
    return all_triangles

def ExtractContour(seed_tet, mesh, scalar_field, isovalue, visited_tets):
    q = [seed_tet]
    triangles = []
    while q:
        tet = q.pop(0)
        if tet in visited_tets:
            continue
        visited_tets.add(tet)
        triangles.append(ComputePartialIntersection(tet, scalar_field, isovalue))
        for n in mesh.neighbors(tet):
            if n not in visited_tets:
                if scalar_field[n.min_vertex] <= isovalue <= scalar_field[n.max_vertex]:
                    q.append(n)
    return triangles

def is_connected(vertices, edges):
    if not vertices:
        return True
    start = vertices[0]
    visited = {start}
    q = [start]
    while q:
        curr = q.pop(0)
        for n in mesh.neighbors(curr):
            if n in vertices and n not in visited:
                visited.add(n)
                q.append(n)
    return len(visited) == len(vertices)

def ComputePartialIntersection(tet, scalar_field, isovalue):
    # Standard PL intersection computation [G]
    return tet.compute_intersection(isovalue)
```

**Parameters and Edge Cases**
*   **Isovalue:** Must be within the range of the scalar field [S1, unofficial].
*   **Precision:** Floating point comparisons for `scalar_field[v] >= isovalue` should use a small epsilon if necessary, though the source uses exact comparisons [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The extraction phase is optimal, visiting only intersected tetrahedra [S1, unofficial]. The pre-processing (partitioning) is $O(V+E)$ where $V$ is vertices and $E$ is edges, due to flood fill [S1, unofficial].
*   **Memory Complexity:** Requires storing partitions and the partition index [S1, unofficial]. The partition index is an interval table, typically $O(P)$ where $P$ is the number of partitions [S1, unofficial].
*   **Strengths:** Avoids visiting non-intersected tetrahedra, which is a significant speedup for sparse isosurfaces or high-resolution grids where most cells do not intersect the level set [S1, unofficial].
*   **Limitations:** Requires pre-processing the mesh to identify saddles and build partitions [S1, unofficial]. This adds overhead for single-frame extraction or dynamic scalar fields that change frequently [S1, unofficial].

## Background
This method originates from visualization exercises and educational materials on isosurface extraction, specifically associated with Julien Tierny [S1, unofficial]. It is presented as a simple to implement optimal algorithm compared to more complex adaptive methods, leveraging the topological properties of saddle points to decompose the mesh [S1, unofficial].

## Sources
[S1] Julien Tierny - Visualization Exercise - Isosurface Extraction, https://julien-tierny.github.io/visualizationExerciseIso.html, fetched 2026-10-30 (unofficial)
