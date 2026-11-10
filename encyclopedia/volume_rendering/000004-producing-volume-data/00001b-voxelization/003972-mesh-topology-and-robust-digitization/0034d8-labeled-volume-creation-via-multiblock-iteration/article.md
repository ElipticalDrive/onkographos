# Labeled Volume Creation via MultiBlock Iteration

Also called: Labeled Volume Creation via Sample and Split

## Overview
This method generates a single `RectilinearGrid` volume from a MultiBlock mesh where each voxel is labeled with the integer ID of the specific sub-mesh block it intersects, rather than using a binary inside/outside flag [S1, unofficial]. It differs from [Relaxed Connectivity Voxelization](2481-relaxed-connectivity-voxelization.md) and [Polygon Mesh Voxelization via Union of Sets](4605-polygon-mesh-voxelization-via-union-of-sets.md) by explicitly preserving block identity through iterative enclosure testing or sampling, enabling distinct material or region assignment per block in the final volume. It builds on [Mesh Topology and Robust Digitization](8496-mesh-topology-and-robust-digitization.md) by enforcing topological consistency across multiple mesh blocks during discretization.

## How it works
The method provides two primary implementation strategies: an iterative enclosure-based approach (`voxelize_volume_multiblock_label`) and a sampling-based approach (`voxelize_sample_split_label`). Both produce a labeled volume but differ in computational strategy and data handling.

### Inputs and Outputs
- **Input**: A `pyvista.MultiBlock` or `pyvista.DataSet` mesh containing multiple sub-meshes (blocks). Each block represents a distinct region.
- **Output**: A `pyvista.RectilinearGrid` (or `MultiBlock` of grids in the sampling variant) where cell data contains an integer label identifying the originating sub-mesh block.

### Data Structures
- **`voi` (Volume of Interest)**: A `pyvista.RectilinearGrid` defined by coordinate arrays `x`, `y`, `z`.
- **`InsideMesh`**: A cell data array of type `int` (or `float` in sampling variant) with size `voi.n_cells`, initialized to zero.
- **`mask_vol`**: A boolean array of size `voi.n_points` indicating which grid points are enclosed by a sub-mesh.
- **`cell_ids`**: A 1D integer array of unique voxel indices identified as interior to a sub-mesh.

### Method 1: Iterative Enclosure (`voxelize_volume_multiblock_label`)
This method iterates over each sub-mesh, tests for enclosure within a pre-defined grid, and assigns labels.

#### Parameters
- `mesh`: `pyvista.DataSet`, the input MultiBlock mesh.
- `density`: `float` or `array_like[float]`, the uniform size of the voxels. Defaults to `mesh.length / 100` if `None`.
- `check_surface`: `bool`, default `True`. Specifies whether to check the surface for closure. If `True`, raises a runtime error if the surface is not closed and manifold.

#### Steps
1. **Wrap Mesh**: Convert input to `pyvista.DataSet` using `pyvista.wrap(mesh)` [S1, unofficial].
2. **Determine Density**:
   - If `density` is `None`, set `density = mesh.length / 100` [S1, unofficial].
   - If `density` is a scalar (`int`, `float`, `np.number`), set `density_x = density_y = density_z = density` [S1, unofficial].
   - If `density` is a sequence or `np.ndarray`, unpack into `density_x`, `density_y`, `density_z` [S1, unofficial].
   - Else, raise `TypeError` [S1, unofficial].
3. **Define Grid Bounds**:
   - Get `mesh.bounds` as `(x_min, x_max, y_min, y_max, z_min, z_max)` [S1, unofficial].
   - Create coordinate arrays:
     $$ x = \text{np.arange}(x_{min}, x_{max}, density_x) $$
     $$ y = \text{np.arange}(y_{min}, y_{max}, density_y) $$
     $$ z = \text{np.arange}(z_{min}, z_{max}, density_z) $$
   - Initialize `voi` as `pyvista.RectilinearGrid(x, y, z)` [S1, unofficial].
4. **Initialize Label Array**:
   - Set `voi['InsideMesh'] = np.zeros(voi.n_cells)` [S1, unofficial].
