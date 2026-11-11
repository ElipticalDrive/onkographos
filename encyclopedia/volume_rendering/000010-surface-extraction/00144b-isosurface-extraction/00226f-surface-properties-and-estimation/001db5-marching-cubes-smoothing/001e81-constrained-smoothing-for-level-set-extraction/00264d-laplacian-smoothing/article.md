# Laplacian Smoothing
Also called: Laplacian Relaxation, Laplacian Smooth
## Overview
Laplacian Smoothing is a mesh smoothing technique applied to vertex positions to reduce noise or jaggedness [S1, unofficial]. It computes the average position of each vertex's immediate neighbors in the mesh graph and updates the vertex position to reduce jaggedness (stairstepping) while preserving overall shape [S2, unofficial].
## How it works
**Inputs:** A triangular mesh defined by a list of vertex positions and triangle indices [G].
**Outputs:** A modified mesh with updated vertex positions [G].

**Data Structures:**
- `vertices`: Array of vertex positions, where each vertex $i$ has a position vector $\mathbf{p}_i$.
- `neighbors`: Adjacency list, a vector of vectors where `neighbors[i]` contains the unique indices of vertices connected to vertex $i$ by an edge [S2, unofficial].

**Steps:**
1. **Build Adjacency List:**
   - Initialize `neighbors` as an empty list of lists for all vertices.
   - Iterate through the triangle index list in chunks of three indices $(a, b, c)$.
   - For each triangle, add bidirectional edges between the three vertices to `neighbors`, ensuring no duplicate neighbors are added [S2, unofficial].
2. **Iterative Smoothing:**
   - For each iteration, store current positions in `old_pos`.
   - For each vertex $i$, if `neighbors[i]` is empty, skip it.
   - Calculate the average position of all neighbors and update the vertex position [S2, unofficial].

**Formulas:**
The average position is computed as $\mathbf{avg} = \frac{1}{|N_i|} \sum_{j \in N_i} \mathbf{p}_j^{old}$, where $N_i$ represents the neighbor indices for vertex $i$ and $|N_i|$ is the neighbor count [S2, unofficial]. The updated position is $\mathbf{p}_i^{new} = \mathbf{avg}$ for the standard method.

**Parameters:**
- `smoothIter`: Number of smoothing iterations, default 0 [S1, unofficial].
- Additional parameters passed to the smoothing function include 0.1, 0.5, `smoothIter`, and a boolean flag [S1, unofficial].

**Pitfalls:**
- **Empty Neighbors:** If a vertex has no neighbors (isolated), the loop skips it to prevent division by zero [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x, self.y, self.z = x, y, z
    def __add__(self, other):
        return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __truediv__(self, scalar):
        return Vec3(self.x / scalar, self.y / scalar, self.z / scalar)

def laplacian_smooth(vertices, triangles, smooth_iter=0):
    n = len(vertices)
    neighbors = [[] for _ in range(n)]
    for a, b, c in triangles:
        if b not in neighbors[a]: neighbors[a].append(b)
        if a not in neighbors[b]: neighbors[b].append(a)
        if c not in neighbors[b]: neighbors[b].append(c)
        if b not in neighbors[c]: neighbors[c].append(b)
        if a not in neighbors[c]: neighbors[c].append(c)
        if c not in neighbors[a]: neighbors[a].append(c)
    current_pos = [Vec3(*v) if isinstance(v, (list, tuple)) else v for v in vertices]
    for _ in range(smooth_iter):
        old_pos = [Vec3(*v) for v in current_pos]
        for i in range(n):
            if not neighbors[i]:
                continue
            avg = Vec3()
            count = len(neighbors[i])
            for j in neighbors[i]:
                avg = avg + old_pos[j]
            avg = avg / count
            current_pos[i] = avg
    return current_pos
```
## Subtopics

### Conservative Mesh Smoothing
A variant that blends the original vertex position with the Laplacian average to preserve volume and shape better than pure averaging [S2, unofficial]. It uses a weighted blend of 60% of the original position and 40% of the neighbor average [S2, unofficial]. This reduces the shrinking effect of standard Laplacian smoothing while still reducing jaggedness [S2, unofficial].

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
[S2] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
