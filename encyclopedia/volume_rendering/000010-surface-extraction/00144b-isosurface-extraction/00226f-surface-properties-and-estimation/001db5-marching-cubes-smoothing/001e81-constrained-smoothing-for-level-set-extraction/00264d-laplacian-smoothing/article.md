# Laplacian Smoothing
Also called: Laplacian Relaxation, Laplacian Smooth
## Overview
Laplacian Smoothing is a mesh smoothing technique applied to vertex positions to reduce noise or jaggedness, specifically addressing the stairstepping artifacts common in isosurface extraction [S1, S2, unofficial]. It moves each vertex towards the average position of its immediate neighbors in the mesh graph [S2, unofficial]. Unlike constrained methods that preserve specific features or boundaries, this basic form prioritizes geometric regularity, often at the cost of slight volume loss or feature blurring [S1, unofficial].
## How it works
**Inputs:** A triangular mesh defined by a list of vertices and triangle indices.
**Outputs:** A modified mesh with updated vertex positions.

**Data Structures:**
- `vertices`: Array of vertex positions, where each vertex $i$ has a position vector $\mathbf{p}_i$.
- `neighbors`: Adjacency list, typically a vector of vectors (`Vec<Vec<usize>>`), where `neighbors[i]` contains the unique indices of vertices connected to vertex $i$ by an edge.

**Steps:**
1. **Build Adjacency List:**
   - Initialize `neighbors` as an empty list of lists for all vertices.
   - Iterate through the triangle index list in chunks of three indices $(a, b, c)$.
   - For each triangle, add bidirectional edges: add $b$ to `neighbors[a]`, $a$ to `neighbors[b]`, $c$ to `neighbors[b]$, $b$ to `neighbors[c]`, $a$ to `neighbors[c]`, and $c$ to `neighbors[a]`.
   - Ensure no duplicate neighbors are added to each vertex's list [S2, unofficial].

2. **Iterative Smoothing:**
   - For each iteration $k$ from $1$ to $N_{iter}$:
     - Store current positions in `old_pos` to prevent cascading updates within a single iteration [S2, unofficial].
     - For each vertex $i$:
       - If `neighbors[i]` is empty, skip vertex $i$ [S2, unofficial].
       - Calculate the average position $\mathbf{avg}$ of all neighbors:
         $$ \mathbf{avg} = \frac{1}{|N_i|} \sum_{j \in N_i} \mathbf{p}_j^{old} $$
         where $N_i$ is the set of neighbor indices for vertex $i$, and $|N_i|$ is the count of neighbors [S2, unofficial].
       - Update the vertex position. In the standard Laplacian Smooth, the new position is often set directly to the average or a weighted blend [S1, S2, unofficial].

**Parameters:**
- `smoothIter`: Number of smoothing iterations. Default is 0 [S1, unofficial].
- Weights: In some implementations, parameters such as 0.1 and 0.5 are passed, potentially representing damping or boundary handling flags [S1, unofficial].

**Pitfalls:**
- **Empty Neighbors:** If a vertex has no neighbors (isolated vertex), the loop must skip it to prevent division by zero [S2, unofficial].
- **Volume Loss:** Repeated application can cause the mesh to shrink or collapse if not constrained [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def laplacian_smooth(vertices, triangles, smooth_iter=0):
    """
    vertices: List of Vec3 positions.
    triangles: List of tuples (a, b, c) of vertex indices.
    smooth_iter: Number of iterations.
    Returns: Updated list of Vec3 positions.
    """
    n = len(vertices)
    neighbors = [[] for _ in range(n)]
    
    # Build adjacency list
    for a, b, c in triangles:
        # Add bidirectional edges, ensuring uniqueness
        if b not in neighbors[a]: neighbors[a].append(b)
        if a not in neighbors[b]: neighbors[b].append(a)
        if c not in neighbors[b]: neighbors[b].append(c)
        if b not in neighbors[c]: neighbors[c].append(b)
        if a not in neighbors[c]: neighbors[c].append(c)
        if c not in neighbors[a]: neighbors[a].append(c)
        
    current_pos = [v for v in vertices]
    
    for _ in range(smooth_iter):
        old_pos = [v for v in current_pos]
        for i in range(n):
            if not neighbors[i]:
                continue
            avg = Vec3(0, 0, 0)
            count = len(neighbors[i])
            for j in neighbors[i]:
                avg = avg + old_pos[j]
            avg = avg / count
            # Standard Laplacian: replace with average
            current_pos[i] = avg
            
    return current_pos
```
## Subtopics

### Conservative Mesh Smoothing
A variant that blends the original vertex position with the Laplacian average to preserve volume and shape better than pure averaging. It uses a weighted blend: 60% of the original position and 40% of the neighbor average [S2, unofficial]. This reduces the shrinking effect of standard Laplacian smoothing while still reducing jaggedness [S2, unofficial]. Choose this when preserving the overall volume of the extracted surface is critical [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(I \cdot E)$, where $I$ is the number of iterations and $E$ is the number of edges (or $O(I \cdot V)$ if adjacency is precomputed and dense). Building the adjacency list is $O(F)$ where $F$ is the number of faces [S2, unofficial].
- **Memory Complexity:** $O(V + E)$ to store the adjacency list [S2, unofficial].
- **Strengths:** Simple to implement; effective at removing high-frequency noise and stairstepping artifacts from voxel-based isosurfaces [S1, S2, unofficial].
- **Limitations:** Can cause mesh shrinkage and loss of fine details if iterations are too high; does not preserve sharp features or boundaries unless constrained [S1, S2, unofficial].
- **When to use:** Post-processing noisy meshes from binary volume extraction where shape fidelity is less critical than visual smoothness [S1, unofficial].
- **When not to use:** When precise volume preservation or feature preservation is required; use Constrained Smoothing instead [S1, unofficial].
## Background
Laplacian smoothing is a fundamental technique in mesh processing. In the context of volume rendering, it is often applied after surface extraction (e.g., Marching Cubes) to clean up the resulting mesh [S1, S2, unofficial]. The specific implementation details cited here are derived from unofficial sources discussing meshing in game engines and voxel processing [S1, S2, unofficial].
## Sources
[S1] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
[S2] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
