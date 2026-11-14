# Sparse Point Cloud Tracer

Also called: Packed SPC Tracer

## Overview
The Sparse Point Cloud (SPC) Tracer is a ray-intersection method for rendering structured point clouds by treating points as cell centers within a sparse voxel hierarchy. It solves the problem of efficiently determining the visible surface of a point cloud by intersecting rays against the sparse grid structure and querying a neural field for color, without performing neural operations during the traversal phase itself. It differs from sibling methods like [Octree Traversal](1587-octree-traversal.md) and [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by operating directly on point indices derived from the grid's bottom-level acceleration structure (BLAS) and returning the color of the intersected cell via an external neural field query, rather than accumulating volume density or traversing explicit voxel nodes for shading.

## How it works
**Inputs and Outputs**
- **Inputs**: A ray buffer (origin, direction), a Neural Field (`nef`) containing the sparse grid (`nef.grid.blas`), and optional parameters `bg_color` and `lod_idx`.
- **Outputs**: A `RenderBuffer` dataclass containing tensors for `depth` (N, 1), `hit` (N, bool), `rgb` (N, 3), and `alpha` (N, 1).

**Data Structures**
- `RaytraceResults`: A container holding three tensors:
  - `ridx`: Ray indices (integers).
  - `pidx`: Point indices (integers).
  - `depth`: Intersection depths (floats).
- `RenderBuffer`: A dataclass with fields:
  - `depth`: Tensor of shape `(N, 1)`.
  - `hit`: Tensor of shape `(N,)`, boolean.
  - `rgb`: Tensor of shape `(N, 3)`.
  - `alpha`: Tensor of shape `(N, 1)`.

**Parameters**
- `bg_color`: Background color tuple, default `(0.0, 0.0, 0.0)` [S1, unofficial].
- `lod_idx`: Level of Detail index for ray sampling. If `None`, defaults to `nef.grid.blas.max_level` [S1, unofficial].

**Steps**
1. **Initialize LOD**: If `lod_idx` is `None`, set `lod_idx = nef.grid.blas.max_level` [S1, unofficial].
2. **Ray-Grid Intersection**: Call `nef.grid.blas.raytrace(rays, lod_idx, with_exit=False)` to compute intersections against the SPC grid. This returns a `RaytraceResults` object containing `ridx`, `pidx`, and `depth` [S1, unofficial].
3. **Identify First Hits**: Use `spc_render.mark_pack_boundaries(ridx)` to generate a boolean mask identifying the first hit for each ray. Extract `first_hits_point`, `first_hits_ray`, and `first_hits_depth` using this mask [S1, unofficial].
4. **Query Color**: Query the neural field `nef` using `first_hits_point.long()` and `channels='rgb'` to retrieve the color tensor for the intersected cells [S1, unofficial].
5. **Initialize Buffers**: Create output tensors of size `N` (number of rays):
   - `depth`: Zeros.
   - `alpha`: Ones for rays that hit, zeros otherwise (initially all ones, then masked).
   - `hit`: Boolean zeros.
   - `rgb`: Zeros.
   - `out_alpha`: Zeros [S1, unofficial].
6. **Assign Data**:
   - Set `depth[first_hits_ray.long()] = first_hits_depth`.
   - Set `hit[first_hits_ray.long()] = alpha[..., 0] > 0.0`.
   - Set `rgb[first_hits_ray.long(), :3] = color`.
   - Set `out_alpha[first_hits_ray.long()] = alpha[..., 0]` [S1, unofficial].
7. **Return**: Construct and return a `RenderBuffer` with the populated `depth`, `hit`, `rgb`, and `alpha` tensors [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_point_cloud_tracer(rays, nef, spc_render, bg_color=(0.0, 0.0, 0.0), lod_idx=None):
    """
    Trace rays against a Sparse Point Cloud grid and query neural field for color.
    
    Args:
        rays: Tensor of ray origins and directions.
        nef: Neural Field object containing the sparse grid.
        spc_render: Renderer helper object with mark_pack_boundaries.
        bg_color: Tuple for background color.
        lod_idx: Level of detail index. None uses max_level.
        
    Returns:
        RenderBuffer: Contains depth, hit, rgb, alpha tensors.
    """
    # Step 1: Initialize LOD
    if lod_idx is None:
        lod_idx = nef.grid.blas.max_level

    # Step 2: Ray-Grid Intersection
    # Returns RaytraceResults with ridx, pidx, depth
    results = nef.grid.blas.raytrace(rays, lod_idx, with_exit=False)
    ridx = results.ridx
    pidx = results.pidx
    depth = results.depth

    # Step 3: Identify First Hits
    # mark_pack_boundaries creates a mask for the first hit per ray
    first_hit_mask = spc_render.mark_pack_boundaries(ridx)
    
    # Extract first hits using the mask
    # Note: Implementation details of extracting specific indices from masked arrays
    # are assumed to be handled by standard tensor operations.
    first_hits_ray = ridx[first_hit_mask]
    first_hits_point = pidx[first_hit_mask]
    first_hits_depth = depth[first_hit_mask]

    # Step 4: Query Color
    # Query neural field for RGB at the hit points
    color = nef(first_hits_point.long(), channels='rgb')

    # Step 5: Initialize Output Buffers
    N = rays.shape[0] # Assuming rays is (N, ...)
    out_depth = torch.zeros((N, 1))
    out_alpha = torch.zeros((N, 1))
    out_hit = torch.zeros((N,), dtype=torch.bool)
    out_rgb = torch.zeros((N, 3))
    
    # Initialize alpha for hit rays to 1.0 before masking
    # The source sets alpha to ones for hit rays, then assigns out_alpha
    alpha_hit = torch.ones((N, 1))

    # Step 6: Assign Hit Data
    # Map first hits back to ray indices
    ray_indices = first_hits_ray.long()
    
    out_depth[ray_indices] = first_hits_depth.unsqueeze(1)
    out_hit[ray_indices] = alpha_hit[ray_indices, 0] > 0.0
    out_rgb[ray_indices, :3] = color
    out_alpha[ray_indices] = alpha_hit[ray_indices]

    # Step 7: Return RenderBuffer
    return RenderBuffer(
        depth=out_depth,
        hit=out_hit,
        rgb=out_rgb,
        alpha=out_alpha
    )
```

**Edge Cases and Pitfalls**
- **No Hit**: Rays that do not intersect the SPC grid will have `hit` set to `False` and `rgb`/`depth` as initialized zeros. The background color `bg_color` is not explicitly blended in the provided steps; it is assumed to be handled by the caller or defaulting to black [S1, unofficial].
- **Precision**: Depth values are stored in the `depth` tensor. Precision depends on the underlying grid's coordinate system and the raytrace implementation [S1, unofficial].
- **Degenerate Input**: If `rays` is empty, the output tensors will be empty. If `nef.grid.blas` is not initialized, the raytrace call will fail [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by the `raytrace` call against the sparse grid BLAS and the neural field query. The traversal is efficient due to the sparse structure, but the neural query adds overhead per hit.
- **Memory Complexity**: Requires storage for the sparse grid, the neural field weights, and output buffers proportional to the number of rays.
- **Strengths**: Efficient for rendering large-scale point clouds by leveraging sparse grid acceleration. Decouples traversal from shading, allowing flexible neural field integration.
- **Limitations**: Returns only the color of the first hit; does not perform volume accumulation or transparency blending within the tracer itself. Relies on an external neural field for color, which may be computationally expensive.
- **When to Use**: Suitable for real-time or interactive visualization of structured point clouds where neural field-based color is desired.
- **When Not to Use**: Not suitable for volumetric rendering requiring transparency, depth-of-field, or complex lighting effects within the tracer.

## Background
The Sparse Point Cloud Tracer is part of the NVIDIA Kaolin-Wisp framework, introduced in the SIGGRAPH 2022 demo. It is designed to visualize Structured Point Cloud (SPC) content efficiently. The method leverages the sparse voxel hierarchy to accelerate ray-point intersections and integrates with neural fields for high-fidelity rendering.

## Sources
[S1] NVIDIAGameWorks/kaolin-wisp, https://github.com/NVIDIAGameWorks/kaolin-wisp, fetched 2026-10-26 (unofficial)
