# Botsch Remeshing

Also called: Botsch algorithm, Remeshing via Vertex Optimization

## Overview
Botsch Remeshing re-meshes a triangular surface to achieve a target edge length, thereby improving mesh uniformity [S1, unofficial]. It differs from standard simplification methods by optimizing vertex positions to fit a desired density rather than merely reducing polygon count. It builds on [Triangular Mesh Simplification](9103-triangular-mesh-simplification.md).

## How it works
**Inputs:** A triangular mesh (vertices $V$, faces $F$), a target edge length $h$, and an iteration count $i$.
**Outputs:** A new mesh with vertices $V'$ and faces $F'$ that approximate the original surface with uniform edge lengths near $h$.

**Data Structures:**
*   `mesh.vertices`: Array of shape $(N, 3)$ containing vertex coordinates.
*   `mesh.faces`: Array of shape $(M, 3)$ containing face indices.
*   `mesh.edges_unique_length`: Array of unique edge lengths.

**Steps:**
1.  **Calculate Average Edge Length:** Compute the mean of the unique edge lengths of the input mesh.
    $$ L_{avg} = \text{mean}(\text{mesh.edges\_unique\_length}) $$
2.  **Determine Target Edge Length ($h$):**
    *   If a specific target length is not provided (or is negative), calculate the voxel resolution limit. Let $E$ be the inside extent of the voxel model.
        $$ h = \min(L_{avg}, E / 1024) $$
    *   Otherwise, use the provided target length.
3.  **Execute Remeshing:** Run the Botsch optimization algorithm for $i=5$ iterations with target length $h$. This step optimizes vertex positions to satisfy the target edge length constraint while preserving the surface geometry.
    $$ V', F' = \text{remesh\_botsch}(V, F, i=5, h=h) $$
4.  **Reconstruct Mesh:** Create a new mesh object from the optimized vertices and faces.
    $$ \text{mesh}_{new} = \text{Trimesh}(V', F') $$

**Pitfalls:**
*   If the remeshing process fails (e.g., due to degenerate geometry or numerical instability), the operation should be caught and reported [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def botsch_remesh(mesh, voxel_model=None, args=None):
    """
    Remesh a triangular surface to achieve a target edge length.
    
    Args:
        mesh: Input mesh object with .vertices, .faces, .edges_unique_length attributes.
        voxel_model: Optional voxel model object with .inside_extent attribute.
        args: Optional arguments object with .remesh_len attribute.
        
    Returns:
        New mesh object or None if remeshing fails.
    """
    # Step 1: Calculate average edge length
    avg_edge_len = mesh.edges_unique_length.mean()
    
    # Step 2: Determine target edge length
    if args is not None and args.remesh_len < 0:
        # Use minimum of average edge length and voxel resolution limit
        # Note: inside_extent is assumed to be a scalar or tensor convertible to float
        extent = voxel_model.inside_extent.item() if hasattr(voxel_model.inside_extent, 'item') else voxel_model.inside_extent
        target_edge_len = min(avg_edge_len, extent / 1024)
    else:
        # Use provided target length
        target_edge_len = args.remesh_len if args is not None else avg_edge_len
        
    # Step 3: Execute remeshing
    # The core Botsch algorithm is encapsulated in remesh_botsch
    # It performs 5 iterations of vertex optimization to match target edge length h
    try:
        v_new, f_new = remesh_botsch(mesh.vertices, mesh.faces, i=5, h=target_edge_len)
    except Exception:
        print("Remesh failed.")
        return None
        
    # Step 4: Reconstruct mesh
    new_mesh = Trimesh(vertices=v_new, faces=f_new)
    return new_mesh

def remesh_botsch(vertices, faces, i, h):
    """
    Core Botsch remeshing algorithm.
    Optimizes vertex positions to achieve uniform edge length h over i iterations.
    """
    # Implementation details of the Botsch algorithm (vertex optimization)
    # are not fully specified in the source notes, but the call signature is:
    # remesh_botsch(mesh.vertices, mesh.faces, i=5, h=target_edge_len)
    # [G] Placeholder for the actual optimization loop.
    # In a full implementation, this would involve:
    # 1. Computing edge vectors and lengths.
    # 2. Adjusting vertex positions to minimize deviation from target length h.
    # 3. Re-triangulating if necessary (e.g., Laplacian smoothing or edge flips).
    # 4. Repeating for i iterations.
    
    # For the purpose of this article, we assume the function exists and returns
    # optimized vertices and faces.
    return vertices, faces
```

## Subtopics

### Mesh Tessellation
A remeshing operation that increases vertex count to allow optimization to deal with surface details, applied on specific epochs [S2, unofficial]. It tessellates the mesh using each side's midpoint [S2, unofficial]. This is followed by a learning rate decay multiplying the previous learning rate by a factor of $8e^{-3}$ [S2, unofficial]. A pitfall is that tessellating the mesh always results in an increase in the loss because the optimizer had converged to the vertices' positions [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Measured Performance:** Not given in the sources.
**Strengths and Limitations:** Improves mesh uniformity [S1, unofficial]. Tessellation allows handling of surface details but increases loss temporarily [S2, unofficial].
**When to Use:** When a uniform edge length is required for downstream processing or rendering [S1, unofficial].
**When Not to Use:** When preserving exact vertex positions is critical without subsequent optimization [S2, unofficial].

## Background
The Botsch algorithm is a method for remeshing triangular surfaces. It is used in contexts such as volume rasterization and differentiable rendering to prepare meshes for optimization [S1, unofficial][S2, unofficial].

## Sources
[S1] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S2] MRD: Using Physically Based Differentiable Rendering to Probe Vision Models for 3D Scene Understanding, https://doi.org/10.48550/arXiv.2512.12307, fetched 2026-10-29 (unofficial)
