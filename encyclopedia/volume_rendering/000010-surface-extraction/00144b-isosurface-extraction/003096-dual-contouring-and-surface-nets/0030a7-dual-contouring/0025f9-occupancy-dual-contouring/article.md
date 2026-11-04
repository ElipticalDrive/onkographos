# Occupancy Dual Contouring

Also called: Occupancy-Based Dual Contouring (ODC)

## Overview
Occupancy Dual Contouring is a surface extraction method that constructs a mesh from a binary occupancy grid by locating vertices at grid edges and solving a Quadratic Error Function (QEF) to determine vertex positions, using dual contouring principles to handle complex topology [S1, unofficial]. It avoids using distance information, instead using auxiliary 2D points to compute local surface normals and identify 3D points via a quadric error function, designed for GPU parallelization [S1, unofficial]. Unlike standard marching cubes which relies on linear interpolation along edges, this method uses hybrid linear/binary searches to find precise intersection points on grid faces and edges, enabling high-fidelity reconstruction from binary voxel data.

## How it works
**Inputs:**
- `imp_func`: An implicit function returning `torch.float64` tensors [S1, unofficial].
- `min_coord`, `max_coord`: Bounding box of the grid space [S1, unofficial].
- `num_grid`: Resolution of the voxel grid (GRID) [S1, unofficial].
- `isolevel`: Threshold for occupancy classification [S1, unofficial].
- `outside`: Boolean flag; if `True`, occupancy is `imp_func(x) < isolevel` [S1, unofficial].
- Parameters: `BINARY_SEARCH`, `VERTICAL_RANGE`, `VERTICAL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_RANGE`, `HORIZNTL_LINEAR_SEARCH`, `HORIZNTL_BINARY_SEARCH`, `ERR`, `QEF_REG` [S1, unofficial].

**Outputs:**
- Vertices: 3D coordinates of mesh vertices.
- Faces: Indices of vertices forming triangular faces.

**Data Structures:**
The method initializes several lookup tables on the GPU device to define the topology and geometry of dual contouring cells [S1, unofficial]:
- `vitogi`: Vertex-to-Grid Index mapping.
- `lctab_2d`: Local cell topology for 2D searches.
- `pfvtab`: Point-face-vertex topology table.
- `qvntab`: Quantity table derived from `pfvtab`.
- `vtstab`: Vertex table storing indices of 3 vertices for each cell configuration.
- `vbitab`: Vertex bit table.
- `cstrtab`: Constraint table for face construction.
- `cpnttab`: Control point table.
- `ambtab`: Ambiguity resolution table.
- `axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`: Axis and case tables for geometry [S1, unofficial].

**Steps:**

1. **Initialize Lookup Tables:** Load precomputed topology tables (`vitogi`, `lctab_2d`, `pfvtab`, `qvntab`, `vtstab`, `vbitab`, `cstrtab`, `cpnttab`, `ambtab`, `axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`) onto the device [S1, unofficial].

2. **Compute Occupancy Grid:** Evaluate `imp_func` on a `GRID x GRID x GRID` grid of points. Convert to binary occupancy values (0 or 1) based on `isolevel` and `outside` flag [S1, unofficial].
   - If `outside` is True: `occupancy = imp_func(x) < isolevel`.
   - If `outside` is False: `occupancy = imp_func(x) > isolevel`.

3. **Compute Cell Configuration Values:** For each cell in the `GRID x GRID x GRID` volume, compute an 8-bit integer representing the occupancy of its 8 corners. The bits are ordered: `[1,1,1]`, `[1,1,0]`, `[1,0,1]`, `[1,0,0]`, `[0,1,1]`, `[0,1,0]`, `[0,0,1]`, `[0,0,0]` relative to the cell's min corner [S1, unofficial].

4. **Identify Ambiguous Cells:** Find cells where `0 < val < 255`. Use `ambtab` to find ambiguous neighbors. Resolve ambiguities by flipping the occupancy of the ambiguous neighbor cell if it is within bounds and has a specific configuration [S1, unofficial].

5. **Extract Vertex Indices:** For each non-ambiguous cell, use `vtstab` to get the indices of the 3 vertices associated with that cell configuration. Map these local indices to global 1D indices (`sel_pidx1`) [S1, unofficial].

6. **Perform 1D Search for Vertices on Grid Edges:** For vertices lying on grid edges parallel to axes, use binary search to find the exact intersection point with the implicit surface. The search range is `[0, 1]` in normalized grid coordinates. The number of iterations is controlled by `BINARY_SEARCH` [S1, unofficial].

