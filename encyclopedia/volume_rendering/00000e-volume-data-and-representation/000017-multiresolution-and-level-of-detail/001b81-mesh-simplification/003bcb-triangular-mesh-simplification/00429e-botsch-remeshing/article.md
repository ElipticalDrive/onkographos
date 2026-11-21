# Botsch Remeshing

Also called: Botsch algorithm, Remeshing via Vertex Optimization

## Overview
Botsch Remeshing re-meshes a triangular surface to achieve a target edge length, improving mesh uniformity [S1, unofficial].

## How it works
**Inputs:** A triangular mesh with vertices $V$ and faces $F$, an arguments object containing `remesh_len`, and an optional voxel model with `inside_extent` [S1, unofficial].
**Outputs:** A new mesh with optimized vertices $V'$ and faces $F'$ [S1, unofficial].

**Data Structures:**
*   `mesh.vertices`: Array of shape $(N, 3)$ containing vertex coordinates.
*   `mesh.faces`: Array of shape $(M, 3)$ containing face indices.
*   `mesh.edges_unique_length`: Array of unique edge lengths.

**Steps:**
1. **Calculate Average Edge Length:** Compute the mean of the unique edge lengths.
   $$ L_{\text{avg}} = \text{mean}(\text{mesh.edges\_unique\_length}) $$
2. **Determine Target Edge Length ($h$):** If `args.remesh_len < 0`, set $h = \min(L_{\text{avg}}, E / 1024)$, where $E$ is the inside extent of the voxel model; otherwise, set $h = \text{args.remesh\_len}$ [S1, unofficial].
3. **Execute Remeshing:** Run the Botsch optimization for $i=5$ iterations with target length $h$: $V', F' = \text{remesh\_botsch}(V, F, i=5, h=h)$ [S1, unofficial].
4. **Reconstruct Mesh:** Create a new mesh object from the optimized vertices and faces [S1, unofficial].

**Pitfalls:**
*   If remeshing fails, the error must be caught and reported with `print(f"Remesh failed.")` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class Mesh:
    def __init__(self, vertices, faces):
        self.vertices = vertices
        self.faces = faces
        self.edges_unique_length = compute_unique_edge_lengths(vertices, faces)

def compute_unique_edge_lengths(vertices, faces):
    edges = set()
    for f in faces:
        for i in range(3):
            edges.add(tuple(sorted((vertices[f[i]], vertices[f[(i+1)%3]]))))
    return [np.linalg.norm(e[1] - e[0]) for e in edges]

def botsch_remesh(mesh, voxel_model=None, args=None):
    # Step 1: Calculate average edge length
    avg_edge_len = mesh.edges_unique_length.mean()

    # Step 2: Determine target edge length
    if args is not None and args.remesh_len < 0:
        extent = voxel_model.inside_extent.item() if hasattr(voxel_model.inside_extent, 'item') else float(voxel_model.inside_extent)
        target_edge_len = min(avg_edge_len, extent / 1024)
    else:
        target_edge_len = args.remesh_len if args is not None else avg_edge_len

    # Step 3: Execute remeshing
    try:
        v_new, f_new = remesh_botsch(mesh.vertices, mesh.faces, i=5, h=target_edge_len)
    except Exception:
        print(f"Remesh failed.")
        return None

    # Step 4: Reconstruct mesh
    new_mesh = Mesh(vertices=v_new, faces=f_new)
    return new_mesh

def remesh_botsch(vertices, faces, i, h):
    # [G] Sources do not specify the internal optimization steps.
    # Standard Botsch remeshing iteratively adjusts vertex positions
    # to minimize deviation from target edge length h, typically using
    # Laplacian smoothing and edge flips.
    return vertices, faces
```

## Subtopics

### Mesh Tessellation
A remeshing operation that increases vertex count to allow optimization to deal with surface details, applied on specific epochs [S2, unofficial]. It tessellates the mesh using each side's midpoint [S2, unofficial]. This is followed by a learning rate decay multiplying the previous learning rate by a factor of $8e^{-3}$ [S2, unofficial]. A pitfall is that tessellating the mesh always results in an increase in the loss because the optimizer had converged to the vertices' positions [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Measured Performance:** Not given in the sources.
**Strengths and Limitations:** Improves mesh uniformity [S1, unofficial]. Tessellation allows handling of surface details but increases loss temporarily [S2, unofficial].

## Background
The Botsch algorithm is a method for remeshing triangular surfaces [S1, unofficial]. It is used in contexts such as volume rasterization and differentiable rendering to prepare meshes for optimization [S1, unofficial][S2, unofficial].

## Sources
[S1] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S2] MRD: Using Physically Based Differentiable Rendering to Probe Vision Models for 3D Scene Understanding, https://doi.org/10.48550/arXiv.2512.12307, fetched 2026-10-29 (unofficial)
