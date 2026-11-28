# Extended Patch Isosurface Extraction for Node-Centered Data

Also called: Extended Patch Pyramid Decomposition, Fang et al. (2004) method

## Overview
Extended Patch Isosurface Extraction is a method for extracting crack-free isosurfaces from node-centered Adaptive Mesh Refinement (AMR) data [S1, S2, unofficial]. It solves the problem of geometric discontinuities (cracks) at level boundaries by preserving the original patch structure and cell identity, allowing users to determine which patch cell generated each isosurface triangle [S1, S2, unofficial]. Unlike methods that strictly enforce hierarchical consistency through interpolation alone, this approach explicitly decomposes coarse cells at refinement boundaries into pyramids, ensuring that standard marching cubes algorithms can be applied without producing t-junctions or cracks [S1, S2, unofficial].

## How it works
**Inputs:**
- Node-centered scalar field values on a hierarchical AMR grid.
- Patch hierarchy structure defining refinement levels and boundaries.

**Outputs:**
- A set of triangles forming the isosurface.
- Metadata linking each triangle to its source patch and cell identity.

**Data Structures:**
The method operates on the existing AMR patch structure. No new global grid is created. Instead, the geometry is redefined locally at boundaries:
1.  **Extended Patches:** Refined patches are logically extended to cover regions where dangling nodes exist, ensuring consistent value assignment.
2.  **Pyramid Cells:** Coarse-level cells at the boundary to a finer level are decomposed into pyramids. Each pyramid connects the coarse cell center to the vertices of a subdivided face facet.

**Steps:**

1.  **Extend Refined Patches:**
    Extend the refined patches until it is possible to assign values to dangling nodes that are consistent with interpolation results in the coarser level [S1, S2, unofficial]. This ensures that the scalar field is well-defined at all node locations required for marching cubes.

2.  **Decompose Coarse Cells at Boundaries:**
    For every coarse-level cell that is adjacent to a finer level, decompose the cell into a set of pyramids [S1, S2, unofficial].
    -   Identify the boundary faces of the coarse cell that touch the finer level.
    -   For each such face, subdivide it into facets corresponding to the finer grid resolution.
    -   Create a separate pyramid for each facet of a subdivided face [S1, S2, unofficial].
    -   Each pyramid connects the coarse cell center to the vertices of its corresponding facet.

3.  **Extract Isosurface:**
    Apply marching cubes to the resulting cell types (standard hexahedra in uniform regions, pyramids at boundaries) using appropriate case tables for these new cell types [S1, S2, unofficial]. The decomposition ensures that marching cubes will not produce cracks in the extracted isosurface [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_patch_extraction(amr_data, isovalue):
    """
    Extract isosurface from node-centered AMR data using extended patches and pyramid decomposition.
    
    Args:
        amr_data: AMR grid structure with node-centered scalar values.
        isovalue: Scalar threshold for isosurface extraction.
        
    Returns:
        triangles: List of triangle vertices.
        patch_ids: List of patch IDs corresponding to each triangle.
    """
    triangles = []
    patch_ids = []
    
    # Step 1: Extend refined patches to handle dangling nodes
    # Assign values to dangling nodes consistent with coarser level interpolation
    extended_data = extend_patches(amr_data)
    
    # Step 2: Process each patch
    for patch in extended_data.patches:
        if patch.is_boundary_to_finer_level():
            # Decompose coarse cells at boundary into pyramids
            pyramids = decompose_coarse_cells_into_pyramids(patch)
            
            # Extract isosurface from pyramids
            for pyramid in pyramids:
                tri_list, pid = marching_cubes_pyramid(pyramid, isovalue)
                triangles.extend(tri_list)
                patch_ids.extend([pid] * len(tri_list))
        else:
            # Standard marching cubes for non-boundary cells
            tri_list, pid = marching_cubes_standard(patch, isovalue)
            triangles.extend(tri_list)
            patch_ids.extend([pid] * len(tri_list))
            
    return triangles, patch_ids

def extend_patches(amr_data):
    """
    Extend refined patches to assign values to dangling nodes.
    Values are assigned to be consistent with interpolation results in the coarser level.
    """
    # Implementation detail: Interpolate values from coarser level to dangling nodes
    # and logically extend the patch boundaries to include these nodes.
    # [G] Specific interpolation method (e.g., trilinear) is not specified in sources.
    pass

def decompose_coarse_cells_into_pyramids(patch):
    """
    Decompose coarse-level cells at the boundary to a finer level into pyramids.
    Each pyramid connects the cell center to a facet of a subdivided face.
    """
    pyramids = []
    for cell in patch.cells:
        if cell.is_adjacent_to_finer_level():
            boundary_faces = cell.get_boundary_faces()
            for face in boundary_faces:
                facets = face.subdivide_to_finer_resolution()
                for facet in facets:
                    pyramid = Pyramid(center=cell.center, base=facet)
                    pyramids.append(pyramid)
    return pyramids

def marching_cubes_pyramid(pyramid, isovalue):
    """
    Apply marching cubes to a pyramid cell.
    Returns list of triangles and the patch ID.
    """
    # [G] Specific case tables for pyramids are not provided in sources.
    # Standard marching cubes logic adapted for pyramid topology.
    pass

def marching_cubes_standard(patch, isovalue):
    """
    Apply standard marching cubes to a non-boundary cell.
    """
    pass
```

**Parameters:**
-   **Isovalue:** The scalar threshold for isosurface extraction. Typical values depend on the specific dataset.
-   **Interpolation Method:** Used to assign values to dangling nodes. Not specified in sources [G].

**Edge Cases and Pitfalls:**
-   **Dangling Nodes:** If dangling nodes are not properly assigned values consistent with the coarser level, cracks may still occur.
-   **Boundary Detection:** Correctly identifying coarse cells adjacent to finer levels is critical for proper pyramid decomposition.
-   **Case Tables:** Appropriate case tables for the new pyramid cell types must be implemented for marching cubes to work correctly [S1, S2, unofficial].

## Subtopics

### Cell Subdivision Isosurface Extraction
The parent method that eliminates non-linear hanging nodes by subdividing boundary cells into pyramids. Extended Patch Isosurface Extraction builds on this by preserving patch identity and extending patches for node-centered data [10502-cell-subdivision-isosurface-extraction.md].

### Marching Cubes on Structured Grids
The base algorithm used for isosurface extraction within the cells (including pyramids) created by this method.

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Strengths:** Preserves original patch structure and cell identity, allowing users to determine which patch cell generated each triangle [S1, S2, unofficial]. Ensures crack-free isosurfaces at level boundaries [S1, S2, unofficial].
-   **Limitations:** Requires implementation of specific case tables for pyramid cells [S1, S2, unofficial]. Relies on consistent interpolation for dangling node values.
-   **When to Use:** When preserving patch identity is important and the data is node-centered [S1, S2, unofficial].
-   **When Not to Use:** When cell-centered data is used, or when patch identity is not required.

## Background
Fang et al. (2004) presented this alternate isosurface extraction approach for node-centered AMR data [S1, S2, unofficial]. The method was developed to address the limitations of previous approaches that did not preserve cell identity or required complex handling of hanging nodes. It is used in the context of volume rendering of AMR data, where maintaining the association between geometry and source data is valuable for analysis [S1, S2, unofficial].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc894061/, fetched 2026-10-26 (unofficial)
[S2] Visualization of Scalar Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc901373/?q=adaptive mesh refinement visualization ray casting, fetched 2026-10-28 (unofficial)
