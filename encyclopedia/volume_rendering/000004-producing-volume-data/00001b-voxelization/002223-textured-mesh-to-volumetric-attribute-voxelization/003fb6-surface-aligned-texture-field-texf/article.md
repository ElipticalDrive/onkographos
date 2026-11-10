# Surface-aligned Texture Field (TexF)


## Overview
Surface-aligned Texture Field (TexF) is a UV-free texture representation that associates material attributes with voxels derived from a mesh surface, preserving spatial locality without requiring UV parameterization [S1, unofficial]. It solves the problem of transferring 2D texture data to 3D geometry by embedding attributes directly in a surface-aligned 3D field, allowing direct queries at mesh surface positions during rendering [S1, unofficial]. Unlike UV-based pipelines that rely on chart-based unfolding and UV coordinates, TexF enables direct 3D spatial queries, reducing transfer and discretization errors through differentiable rendering refinement [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A textured mesh $M$ with geometry and source material maps (e.g., base color, roughness, metallicity).
*   **Output:** A sparse voxel field $T$ where each active voxel $q \in Q$ stores texture attributes $t(q)$, typically RGB color by default, extendable to other PBR attributes [S1, unofficial].

**Data Structures:**
*   **Canonical Grid:** A $K \times K \times K$ axis-aligned bounding box (AABB) discretization shared between the mesh geometry and the TexF [S1, unofficial].
*   **Active Set $Q$:** A sparse set of voxels intersecting the mesh surface. The grid is not allocated densely; only active voxels are stored [S1, unofficial].
*   **Voxel Attribute:** $t(q)$, a vector of material properties.

**Parameters:**
*   $K$: The resolution of the canonical AABB grid. Typical values are not specified in the sources [S1, unofficial].

**Construction Steps:**
1.  **Discretization:** Define the canonical AABB of mesh $M$ and discretize it into a $K \times K \times K$ grid [S1, unofficial].
2.  **Active Voxel Identification:** Identify voxels intersecting the mesh surface to form the active set $Q$. This is done via surface sampling and voxelization without allocating the dense $K^3$ grid [S1, unofficial].
3.  **UV Texture Transfer Initialization:**
    *   Sample points on the mesh surface. For each sample, compute its 3D position $\mathbf{p}$ and UV coordinate $\mathbf{u}$ using barycentric coordinates [S1, unofficial].
    *   Sample the source material maps using $\mathbf{u}$ to retrieve attribute values [S1, unofficial].
    *   Assign the retrieved attributes to the voxel corresponding to $\mathbf{p}$ in the grid [S1, unofficial].
    *   If multiple samples map to the same voxel, average their attributes [S1, unofficial].
4.  **Differentiable Rendering Refinement (Inverse Rendering):**
    *   Render the mesh using the initialized TexF via a differentiable renderer [S1, unofficial].
    *   Use multi-view image-space supervision to refine voxel attributes $t(q)$ and reduce transfer/discretization errors introduced by voxelization and interpolation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_texf(mesh, source_maps, K):
    """
    Construct a Surface-aligned Texture Field from a textured mesh.
    
    Args:
        mesh: Triangular mesh with geometry and UV coordinates.
        source_maps: Dictionary of 2D texture maps (e.g., 'color', 'roughness').
        K: Resolution of the canonical AABB grid.
        
    Returns:
        texf: Sparse dictionary mapping voxel indices to attribute vectors.
    """
    # 1. Define Canonical AABB and Grid
    bbox_min, bbox_max = mesh.compute_aabb()
    voxel_size = (bbox_max - bbox_min) / K
    
    # 2. Initialize Sparse Active Set
    # Q is a set of voxel indices (i, j, k)
    Q = set()
    # texf stores attributes for active voxels
    texf = {} 
    
    # 3. Surface Sampling and UV Transfer
    # Sample points on the mesh surface
    surface_samples = mesh.sample_surface_points(num_samples=10000)
    
    for sample in surface_samples:
        # Get 3D position and UV via barycentric coordinates
        p_3d = sample.position
        u_uv = sample.uv_coordinate
        
        # Sample source maps using UV
        # Assume source_maps['color'] is a function sample_color(u, v)
        attr = {}
        for map_name, map_data in source_maps.items():
            attr[map_name] = map_data.sample(u_uv)
            
        # Map 3D position to voxel index
        voxel_idx = world_to_voxel(p_3d, bbox_min, voxel_size)
        
        # Add to active set
        Q.add(voxel_idx)
        
        # Average attributes for the same voxel
        if voxel_idx not in texf:
            texf[voxel_idx] = {k: 0.0 for k in attr}
            texf[voxel_idx]['count'] = 0
            
        for k, v in attr.items():
            texf[voxel_idx][k] += v
        texf[voxel_idx]['count'] += 1
        
    # Normalize averages
    for voxel_idx in texf:
        count = texf[voxel_idx]['count']
        for k in texf[voxel_idx]:
            if k != 'count':
                texf[voxel_idx][k] /= count
        del texf[voxel_idx]['count']
        
    # 4. Refinement via Differentiable Rendering
    # This step requires a differentiable renderer and multi-view images
    # The refinement loop is not detailed in the construction phase notes
    # [G] In practice, this involves optimizing t(q) to minimize 
    # reconstruction error against reference images.
    
    return texf, Q

def query_texf(texf, Q, p_3d, bbox_min, voxel_size):
    """
    Query texture attributes at a 3D surface position.
    """
    voxel_idx = world_to_voxel(p_3d, bbox_min, voxel_size)
    if voxel_idx in Q:
        return texf[voxel_idx]
    else:
        # [G] Fallback or interpolation logic would be needed here
        # for positions not exactly on a voxel center.
        return None
```

**Edge Cases and Pitfalls:**
*   **Transfer Errors:** Voxel discretization and interpolation during UV texture transfer introduce errors [S1, unofficial]. These are mitigated by the inverse-rendering refinement step [S1, unofficial].
*   **Coordinate System:** TexF and mesh geometry share a common coordinate system (the canonical AABB), which is critical for direct querying [S1, unofficial].

## Subtopics

### [Bitstream-oriented TexF Compression](10086-bitstream-oriented-texf-compression.md)
Encodes voxel attributes directly using established 3D attribute codecs (Unicorn or G-PCC), exploiting spatial correlations in native 3D neighborhoods. Voxel locations are reconstructed from the decoded mesh, avoiding separate voxel-position coding.

### [Differentiable TexF Rendering](10095-differentiable-texf-rendering.md)
Uses a differentiable renderer to enable inverse-rendering (IR) refinement, using multi-view image-space supervision to reduce transfer and discretization errors.

## Operating characteristics
*   **Time Complexity:** Construction involves surface sampling and voxelization, which is efficient due to sparse support. Refinement via differentiable rendering adds computational cost proportional to the number of views and iterations.
*   **Memory Complexity:** Sparse voxel support avoids allocating the dense $K^3$ grid. Memory usage depends on the number of active voxels $|Q|$ and the size of attributes per voxel.
*   **Performance:** Experiments on MPEG and AOM mesh compression benchmarks demonstrate improved average rate-distortion performance over representative UV-based methods for both bitstream and GPU-resident compression [S1, unofficial].
*   **Strengths:** Direct 3D queries without UV parameterization; compatibility with standard 3D attribute codecs; support for real-time rendering via GPU-resident compression (3DNTC) [S1, unofficial].
*   **Limitations:** Requires differentiable rendering for optimal fidelity; discretization errors if refinement is skipped [S1, unofficial].
*   **When to Use:** When UV parameterization is difficult or undesirable; when direct 3D spatial queries are needed; for compression tasks leveraging 3D spatial correlations.
*   **When Not to Use:** When strict compatibility with legacy UV-based pipelines is required without baking; when computational resources for differentiable refinement are unavailable.

## Background
TexF was introduced in the paper "Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields" [S1, unofficial]. It represents a shift from UV-based paradigms, which dominate production pipelines, to UV-free approaches that embed texture attributes directly in a 3D field. The method supports both bitstream compression (using codecs like Unicorn and G-PCC) and GPU-resident compression (using a neural codec called 3DNTC) [S1, unofficial].

## Sources
[S1] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
