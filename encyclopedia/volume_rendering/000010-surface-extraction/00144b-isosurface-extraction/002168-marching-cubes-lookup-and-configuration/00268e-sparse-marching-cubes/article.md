# Sparse Marching Cubes

Also called: Sparse Marching Cubes from Points

## Overview
Sparse Marching Cubes generates a triangle mesh from a sparse set of 3D points with scalar values (e.g., Signed Distance Fields), avoiding the memory overhead of dense 3D grids [S1, unofficial]. It differs from standard Marching Cubes by dynamically gathering corner values for active voxels via lookup, rather than iterating over a fixed dense volume [S1, unofficial]. It builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md).

## How it works
**Inputs:**
1. `coords`: Integer tensor of shape $[N, 3]$ containing the $(x, y, z)$ coordinates of the active voxels (or points defining the scalar field).
2. `corners` OR `point_values`:
   - Option A: `corners` Float tensor of shape $[N, 8]$ containing pre-computed scalar values at the 8 corners of each voxel.
   - Option B: `point_values` Float tensor of shape $[N]$ containing scalar values at the coordinates in `coords`.
3. `iso`: Float value representing the iso-level to contour at.
4. `default_value`: Float value assumed for corner points not present in the input point set (typically $1.0$ for SDFs, indicating outside) [S1, unofficial].
5. `ensure_consistency`: Boolean flag; if True, averages corner values at shared locations to ensure topological consistency across adjacent voxels [S1, unofficial].

**Outputs:**
1. `verts`: Float tensor of shape $[V, 3]$ with vertex positions.
2. `tris` (or `faces`): Int tensor of shape $[T, 3]$ with triangle indices [S1, unofficial].

**Data Structures:**
- The method operates on a list of active voxels defined by integer coordinates.
- Corner values are gathered dynamically if only point values are provided. A hash map or sorted lookup is implied to find the scalar value for a specific integer coordinate $(x,y,z)$ [G]. If the coordinate is not found, `default_value` is used.

**Steps:**
1. **Gather Corner Values:** For each active voxel at `coords[i]`, determine the integer coordinates of its 8 corners. The corners are located at `coords[i] + offset`, where `offset` ranges over the 8 binary combinations of $(0,0,0)$ to $(1,1,1)$.
2. **Lookup Scalars:** For each corner coordinate, look up the scalar value in the input data.
   - If `corners` is provided, index directly into the $[N, 8]$ tensor.
   - If `point_values` is provided, search for the corner coordinate in `coords`. If found, use the corresponding `point_values` entry. If not found, use `default_value` [S1, unofficial].