5. **Iterate Over Sub-Meshes**:
   - For each `sub_mesh` in `mesh` with index `i`:
     - Set `sub_mesh_number = i + 1` [S1, unofficial].
     - Extract geometry: `surface = sub_mesh.extract_geometry()` [S1, unofficial].
     - **Pre-processing**:
       - If `surface.faces.size == 0`, raise `ValueError("Input mesh must have faces for voxelization.")` [S1, unofficial].
       - If `not surface.is_all_triangles`, call `surface.triangulate(inplace=True)` to reduce artifacts [S1, unofficial].
     - **Enclosure Test**:
       - Call `selection = voi.select_enclosed_points(surface, tolerance=0.0, check_surface=check_surface)` [S1, unofficial].
       - Extract boolean mask: `mask_vol = selection.point_data['SelectedPoints'].view(np.bool_)` [S1, unofficial].
     - **Identify Voxels**:
       - Get indices of true points: `point_indices = np.argwhere(mask_vol)` [S1, unofficial].
       - Extract points from `voi` at these indices: `pts = voi.extract_points(point_indices)` [S1, unofficial].
       - Get unique cell IDs: `cell_ids = np.unique(pts['vtkOriginalCellIds'])` [S1, unofficial].
     - **Assign Label**:
       - Set `voi['InsideMesh'][cell_ids] = sub_mesh_number` [S1, unofficial].
6. **Return**: Return `voi` [S1, unofficial].

### Method 2: Sampling and Split (`voxelize_sample_split_label`)
This method voxelize the entire mesh, samples the original mesh's cell data onto the voxels, converts point data to cell data, and splits the result into a MultiBlock based on unique labels.

#### Steps
1. **Prepare Input Mesh**:
   - Load mesh and add cell data labels (e.g., `labels = np.zeros((mesh.n_cells,))`; `labels[:700] = 1`; `mesh['labels'] = labels`) [S1, unofficial].
2. **Voxelize**:
   - Call `vox = pv.voxelize(mesh, density=0.01)` [S1, unofficial].
3. **Sample Data**:
   - Call `sampled = vox.sample(target=mesh, tolerance=1)` [S1, unofficial].
   - Note: `tolerance` must be sufficiently large (e.g., 1 or `inf`) to ensure correct sampling [S1, unofficial].
   - Set active scalars: `sampled.set_active_scalars('labels')` [S1, unofficial].
4. **Convert Point to Cell Data**:
   - Call `sampled = sampled.ptc()` (point-to-cell) [S1, unofficial].
   - Note: This step is necessary because `sample` operates on point data, but cell data is desired. This interpolation is noted as potentially "hacky" due to double interpolation [S1, unofficial].
5. **Round Labels**:
   - Set `sampled['labels'] = np.round(sampled['labels'])` [S1, unofficial].
6. **Split into MultiBlock**:
   - Call `multiblock = sampled.split_values()` [S1, unofficial].
   - Note: `split_values` is a filter available from `pyvista==0.44.0` main branch [S1, unofficial].
7. **Return**: Return `multiblock` [S1, unofficial].

### Edge Cases and Pitfalls
- **Empty Mesh**: If `surface.faces.size == 0`, the method raises `ValueError` [S1, unofficial].
- **Non-Triangular Meshes**: Non-triangular surfaces may cause artifacts; triangulation is recommended [S1, unofficial].
- **Sampling Tolerance**: In Method 2, if `tolerance` is too small, data may not be sampled correctly [S1, unofficial].
- **Point vs. Cell Data**: Method 2 requires `ptc()` conversion because `sample` only handles point data, which can introduce interpolation errors [S1, unofficial].
- **Surface Closure**: Method 1 checks for surface closure if `check_surface=True`; non-manifold or open surfaces raise errors [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Method 1 is $O(N \cdot M)$ where $N$ is the number of voxels and $M$ is the number of sub-meshes, due to iterative enclosure testing. Method 2 is dominated by the initial voxelization and sampling steps, which are generally more efficient for large meshes but involve interpolation overhead.
- **Memory Complexity**: Method 1 stores a single `RectilinearGrid` with label data. Method 2 may require additional memory for intermediate sampled data and the split MultiBlock structure.
- **Performance**: Method 1 is robust for small to medium meshes with few blocks. Method 2 is more scalable for complex meshes but depends on the efficiency of `pyvista.voxelize` and `sample`.
- **Strengths**: Method 1 provides precise block identification without interpolation. Method 2 leverages existing PyVista filters for simplicity.
- **Limitations**: Method 1 can be slow for many blocks. Method 2 introduces interpolation artifacts and requires specific PyVista versions for `split_values`.

## Background
This method originates from discussions in the PyVista GitHub repository regarding labeled volume creation for MultiBlock meshes [S1, unofficial]. It was proposed as a solution to create voxel volumes where each voxel is labeled with the ID of the sub-mesh block it intersects, addressing limitations in standard voxelization methods that only provide binary inside/outside flags. The iterative enclosure approach was implemented by MosGeo, while the sampling-based approach was suggested by user27182 [S1, unofficial].

## Sources
[S1] Labeled volumes for MultiBlock with voxelize_volume and voxelize, https://github.com/pyvista/pyvista/issues/6255, fetched 2026-10-28 (unofficial)
