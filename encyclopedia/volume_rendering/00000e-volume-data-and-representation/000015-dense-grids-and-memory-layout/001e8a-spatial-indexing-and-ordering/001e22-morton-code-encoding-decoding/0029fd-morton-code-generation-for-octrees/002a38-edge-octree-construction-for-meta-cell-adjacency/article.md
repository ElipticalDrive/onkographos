# Edge-Octree Construction for Meta-Cell Adjacency

Also called: Hierarchical edge-octree construction, edge-octree

## Overview
This method constructs a hierarchical adjacency structure to resolve topological disconnections between cells in an adaptive octree that belong to different meta-cells [S1, unofficial]. It specifically addresses edges shared by multiple meta-cells by modifying spatial keys to embed meta-cell identity, thereby creating a unified hierarchy for boundary processing [S1, unofficial]. Unlike its parent method, [Morton Code Generation for Octrees](4673-morton-code-generation-for-octrees.md), which provides unique spatial keys, this method extends those keys to encode inter-meta-cell relationships [S1, unofficial]. It differs from its child, [Meta-Cell Edge Octree Connection](4758-meta-cell-edge-octree-connection.md), by focusing on the structural construction and key modification of the adjacency graph rather than the final mesh connectivity resolution [S1, unofficial].

## How it works
**Inputs:** A set of meta-cells containing octree cells, their Morton codes, and their local indices $(i, j, k)$ within the global meta-grid [S1, unofficial].
**Outputs:** Modified Morton codes for cells at inter-meta-cell boundaries, forming an augmented hierarchy where cells sharing a meta-edge are connected via an extra depth level [S1, unofficial].
**Data Structures:** Meta-cell indices $(i, j, k)$, original Morton codes, and new Edge-Octree Keys formed by concatenating a 3-bit meta-cell identifier with the original Morton code [S1, unofficial].

**Steps:**

1. **Identify Meta-Edges:** Locate meta-edges ($E_m$) parallel to a coordinate axis (Y or Z) shared by four distinct meta-cells [S1, unofficial]. For a Y-parallel edge, the meta-cells are indexed as $(i, j, k)$, $(i+1, j, k)$, $(i, j, k+1)$, and $(i+1, j, k+1)$ [S1, unofficial].
2. **Extract Boundary Cells:** Identify the set of octree cells belonging to the meta-faces containing $E_m$ [S1, unofficial]. These cells are spatially adjacent but hierarchically disconnected in the standard structure [S1, unofficial].
3. **Assign Meta-Cell Identifiers:** Assign a 3-bit code to each of the four meta-cells sharing the edge based on their relative position and the edge orientation, following the adjacency rules in Table 4.1 [S1, unofficial].
4. **Construct Edge-Octree Keys:** Modify the Morton code of each extracted cell by concatenating its assigned 3-bit meta-cell identifier with its original Morton code [S1, unofficial]. The new key is calculated as:
   $$ \text{Morton}_{\text{new}} = (\text{Morton}_{\text{meta\_cell}} \ll 24) \mid \text{Morton}_{\text{original}} $$
   where $\text{Morton}_{\text{meta\_cell}}$ is the 3-bit code and the shift amount (24 bits) places the identifier as the most significant bits, effectively adding an extra hierarchy level [S1, unofficial].
5. **Surface Extraction:** Apply the standard surface extraction algorithm to the modified edge-octree keys to generate remaining polygons and close the polygonal mesh [S1, unofficial].

**Table 4.1: Adjacency Codes for Meta-Cells**

| Edge Orientation | Neighbor Meta-Cell Relative to Reference $(i,j,k)$ | 3-Bit Code |
| :--- | :--- | :--- |
| Parallel to Y | $(i, j, k)$ | `000` |
| Parallel to Y | $(i+1, j, k)$ | `001` |
| Parallel to Y | $(i, j, k+1)$ | `100` |
| Parallel to Y | $(i+1, j, k+1)$ | `101` |
| Parallel to Z | $(i, j, k)$ | `000` |
| Parallel to Z | $(i+1, j, k)$ | `001` |
| Parallel to Z | $(i, j+1, k)$ | `010` |
| Parallel to Z | $(i+1, j+1, k)$ | `011` |