3. **Compute Case Index:** Calculate the marching cubes case index $C$ for each voxel based on the 8 corner values $v_0 \dots v_7$ relative to `iso`. The standard bit-masking formula is used [S1, unofficial]:
   $$
   C = \sum_{k=0}^{7} b_k 2^k
   $$
   where $b_k = 1$ if $v_k < \text{iso}$, and $0$ otherwise. The order of vertices $v_0 \dots v_7$ must match the standard Marching Cubes convention (e.g., [Lorensen and Cline, 1987] or the parent article's configuration).
4. **Ensure Consistency (Optional):** If `ensure_consistency` is True, identify shared corners between adjacent active voxels. Average the scalar values at these shared corners to ensure that the iso-surface intersection is consistent across voxel boundaries [S1, unofficial]. This step prevents gaps or overlaps in the mesh.
5. **Triangulation:** For each voxel, use the case index $C$ to look up the corresponding triangle configuration from the Marching Cubes lookup table (provided by [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md)).
6. **Vertex Interpolation:** For each edge intersected by the iso-surface, interpolate the vertex position. If edge endpoints have values $v_a, v_b$ at positions $p_a, p_b$, the intersection point $p$ is:
   $$
   p = p_a + \frac{\text{iso} - v_a}{v_b - v_a} (p_b - p_a)
   $$
7. **Mesh Assembly:** Collect all generated vertices and triangles. Handle vertex sharing by using a global vertex index map to merge identical interpolated positions [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sparse_marching_cubes_from_points(coords, point_values, iso, default_value=1.0, ensure_consistency=False):
    """
    Performs sparse marching cubes from a sparse set of points.
    
    Args:
        coords: Tensor [N, 3], integer coordinates of points.
        point_values: Tensor [N], scalar values at coords.
        iso: Float, iso-level.
        default_value: Float, value for missing corners.
        ensure_consistency: Bool, average shared corner values.
        
    Returns:
        verts: Tensor [V, 3]
        tris: Tensor [T, 3]
    """
    # 1. Ensure CUDA tensors
    if not coords.is_cuda: coords = coords.cuda()
    if not point_values.is_cuda: point_values = point_values.cuda()
    coords = coords.int().contiguous()
    point_values = point_values.float().contiguous()
    
    # 2. Gather corner values for all active voxels
    # Offsets for the 8 corners of a voxel at (x,y,z)
    offsets = torch.tensor([
        [0, 0, 0], [1, 0, 0], [0, 1, 0], [1, 1, 0],
        [0, 0, 1], [1, 0, 1], [0, 1, 1], [1, 1, 1]
    ], dtype=torch.int32, device=coords.device)
    
    # Expand coords to [N, 8, 3] and add offsets
    voxel_corners = coords.unsqueeze(1) + offsets.unsqueeze(0) # [N, 8, 3]
    
    # Flatten to [N*8, 3] to lookup values
    flat_corners = voxel_corners.view(-1, 3)
    
    # Lookup values: Create a hash map or use torch.searchsorted if sorted
    # For implementation, assume a helper `lookup_values` that maps coordinates to values
    # If coordinate not found, return default_value
    flat_values = lookup_values(flat_corners, coords, point_values, default_value) # [N*8]
    
    # Reshape to [N, 8]
    corners = flat_values.view(-1, 8)
    
    # 3. Ensure Consistency (Optional)
    if ensure_consistency:
        # Identify shared corners between adjacent voxels
        # Average their values
        corners = average_shared_corners(corners, voxel_corners) # [N, 8]
        
    # 4. Compute Case Indices
    cube_indices = get_cube_indices(corners, iso)
    
    # 5. Call Backend Implementation
    # The backend handles triangulation, interpolation, and mesh assembly
    verts, tris = _backend.sparse_marching_cubes(coords, corners, iso, ensure_consistency)
    
    return verts, tris

def get_cube_indices(corners, iso):
    """
    Calculates the marching cubes case index for each voxel.
    """
    v = corners.float()
    # Bitmask: bit k is 1 if v[k] < iso
    cube_indices = (v < iso).int()
    cube_indices = (
        cube_indices[:, 0] * 1 |
        cube_indices[:, 1] * 2 |
        cube_indices[:, 2] * 4 |
        cube_indices[:, 3] * 8 |
        cube_indices[:, 4] * 16 |
        cube_indices[:, 5] * 32 |
        cube_indices[:, 6] * 64 |
        cube_indices[:, 7] * 128
    ).int()
    return cube_indices

def lookup_values(flat_corners, coords, point_values, default_value):
    """
    [G] Helper to map integer coordinates to scalar values.
    In a real implementation, this uses a hash map or sorted search.
    """
    # Placeholder for the actual lookup logic which is not fully detailed in sources
    # but is required for the method to work.
    # Returns a tensor of shape [N*8]
    pass
```

## Subtopics

### [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)
Converts the resulting mesh to PLY format.

### [CPU Immediate Mode Marching Cubes](2463-cpu-immediate-mode-marching-cubes.md)
CPU-based alternative for non-GPU environments.

### [Marching Cubes Triangulation](3682-marching-cubes-triangulation.md)
Details the triangulation logic used by the backend.

### [Simplified Marching Cubes](3759-simplified-marching-cubes.md)
A variant with fewer cases.

### [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md)
Precomputes configurations for speed.

### [Marching Cubes with Narrow-Band Filtering](3826-marching-cubes-with-narrow-band-filtering.md)
Filters noise in SDFs.

### [Modified Marching Cubes with Triangle Decimation](3903-modified-marching-cubes-with-triangle-decimation.md)
Reduces triangle count.

### [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md)
General GPU optimization techniques.

### [Marching Cubes Vertex List Generation](4447-marching-cubes-vertex-list-generation.md)
Generates vertex lists.

### [Lookup Table and Configuration](4978-lookup-table-and-configuration.md)
Defines the case indices.

### [Marching Cubes with Disjoint Patch Decomposition](5004-marching-cubes-with-disjoint-patch-decomposition.md)
Decomposes the volume.

### [Transvoxel](6277-transvoxel.md)
Multi-resolution surface extraction.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of active voxels $N$, $O(N)$, assuming $O(1)$ lookup for corner values [S1, unofficial].
- **Memory Complexity:** Proportional to the number of active voxels and the resulting mesh size. Significantly lower than dense Marching Cubes for sparse fields [S1, unofficial].
- **Performance:** Accelerated by CUDA, processing thousands of voxels simultaneously [S1, unofficial].
- **Strengths:** Efficient for sparse SDF representations; avoids allocating memory for empty space [S1, unofficial].
- **Limitations:** Requires efficient lookup for corner values; `ensure_consistency` adds overhead for averaging shared corners [S1, unofficial].
- **When to use:** When the scalar field is defined by a sparse set of points (e.g., point cloud SDFs) and a dense grid is impractical [S1, unofficial].
- **When not to use:** When the field is dense or the lookup overhead dominates the computation [S1, unofficial].

## Background
Sparse Marching Cubes was introduced in the `spcumc` library by Hwanhuh [S1, unofficial]. It leverages PyTorch and CUDA for high-performance mesh generation from sparse data. The method adapts the classic Marching Cubes algorithm to operate on explicit voxel lists rather than dense arrays [S1, unofficial].

## Sources
[S1] hwanhuh/spcumc, https://github.com/hwanhuh/spcumc, fetched 2026-10-27 (unofficial)