7. **Perform 2D Search for Vertices on Grid Faces:** For vertices lying on grid faces, use a hybrid linear/binary search.
   - First, perform linear steps to narrow down the search range (controlled by `VERTICAL_LINEAR_SEARCH` or `HORIZNTL_LINEAR_SEARCH`).
   - Then, perform binary search within that range (controlled by `VERTICAL_BINARY_SEARCH` or `HORIZNTL_BINARY_SEARCH`) [S1, unofficial].

8. **Compute Vertex Positions:** Combine the results from 1D and 2D searches to determine the final 3D coordinates of each vertex. The coordinates are normalized to the grid space [S1, unofficial].

9. **Compute Normals and QEF:** For each vertex, compute the normal vector using the cross product of the edge direction and the face normal. This normal is used to construct the QEF for vertex optimization [S1, unofficial].
   - Compute vector `vp3` as `epnts` minus `t_ unsqueezed at dim 1` multiplied by `n3` [S1, unofficial].
   - Compute condition `cndt` by summing the dot product of the cross product of `v12` and `(vp3 - pnts1)` with `n3` over dim 1, checking if greater than 0 [S1, unofficial].
   - Update `vp3` using `th.where`: if `cndt.unsqueeze(1)` is true, keep `vp3`, else set `vp3` to `vp12` [S1, unofficial].
   - Compute `lf12` as the minimum of the norm of `(vp0 - epnts)` and the norm of `(vp3 - epnts)` over dim 1 [S1, unofficial].
   - Update `cnn12` by bitwise OR with `((lf12 < lf03) OR bnn03 OR sit12)`, then bitwise AND with NOT `(cnn03 OR sit03)` [S1, unofficial].

10. **Construct Faces:** Concatenate two sets of stacked vertex indices based on `cnn12` condition, then concatenate with existing faces along dim 0 [S1, unofficial].
    - Filter `cndts` using `sitnp` and `sitsp` indices, then concatenate `cndtn` twice and `cndts` four times [S1, unofficial].
    - Swap the second and third columns of faces indexed by `cndts` to ensure consistent winding order [S1, unofficial].
    - Return unique faces with inverse indices [S1, unofficial].

11. **Extract Vertices and Faces:** Call `_get_face()`, then map indices to points and scale/offset to coordinates using `min_coordc` and `siz_coordc` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch as th

class OccupancyDualContouring:
    def __init__(self, device):
        self.device = device
        # Initialize lookup tables as described in the source
        self.vitogi = th.tensor(vitogi, dtype=th.int64, device=device)
        # ... (other tables initialization omitted for brevity, see source [S1, unofficial])

    def extract_mesh(self, imp_func, min_coord=[-0.5, -0.5, -0.5], max_coord=[0.5, 0.5, 0.5], num_grid=128, isolevel=0.0, outside=True, BINARY_SEARCH=15, VERTICAL_RANGE=0.8, VERTICAL_LINEAR_SEARCH=4, VERTICAL_BINARY_SEARCH=11, HORIZNTL_RANGE=0.71, HORIZNTL_LINEAR_SEARCH=3, HORIZNTL_BINARY_SEARCH=12, ERR=3e-7, QEF_REG=0.05):
        GRID = num_grid
        GRID_ = GRID + 1
        GRID2_, GRID3_ = GRID_ ** 2, GRID_ ** 3
        device = self.device

        min_coordc = th.tensor(min_coord, dtype=th.float64, device=device).reshape(1, 3)
        siz_coordc = th.tensor(max_coord, dtype=th.float64, device=device).reshape(1, 3) - min_coordc

        # Helper functions
        pidx1_to_pidx3 = lambda pi1: th.remainder(th.stack((th.div(pi1, GRID2_, rounding_mode='floor'), th.div(pi1, GRID_, rounding_mode='floor'), pi1), dim=-1), GRID_)
        pidx3_to_pnts = lambda pi3: (pi3 / GRID).to(th.float64)

        if outside:
            get_occv_ = lambda x: imp_func(x) < isolevel
        else:
            get_occv_ = lambda x: imp_func(x) > isolelevel

        # Compute occupancy grid
        pidx1 = th.arange(GRID3_, dtype=th.int64, device=device)
        occs = get_occv_(pidx3_to_pnts(pidx1_to_pidx3(pidx1))).reshape(GRID_, GRID_, GRID_)

        # Compute cell configuration values
        vals = th.zeros((GRID, GRID, GRID), dtype=th.uint8, device=device)
        vals += occs[1:, 1:, 1:]; vals <<= 1
        vals += occs[1:, 1:, :-1]; vals <<= 1
        vals += occs[1:, :-1, 1:]; vals <<= 1
        vals += occs[1:, :-1, :-1]; vals <<= 1
        vals += occs[:-1, 1:, 1:]; vals <<= 1
        vals += occs[:-1, 1:, :-1]; vals <<= 1
        vals += occs[:-1, :-1, 1:]; vals <<= 1
        vals += occs[:-1, :-1, :-1]

        # Identify ambiguous cells and resolve
        # ... (Ambiguity resolution logic using ambtab)

        # Extract vertex indices
        # ... (Using vtstab to get vertex indices)

        # Perform 1D and 2D searches
        # ... (Binary and hybrid searches for vertex positions)

        # Compute normals and QEF
        # ... (Normal computation and QEF setup)

        # Construct faces
        # ... (Face construction using cstrtab and cpnttab)

        # Extract vertices and faces
        # ... (Final mapping and scaling)
        return vertices, faces
