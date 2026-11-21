# Labeled Volume Creation via MultiBlock Iteration

Also called: Labeled Volume Creation via Sample and Split

## Overview
This method generates a single `RectilinearGrid` volume from a `MultiBlock` mesh where each voxel is labeled with the integer ID of the specific sub-mesh block it intersects, rather than a binary inside/outside flag [S1, unofficial]. It sets itself apart from sibling methods like [Polygon Mesh Voxelization via Union of Sets](4605-polygon-mesh-voxelization-via-union-of-sets.md) by preserving block identity through iterative enclosed-point testing, and from [Improved Tetrahedral Mesh Generation for Voxelization](7476-improved-tetrahedral-mesh-generation-for-voxelization.md) by operating directly on polygonal surfaces without tetrahedralization [S1, unofficial].

## How it works
**Inputs:**
- `mesh`: A `pyvista.MultiBlock` dataset containing multiple sub-meshes.
- `density`: Voxel size. Can be a single scalar (uniform) or a sequence of three scalars `(density_x, density_y, density_z)` [S1, unofficial].
- `check_surface`: Boolean flag (default `True`) to validate surface closure and manifoldness [S1, unofficial].

**Outputs:**
- `voi`: A `pyvista.RectilinearGrid` where the cell data array `InsideMesh` contains the 1-based index of the sub-mesh block that encloses each voxel. Voxels outside all blocks are labeled `0` [S1, unofficial].

**Data Structures:**
- `voi`: `RectilinearGrid` defined by coordinate arrays `x`, `y`, `z`.
- `InsideMesh`: 1D integer array of size `voi.n_cells`, initialized to zeros.
- `mask_vol`: Boolean array of size `voi.n_points`, indicating which grid points are enclosed by a specific sub-mesh.
- `cell_ids`: 1D integer array of unique voxel indices identified as enclosed.

**Steps:**
1.  **Wrap and Validate:** Wrap the input `mesh` using `pyvista.wrap(mesh)`. If `density` is `None`, set it to `mesh.length / 100` [S1, unofficial].
2.  **Parse Density:**
    - If `density` is a scalar (`int`, `float`, or `np.number`), set `density_x = density_y = density_z = density` [S1, unofficial].
    - If `density` is a sequence or `np.ndarray`, unpack it into `density_x`, `density_y`, `density_z`.
    - Otherwise, raise `TypeError` [S1, unofficial].
3.  **Define Grid:**
    - Retrieve bounds: `x_min, x_max, y_min, y_max, z_min, z_max = mesh.bounds` [S1, unofficial].
    - Create coordinate arrays:
      $$x = \text{np.arange}(x_{min}, x_{max}, density_x)$$
      $$y = \text{np.arange}(y_{min}, y_{max}, density_y)$$
      $$z = \text{np.arange}(z_{min}, z_{max}, density_z)$$
    - Initialize `voi` as `pyvista.RectilinearGrid(x, y, z)` [S1, unofficial].
    - Initialize `voi['InsideMesh']` with zeros of length `voi.n_cells` [S1, unofficial].
4.  **Iterate Blocks:** For each `sub_mesh` in `mesh` with index `i` (starting from 0):
    a.  Set `sub_mesh_number = i + 1` [S1, unofficial].
    b.  Extract surface geometry: `surface = sub_mesh.extract_geometry()` [S1, unofficial].
    c.  **Pre-process Surface:**
        - If `surface.faces.size` is 0, raise `ValueError: 'Input mesh must have faces for voxelization.'` [S1, unofficial].
        - If `not surface.is_all_triangles`, call `surface.triangulate(inplace=True)` to reduce artifacts [S1, unofficial].
    d.  **Select Enclosed Points:**
        - Call `selection = voi.select_enclosed_points(surface, tolerance=0.0, check_surface=check_surface)` [S1, unofficial].
        - Extract boolean mask: `mask_vol = selection.point_data['SelectedPoints'].view(np.bool_)` [S1, unofficial].
    e.  **Identify Voxels:**
        - Get indices of true points: `point_indices = np.argwhere(mask_vol)` [S1, unofficial].
        - Extract points from `voi` at these indices and retrieve their original cell IDs:
          `cell_ids = np.unique(voi.extract_points(point_indices)['vtkOriginalCellIds'])` [S1, unofficial].
    f.  **Label Voxels:**
        - Assign the block ID to the volume: `voi['InsideMesh'][cell_ids] = sub_mesh_number` [S1, unofficial].
