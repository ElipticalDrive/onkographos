# Gap and Seam Handling in Chunked Volume Rendering

Also called: Overlapping Cell Contouring, Seam Mesh Generation

## Overview
Chunked volume rendering splits data into independent octants to enable culling, but this isolation causes disconnected isosurface meshes at cell boundaries, creating visual gaps [S1]. This method addresses the problem of connecting these disjointed meshes, with a specific focus on maintaining connectivity across different Levels of Detail (LOD) [S1]. It is distinguished from sibling methods like [Static Level of Detail Generation via Vertex Clustering](9052-static-level-of-detail-generation-via-vertex-clustering.md) by explicitly managing the topology at the boundaries of spatial partitions rather than relying on vertex merging or clustering to resolve discontinuities [S1].

## How it works
The method consists of two primary approaches: Overlapping Cells and Seam Mesh Generation. The choice depends on whether the rendering system uses uniform LOD or mixed LODs across adjacent cells.

### Inputs and Outputs
*   **Inputs:**
    *   Volume data partitioned into spatial cells (octants).
    *   A defined Level of Detail (LOD) for each cell.
    *   Adjacency information between cells.
*   **Outputs:**
    *   A continuous isosurface mesh without gaps.
    *   Seam meshes for mixed-LOD boundaries.

### Data Structures
*   **Cell Data:** Standard voxel grid for each cell.
*   **Seam Mesh:** A separate mesh structure storing vertices and indices that bridge gaps between cells of different LODs.
*   **Border Voxels:** A subset of voxels exported from contouring tasks at LOD boundaries.

### Steps

#### Approach 1: Overlapping Cells (Same-LOD Connectivity)
This approach ensures natural connectivity for meshes of the same LOD by extending data domains [S1].

1.  **Extend Data Domain:** For each cell, extend its data domain by one voxel in the positive directions (e.g., +x, +y, +z) to include overlapping boundary voxels [S1].
2.  **Contouring:** Perform contouring on the extended cell data.
3.  **Duplicate Prevention:** During contouring, ensure that additional voxels within the extended boundary are not connected among each other if they are shared with a neighbor. Specifically, if Cell A extends into Cell B, the edge connecting vertices $b$ and $d$ with $a_0$ and $c_0$ (to close the gap) must only be created by one cell to prevent duplicate polygons [S1].
    *   *Note:* The exact algorithm for determining which cell owns the shared edge is not fully specified in the sources, but the constraint is that shared edges must not be generated twice [S1].

#### Approach 2: Seam Mesh Generation (Mixed-LOD Connectivity)
This approach is required when adjacent cells have different LODs, as overlapping cells fail to connect meshes of different details due to extra vertices creating cracks or holes [S1].