```

**Parameters:**
- `GRID`: The resolution of the voxel grid, derived from `num_grid`. The grid indices range from 0 to `GRID+1` [S1, unofficial].
- `isolevel`: The threshold value for the implicit function [S1, unofficial].
- `BINARY_SEARCH`: Number of iterations for the 1D binary search along grid axes (default 15) [S1, unofficial].
- `VERTICAL_RANGE`: Search range for vertical 2D searches (default 0.8) [S1, unofficial].
- `VERTICAL_LINEAR_SEARCH`: Number of linear steps in vertical 2D search (default 4) [S1, unofficial].
- `VERTICAL_BINARY_SEARCH`: Number of binary search iterations in vertical 2D search (default 11) [S1, unofficial].
- `HORIZNTL_RANGE`: Search range for horizontal 2D searches (default 0.71) [S1, unofficial].
- `HORIZNTL_LINEAR_SEARCH`: Number of linear steps in horizontal 2D search (default 3) [S1, unofficial].
- `HORIZNTL_BINARY_SEARCH`: Number of binary search iterations in horizontal 2D search (default 12) [S1, unofficial].
- `ERR`: Convergence threshold for edge length in 2D search (default 3e-7). If edge length < `ERR`, it is normalized by 1.0 to avoid division by zero [S1, unofficial].
- `QEF_REG`: QEF regularization weight (default 0.05) [S1, unofficial].
- `GRID`: A constant used to normalize point values (`val_pnts / GRID`) during coordinate conversion [S1, unofficial].
- `min_coordc`: The minimum coordinate offset applied to scaled points to determine final vertex positions [S1, unofficial].
- `siz_coordc`: The scaling factor applied to normalized points to determine final vertex positions [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero in edge length normalization is handled by checking against `ERR` [S1, unofficial].
- Ambiguous cells must be resolved to ensure topological correctness [S1, unofficial].
- The method assumes the implicit function is evaluated on a regular grid [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method involves evaluating the implicit function on a `GRID^3` grid, followed by cell configuration computation, ambiguity resolution, and search operations for each cell. The search operations (1D and 2D) are logarithmic or linear depending on the parameters. Overall complexity is dominated by the implicit function evaluation and the search steps, which are parallelizable on GPU.
- **Memory Complexity:** Requires storage for the occupancy grid, lookup tables, and intermediate vertex/face data. The lookup tables are constant size, while the grid and mesh data scale with `GRID^3` and the number of surface intersections, respectively.
- **Measured Performance:** Implemented in PyTorch for GPU acceleration [S1, unofficial]. Performance depends on `num_grid`, `BINARY_SEARCH`, and other search parameters.
- **Strengths:** Handles complex topology, designed for GPU parallelization, avoids distance information [S1, unofficial].
- **Limitations:** Requires precomputed lookup tables, sensitive to `ERR` and search parameters, may produce artifacts if ambiguities are not resolved correctly [S1, unofficial].
- **When to Use:** Suitable for high-fidelity surface extraction from binary occupancy grids on GPU, especially when distance information is unavailable or unreliable [S1, unofficial].
- **When Not to Use:** Not suitable for CPU-only environments due to heavy reliance on GPU-optimized lookup tables and parallel operations [S1, unofficial].

## Background
Occupancy Dual Contouring is implemented in the PyTorch library `occupancy_dual_contouring.py`, used with implicit functions returning `torch.float64` tensors [S1, unofficial]. It is part of the ODC (Occupancy Dual Contouring) project by KAIST Visual AI Group [S1, unofficial]. The method builds on dual contouring principles, adapting them for occupancy grids and GPU implementation [S1, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
