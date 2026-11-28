# Adaptive Merge Grid Construction
Also called: Merge Grid Construction

## Overview
Adaptive Merge Grid Construction is a divide-and-conquer strategy that subdivides a spatial domain using an adaptive octree to localize Constructive Solid Geometry (CSG) operations [S1]. It solves the problem of complex patch relationships in implicit surface representations by ensuring each grid cell contains the simplest possible patch interaction, thereby improving robustness over global approaches [S1]. The subdivision criteria prioritize topological simplicity, specifically checking whether the adjacency graph of patches in a cell forms a clique [S1].

## How it works
**Inputs:** Surface shape $S$ (implicit function or patch set) [S1]. Initial merge grid $G$ (initialized as an octree grid) [S1]. Maximal depth $d_{max}$ (integer constraint for computational practicality) [S1].

**Outputs:** Refined merge grid $G$ where leaf cells contain simplified patch relationships [S1].

**Data Structures:** Cell $C$ represents a node in the octree with fields $C.d$ (current depth), $C.sub$ (boolean subdivision flag), and $S_C$ (subset of patches in $C$) [S1]. Adjacency graph $g$ is a graph where nodes are patches in $S_C$ and edges represent adjacency or intersection relationships between patches [S1].

**Algorithm Steps:**
1. Initialize $G$ as an octree grid [S1].
2. Iterate while at least one leaf cell $C$ in $G$ has $C.sub$ set to True [S1].
3. For each cell $C$ in $G$: skip if $C.sub$ is False [S1].
4. If $S_C$ is not empty and $C.d < d_{max}$: identify all connected components (graphs $g$) in the adjacency graph of $C$ [S1].
5. For each connected graph $g$: check if $g$ is a clique graph [S1].
6. If $g$ is not a clique: subdivide $C$ into children $M$, set $C'.sub \leftarrow \text{True}$ for all $C' \in M$, and break the inner loop [S1].
7. If all connected graphs are cliques (or subdivision was skipped): set $C.sub \leftarrow \text{False}$ [S1].
8. Else (if $S_C$ is empty or $C.d \ge d_{max}$): set $C.sub \leftarrow \text{False}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_merge_grid(surface_patches, grid, d_max):
    """
    surface_patches: list of patch objects
    grid: list of Cell objects representing the octree
    d_max: integer, maximum subdivision depth
    """
    def is_clique(nodes, edges):
        if len(nodes) <= 2:
            return True
        for i in range(len(nodes)):
            for j in range(i + 1, len(nodes)):
                if (nodes[i], nodes[j]) not in edges and (nodes[j], nodes[i]) not in edges:
                    return False
        return True

    def get_connected_components(nodes, edges):
        visited = set()
        components = []
        for node in nodes:
            if node not in visited:
                comp_nodes = []
                comp_edges = []
                queue = [node]
                visited.add(node)
                while queue:
                    curr = queue.pop(0)
                    comp_nodes.append(curr)
                    for neighbor in [n for (u, n) in edges if u == curr] + \
                                    [n for (n, u) in edges if u == curr]:
                        if neighbor not in visited:
                            visited.add(neighbor)
                            queue.append(neighbor)
                            comp_edges.append((curr, neighbor))
                components.append((comp_nodes, comp_edges))
        return components

    def build_adjacency_graph(patches):
        nodes = list(patches)
        edges = []
        for i in range(len(nodes)):
            for j in range(i + 1, len(nodes)):
                if nodes[i].intersects(nodes[j]):
                    edges.append((nodes[i], nodes[j]))
        return nodes, edges

    def subdivide(cell):
        children = []
        for _ in range(8):
            child = Cell()
            child.parent = cell
            child.d = cell.d + 1
            child.sub = True
            child.patches = [p for p in cell.patches if p.in_bounds(child.bounds)]
            children.append(child)
        return children

    class Cell:
        def __init__(self):
            self.d = 0
            self.sub = True
            self.patches = []
            self.bounds = None
            self.parent = None

    while any(c.sub for c in grid):
        for i in range(len(grid)):
            C = grid[i]
            if not C.sub:
                continue
            if len(C.patches) > 0 and C.d < d_max:
                nodes, edges = build_adjacency_graph(C.patches)
                components = get_connected_components(nodes, edges)
                subdivided = False
                for comp_nodes, comp_edges in components:
                    if not is_clique(comp_nodes, comp_edges):
                        children = subdivide(C)
                        for child in children:
                            child.sub = True
                        grid[i] = children
                        subdivided = True
                        break
                if not subdivided:
                    C.sub = False
            else:
                C.sub = False
    return grid
```

**Edge Cases and Pitfalls:**
*   **Disjoint Patches / Self-Intersections:** A leaf cell may contain multiple disconnected components in its adjacency graph, which occurs for disjoint patches with close proximity or self-intersecting shapes [S1].
*   **Handling Disconnected Components:** The algorithm creates $Q$ copies of the cell, where $Q$ is the number of connected components, with each copy representing one connected component [S1]. Merge loss computation is assigned to these duplicated cells based on their corresponding patch identity [S1].

## Operating characteristics
*   **Complexity:** The complexity depends on the number of patches and the depth of the octree [S1]. Subdivision reduces the complexity of the adjacency graph in each cell by isolating non-clique interactions [S1].
*   **Strengths:** Improves robustness of CSG operations by localizing them to cells with simple (clique) patch relationships [S1].
*   **Limitations:** At $d_{max}$, some cells may still contain non-clique graphs, though the complexity is significantly reduced compared to the global case [S1].
*   **Performance:** Empirically shown to be efficient, though specific benchmarks, hardware, resolution, and scene details are not given in the sources.

## Background
The method is introduced in the context of "Patch-Grid," an efficient and feature-preserving neural implicit surface representation [S1]. It addresses the challenge of handling sharp geometric features (edges, corners) in implicit surfaces by simplifying the local patch interactions [S1].

## Sources
[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
