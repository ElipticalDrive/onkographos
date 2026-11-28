# Labeled Volume Creation via MultiBlock Iteration

Also called: Labeled Volume Creation via Sample and Split

## Overview
This method voxelize a MultiBlock mesh into a single `RectilinearGrid` volume where each voxel is labeled with the integer ID of the specific sub-mesh block it intersects, rather than using a binary inside/outside flag [S1, unofficial]. It differs from [Relaxed Connectivity Voxelization](2481-relaxed-connectivity-voxelization.md) and [Polygon Mesh Voxelization via Union of Sets](4605-polygon-mesh-voxelization-via-union-of-sets.md) by explicitly preserving block identity through iterative enclosure testing or sampling, enabling distinct material or region assignment per block in the final volume [S1, unofficial]. It builds on [Mesh Topology and Robust Digitization](8496-mesh-topology-and-robust-digitization.md) by enforcing topological consistency across multiple mesh blocks during discretization [G].

## How it works
**Inputs and Outputs**
- **Input**: `mesh` (`pyvista.DataSet` or `MultiBlock`), `density` (`float` or `array_like[float]`), `check_surface` (`bool`, default `True`) [S1, unofficial].
- **Output**: `voi` (`pyvista.RectilinearGrid` with `InsideMesh` cell data) for Method 1, or `multiblock` (`MultiBlock` of grids) for Method 2 [S1, unofficial].

**Data Structures**
- **`voi`**: `pyvista.RectilinearGrid` defined by coordinate arrays `x`, `y`, `z` [S1, unofficial].
- **`InsideMesh`**: cell data array of size `voi.n_cells`, initialized to zero [S1, unofficial].
- **`mask_vol`**: boolean array of size `voi.n_points` indicating enclosed grid points [S1, unofficial].
- **`cell_ids`**: 1D integer array of unique voxel indices interior to a sub-mesh [S1, unofficial].

**Parameters**
- `density`: uniform voxel size. Defaults to `mesh.length / 100` if `None` [S1, unofficial].
- `check_surface`: specifies whether to check the surface for closure. If `True`, raises a runtime error if the surface is not closed and manifold [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import pyvista as pv
import collections.abc

def voxelize_volume_multiblock_label(mesh, density=None, check_surface=True):
    """Method 1: Iterative Enclosure"""
    mesh = pv.wrap(mesh)
    
    # Determine density
    if density is None:
        density = mesh.length / 100
    if isinstance(density, (int, float, np.number)):
        density_x, density_y, density_z = [density] * 3
    elif isinstance(density, (collections.abc.Sequence, np.ndarray)):
        density_x, density_y, density_z = density
    else:
        raise TypeError(f"Invalid density {density!r}, expected number or array-like.")
    
    # Define grid bounds
    x_min, x_max, y_min, y_max, z_min, z_max = mesh.bounds
    x = np.arange(x_min, x_max, density_x)
    y = np.arange(y_min, y_max, density_y)
    z = np.arange(z_min, z_max, density_z)
    
    voi = pv.RectilinearGrid(x, y, z)
    voi["InsideMesh"] = np.zeros(voi.n_cells)
    
    # Iterate over sub-meshes
    for i, sub_mesh in enumerate(mesh):
        sub_mesh_number = i + 1
        surface = sub_mesh.extract_geometry()
        
        # Pre-processing
        if not surface.faces.size:
            raise ValueError("Input mesh must have faces for voxelization.")
        if not surface.is_all_triangles:
            surface.triangulate(inplace=True)
            
        # Enclosure test
        selection = voi.select_enclosed_points(surface, tolerance=0.0, check_surface=check_surface)
        mask_vol = selection.point_data["SelectedPoints"].view(np.bool_)
        
        # Identify voxels
        cell_ids = np.unique(voi.extract_points(np.argwhere(mask_vol))["vtkOriginalCellIds"])
        voi["InsideMesh"][cell_ids] = sub_mesh_number
        
    return voi

def voxelize_sample_split_label(mesh, density=0.01, tolerance=1):
    """Method 2: Sampling and Split"""
    # Assume mesh already has cell data labels assigned (e.g., mesh['labels'] = block_indices)
    vox = pv.voxelize(mesh, density=density)
    sampled = vox.sample(target=mesh, tolerance=tolerance)
    sampled.set_active_scalars('labels')
    
    # Convert point data to cell data
    sampled = sampled.ptc()
    sampled['labels'] = np.round(sampled['labels'])
    
    # Split into MultiBlock based on unique label values
    multiblock = sampled.split_values()
    return multiblock
```

**Edge Cases and Pitfalls**
- **Empty Mesh**: If `surface.faces.size == 0`, the method raises `ValueError` [S1, unofficial].
- **Non-Triangular Meshes**: Non-triangular surfaces may cause artifacts; triangulation is recommended [S1, unofficial].
- **Sampling Tolerance**: In Method 2, if `tolerance` is too small, data may not be sampled correctly [S1, unofficial].
- **Point vs. Cell Data**: Method 2 requires `ptc()` conversion because `sample` only handles point data, which can introduce interpolation errors [S1, unofficial].
- **Surface Closure**: Method 1 checks for surface closure if `check_surface=True`; non-manifold or open surfaces raise errors [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Method 1 is robust for small to medium meshes with few blocks. Method 2 is more scalable for complex meshes but depends on the efficiency of `pyvista.voxelize` and `sample` [S1, unofficial].
- **Strengths**: Method 1 provides precise block identification without interpolation. Method 2 leverages existing PyVista filters for simplicity [S1, unofficial].
- **Limitations**: Method 1 can be slow for many blocks. Method 2 introduces interpolation artifacts and requires specific PyVista versions for `split_values` [S1, unofficial].

## Background
This method originates from discussions in the PyVista GitHub repository regarding labeled volume creation for MultiBlock meshes [S1, unofficial]. It was proposed as a solution to create voxel volumes where each voxel is labeled with the ID of the sub-mesh block it intersects, addressing limitations in standard voxelization methods that only provide binary inside/outside flags [S1, unofficial]. The iterative enclosure approach was implemented by MosGeo, while the sampling-based approach was suggested by user27182 [S1, unofficial].

## Sources
[S1] Labeled volumes for MultiBlock with voxelize_volume and voxelize, https://github.com/pyvista/pyvista/issues/6255, fetched 2026-10-28 (unofficial)