*Note: The source lists duplicate entries for neighbor $(i+1, j, k)$ with codes `000` and `001` for both axes, which is a typographical error [S1, unofficial]. The table above applies a logical correction [G] to assign unique codes based on binary counting of the offset indices, ensuring distinct neighbors receive distinct identifiers.*

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_edge_octree(meta_cells, meta_edges):
    """
    Constructs edge-octree keys for cells at meta-cell boundaries.
    Args:
        meta_cells: Dict mapping meta-cell index (i,j,k) to list of cell dicts.
                    Each cell dict has 'morton_code'.
        meta_edges: List of dicts with 'axis' ('Y' or 'Z'), 'i', 'j', 'k' (reference corner).
    Returns:
        Updated meta_cells with modified Morton codes for boundary cells.
    """
    for edge in meta_edges:
        ref_i, ref_j, ref_k = edge['i'], edge['j'], edge['k']
        axis = edge['axis']
        
        # Define the 4 meta-cells involved in the edge
        if axis == 'Y':
            indices = [(ref_i, ref_j, ref_k), (ref_i + 1, ref_j, ref_k),
                       (ref_i, ref_j, ref_k + 1), (ref_i + 1, ref_j, ref_k + 1)]
        else: # Z
            indices = [(ref_i, ref_j, ref_k), (ref_i + 1, ref_j, ref_k),
                       (ref_i, ref_j + 1, ref_k), (ref_i + 1, ref_j + 1, ref_k)]
        
        for m_idx in indices:
            mi, mj, mk = m_idx
            dx = mi - ref_i
            if axis == 'Y':
                dz = mk - ref_k
                meta_code = (dx << 1) | dz  # Logical correction for unique 3-bit codes [G]
            else:
                dy = mj - ref_j
                meta_code = (dx << 1) | dy  # Logical correction for unique 3-bit codes [G]
            
            # Process cells in this meta-cell that lie on the boundary faces
            if m_idx in meta_cells:
                for cell in meta_cells[m_idx]:
                    original_morton = cell['morton_code']
                    new_morton = (meta_code << 24) | original_morton
                    cell['morton_code'] = new_morton
                    cell['edge_octree_key'] = new_morton
    return meta_cells
```

**Edge Cases and Pitfalls:**
- **Source Ambiguity:** Table 4.1 contains duplicate entries for neighbor $(i+1, j, k)$ with codes `000` and `001` [S1, unofficial]. The implementation uses a logical correction [G] mapping offsets $(dx, dz)$ or $(dx, dy)$ to unique 3-bit codes to prevent identifier collisions.
- **Bit Width:** The shift amount of 24 bits assumes the original Morton code fits within 24 bits [S1, unofficial]. Deeper hierarchies or larger coordinates require adjusting the shift to prevent overflow [G].
- **Face Identification:** The method requires extracting only cells belonging to the specific meta-faces containing the meta-edge [S1, unofficial]. Spatial filtering based on cell coordinates relative to meta-cell boundaries is required but not detailed in the source [G].

## Subtopics

### [Meta-Cell Edge Octree Connection](4758-meta-cell-edge-octree-connection.md)
Connects mesh components from individual meta-cells by processing boundary edges skipped during initial extraction. It uses the edge-octree structure to resolve hierarchical disconnections. Choose this when the goal is final mesh connectivity rather than key construction.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of cells on the meta-cell boundaries. Key modification is a constant-time bitwise operation per cell [S1, unofficial].
- **Memory Complexity:** No additional memory structures are created; the hierarchy is encoded within the modified Morton codes [S1, unofficial].
- **Strengths:** Resolves topological gaps between meta-cells without global communication or complex graph traversal during extraction. Integrates seamlessly with existing Morton-based surface extraction algorithms [S1, unofficial].
- **Limitations:** Relies on the correctness of the meta-cell adjacency table. Performance depends on the efficiency of identifying boundary cells [S1, unofficial].
- **When to Use:** When rendering massive volumes using meta-cell based adaptive octrees where inter-meta-cell boundaries cause mesh holes [S1, unofficial].
- **When Not to Use:** For uniform grids or small volumes where meta-cell decomposition is unnecessary [G].

## Background
This method originates from research on isosurface extraction from massive binary volumes, specifically applied to salt dome data [S1, unofficial]. It was developed by Lobello et al. (2014) and is described in the thesis "EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS" [S1, unofficial]. The technique addresses limitations of parallel surface extraction in distributed or hierarchical memory layouts, where data locality constraints prevent adjacent cells in different meta-cells from being processed together [S1, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
