# Geometry Cleaning

Also called: Clean-up, Geometry Filter, PolyData Geometry Cleaning, vtkPVGeometryFilter

## Overview
Geometry Cleaning is a post-processing step for polygonal meshes extracted from volumetric data, specifically designed to remove artifacts introduced by discretization and snapping processes. It solves the problem of non-manifold geometry, duplicate vertices, and degenerate cells that arise when converting voxel grids to triangle meshes. Unlike Connected Component Filtering, which removes isolated fragments based on connectivity, Geometry Cleaning focuses on topological validity by eliminating redundant and invalid geometric primitives to produce a clean, watertight manifold mesh.

## How it works
**Inputs and Outputs:**
- Input: A `vtkPolyData` object containing a mesh with potential duplicates, unused points, and degenerate cells (e.g., output from `vtkMarchingCubes`).
- Output: A cleaned `vtkPolyData` object with valid manifold topology.

**Data Structures:**
The method operates on `vtkPolyData`, which consists of:
- `Points`: A list of 3D coordinates.
- `Verts`, `Lines`, `Polys`: Cell definitions referencing point indices.
- `PointData` and `CellData`: Associated attributes.

**Steps:**
1. **Initialize Filter:** Create an instance of the geometry cleaning filter (e.g., `vtkPVGeometryFilter` or equivalent cleanup routine) [S2, unofficial].
2. **Connect Input:** Set the input mesh to the filter using the `SetInput` method [S2, unofficial].
3. **Execute Cleaning:** The filter performs the following internal operations:
   - **Remove Duplicate Points:** Identifies points with identical coordinates and merges them into a single vertex, updating all cell references [S2, unofficial].
   - **Remove Unused Points:** Deletes points that are not referenced by any cell in the mesh [S2, unofficial].
   - **Remove Degenerate Cells:** Identifies and removes triangles (or other polygons) with zero area, such as those with coincident vertices or collinear edges, which are often produced by snapping algorithms [S1, unofficial].
   - **Remove Double Triangles:** Eliminates duplicate triangles that occupy the same geometric space [S1, unofficial].
4. **Output Result:** Retrieve the cleaned mesh from the filter's output port [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming a VTK-like API structure
def geometry_cleaning(input_polydata):
    """
    Cleans up a polydata mesh by removing duplicates, unused points, and degenerate cells.
    
    Args:
        input_polydata: vtkPolyData object containing the raw mesh.
        
    Returns:
        vtkPolyData: The cleaned mesh.
    """
    # Step 1: Create the geometry filter instance
    # Note: In VTK, this is typically vtkPVGeometryFilter or vtkCleanPolyData
    # depending on the specific variant of "cleaning" required.
    # Based on [S2, unofficial], vtkPVGeometryFilter is used in the context of VTK volume rendering pipelines.
    geometry_filter = vtkPVGeometryFilter()
    
    # Step 2: Connect the input polydata
    geometry_filter.SetInput(input_polydata)
    
    # Step 3: Execute the filter
    # The filter internally handles:
    # - Removing duplicate points
    # - Removing unused points
    # - Removing degenerate and double triangles [S1, S2, unofficial]
    geometry_filter.Update()
    
    # Step 4: Retrieve the output
    cleaned_polydata = geometry_filter.GetOutput()
    
    return cleaned_polydata
```

**Parameters:**
- Not given in the sources. Standard VTK geometry filters may have parameters for tolerance, but specific values are not provided in the cited texts.

**Edge Cases and Pitfalls:**
- **Precision:** Snapping algorithms may produce nearly coincident points that are not exactly identical due to floating-point errors. The cleaning step must handle a tolerance threshold to merge these points correctly [S1, unofficial].
- **Degenerate Input:** If the input mesh contains only degenerate cells, the output may be empty.
- **Manifold Validity:** While cleaning removes degenerate and double triangles, it does not necessarily fix non-manifold edges or vertices unless specifically configured to do so. The result is described as a "valid manifold mesh" in the context of the snapping algorithm's output [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. Typically, point merging and cell removal are linear or near-linear with respect to the number of points and cells.
- **Memory Complexity:** Not given in the sources. The output mesh size is generally smaller than or equal to the input mesh size.
- **Strengths:** Produces a watertight, manifold mesh suitable for rendering and further processing. Removes artifacts from voxel-to-surface conversion.
- **Limitations:** Does not remove isolated components (requires Connected Component Filtering). Does not optimize mesh quality (e.g., vertex smoothing) unless an additional optimization step is applied [S1, unofficial].
- **When to Use:** After isosurface extraction (e.g., Marching Cubes) when the source algorithm produces degenerate or duplicate geometry.
- **When Not to Use:** When the mesh is already clean, or when isolated components need to be removed instead of topological errors.

## Background
Geometry Cleaning is a standard operation in VTK (Visualization Toolkit) pipelines for volume rendering. It is often used after polygonization steps like `vtkMarchingCubes` to ensure the resulting mesh is valid for rendering and analysis. The specific filter `vtkPVGeometryFilter` is highlighted in VTK-based volume rendering examples [S2, unofficial]. The need for cleaning arises because snapping algorithms, which map voxel surfaces to triangle meshes, are not injective and can produce degenerate and double triangles [S1, unofficial].

## Sources
[S1] , https://www.scitepress.org/PublishedPapers/2007/20798/pdf/index.html, fetched 2026-10-27 (unofficial)
[S2] Isosurface Generation using VTK: Trying Different Volumes of Interest, https://www.evl.uic.edu/aspale/cs526/final/3-5-4-0.htm, fetched 2026-10-30 (unofficial)