5.  **Return:** Return `voi` [S1, unofficial].

**Alternative Implementation (Sample and Split):**
An alternative approach, noted as potentially "hacky" due to double interpolation, uses `voxelize`, `sample`, `ptc`, and `split_values` [S1, unofficial]:
1.  Load mesh and add cell data labels (e.g., `mesh['labels'] = labels`) [S1, unofficial].
2.  Voxelize: `vox = pv.voxelize(mesh, density=0.01)` [S1, unofficial].
3.  Sample: `sampled = vox.sample(target=mesh, tolerance=1)`. Note: `tolerance` must be sufficiently large (e.g., 1 or `inf`) [S1, unofficial].
4.  Set active scalars: `sampled.set_active_scalars('labels')` [S1, unofficial].
5.  Convert point data to cell data: `sampled = sampled.ptc()` [S1, unofficial].
6.  Round labels: `sampled['labels'] = np.round(sampled['labels'])` [S1, unofficial].
7.  Split: `multiblock = sampled.split_values()` (requires `pyvista>=0.44.0`) [S1, unofficial].

**Pitfalls:**
- **Empty Mesh:** If the input mesh has no faces, the method raises a `ValueError` [S1, unofficial].
- **Non-Triangular Faces:** Non-triangular faces should be triangulated in-place to avoid artifacts [S1, unofficial].
- **Interpolation Artifacts:** In the alternative method, `sample` operates on point data, interpolating cell data. This requires `ptc` conversion and rounding, which may introduce inaccuracies [S1, unofficial].
- **Tolerance:** The `sample` filter requires a large tolerance to ensure correct data transfer [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(B \cdot (V + P))$, where $B$ is the number of blocks, $V$ is the number of voxels, and $P$ is the number of grid points. The dominant cost is the `select_enclosed_points` operation for each block [S1, unofficial].
**Memory Complexity:** $O(V + P)$ for the volume and point data structures [S1, unofficial].
**Performance:** Performance scales linearly with the number of blocks and voxel resolution. The iterative approach is robust for small to medium numbers of blocks but may become slow for very high-resolution grids with many blocks [S1, unofficial].
**Strengths:** Produces a single labeled volume; handles arbitrary block IDs; robust surface checking [S1, unofficial].
**Limitations:** Iterative nature can be slow for many blocks; requires closed, manifold surfaces for accurate enclosed point detection [S1, unofficial].
**When to use:** When block identity is required in the voxelized output and the mesh consists of closed, manifold sub-meshes [S1, unofficial].
**When not to use:** For open surfaces or when performance is critical for very high-resolution grids with many blocks [S1, unofficial].

## Background
This method was discussed in the context of PyVista development to support labeled volume creation for MultiBlock meshes [S1, unofficial]. It was proposed as a solution to create voxel volumes where each cell is labeled with the ID of the intersecting sub-mesh block, addressing a gap in standard voxelization tools that typically produce binary or scalar fields [S1, unofficial]. The alternative "Sample and Split" method was also discussed as a workaround using existing filters, though it was noted to have interpolation drawbacks [S1, unofficial].

## Sources
[S1] Labeled volumes for MultiBlock with voxelize_volume and voxelize, https://github.com/pyvista/pyvista/issues/6255, fetched 2026-10-28 (unofficial)