1.  **Identify Border Tasks:** Identify contouring tasks that process cells located at the border of a LOD shell [S1].
2.  **Export Border Voxels:** Export the bordering voxels from these identified contouring tasks [S1].
3.  **Create Seam Mesh:** Use the exported bordering voxels to create a seam mesh that fills the gap between the LOD shells [S1].
4.  **Update Seam Mesh:** Whenever a contouring task is performed on a cell that touches the seam, update the seam mesh accordingly [S1].
5.  **Boundary Constraint:** Ensure that meshes generated from cells at LOD borders do not extend into the seam mesh area to prevent geometry overlap [S1].
6.  **Duplicate Prevention:** The seam contouring process must only connect voxels from different data sets to prevent the creation of duplicate polygons [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_cell(cell_data, cell_id, is_border_cell, seam_mesh_manager):
    """
    Contour a single cell, handling gaps based on the approach.
    
    Args:
        cell_data: The voxel data for the cell.
        cell_id: Identifier for the cell.
        is_border_cell: Boolean indicating if the cell is at a LOD boundary.
        seam_mesh_manager: Object managing seam mesh creation and updates.
    """
    if is_border_cell:
        # Export bordering voxels for seam mesh generation
        border_voxels = export_border_voxels(cell_data)
        seam_mesh_manager.update_seam(cell_id, border_voxels)
        
        # Contour the cell, ensuring no extension into seam mesh area
        mesh = contour(cell_data, exclude_seam_area=True)
    else:
        # Standard contouring for non-border cells
        # If using overlapping cells, extend data domain by 1 voxel in positive directions
        # and handle duplicate edge prevention for shared boundaries
        extended_data = extend_data_domain_positive(cell_data, overlap=1)
        mesh = contour(extended_data, prevent_duplicate_edges=True)
        
    return mesh

def prevent_duplicate_edges(mesh, neighbor_mesh):
    """
    Prevents duplicate polygon generation at overlapping boundaries.
    Specifically, ensures that edges like the one connecting vertices b and d 
    with a_0 and c_0 are not created twice by adjacent cells.
    """
    # Logic to identify shared edges between overlapping cells
    # and remove duplicates based on cell ownership rules
    pass

def extend_data_domain_positive(data, overlap=1):
    """
    Extends the data domain by 'overlap' voxels in positive directions.
    """
    # Implementation details for extending voxel grid
    pass

def export_border_voxels(cell_data):
    """
    Extracts voxels located at the border of the cell.
    """
    # Implementation details for extracting border voxels
    pass
```

### Parameters
*   **Overlap Size:** 1 voxel in positive directions for overlapping cells approach [S1].
*   **Seam Update Trigger:** Whenever a contouring task is performed on a cell touching the seam [S1].

### Edge Cases and Pitfalls
*   **Mixed-LOD Gaps:** Overlapping cells fail to connect meshes of different LODs because extra vertices from higher LOD meshes create cracks or holes when merged with lower LOD meshes [S1]. Seam meshes are required in this case [S1].
*   **Duplicate Polygons:** In the overlapping cells approach, additional voxels must not be connected among each other within the same cell's extended boundary, as neighboring cells will also generate these polygons, causing duplicates [S1].
*   **Geometry Overlap:** Meshes generated from cells at LOD borders must not extend into the seam mesh area to prevent geometry overlap [S1].
*   **Specific Duplicate Edge Case:** In overlapping cells A and B, if A extends into B, the edge connecting vertices $b$ and $d$ with $a_0$ and $c_0$ (closing the gap) must not be created twice by both cells [S1].
*   **Incomplete Solution:** The source notes that due to a lack of information on the duplicate edge issue, the developed system did not include a complete gap closing solution [S1].

## Operating characteristics
*   **Overlapping Cells:**
    *   **Pros:** Low overhead, natural connection for same-LOD meshes, considered the most elegant solution for same-LOD connectivity [S1].
    *   **Cons:** Fails for mixed-LOD scenarios; introduces redundant data storage overhead as all cells must contain overlapping voxel data [S1].
*   **Seam Mesh Generation:**
    *   **Pros:** Required for mixed-LOD connectivity, prevents cracks and holes between different LODs [S1].
    *   **Cons:** Adds complexity to the rendering pipeline; requires managing seam mesh updates and ensuring no geometry overlap [S1].
*   **Performance:** The source mentions performance tests on an LG Google Nexus 5X smartphone using Android 8.1.0 and Chrome App 7.17.28.21.arm64, but specific metrics for gap handling are not provided [S1].

## Background
The problem of gaps in chunked volume rendering arises when splitting surface meshes into patches for culling, which increases draw calls and reduces performance if not managed correctly [S1]. The issue is particularly challenging when a LOD system is involved [S1]. Two known approaches to solving this problem are letting cells overlap by one voxel in each positive direction and gathering bordering voxels from neighboring cells to create a seam mesh [S1]. The overlapping cells approach is attributed to reference [6], while the seam mesh approach is attributed to references [8] and [6] [S1]. The source notes that the developed system did not include a complete gap closing solution due to a lack of information on the duplicate edge issue [S1].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
