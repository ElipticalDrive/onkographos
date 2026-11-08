# Trace-based SDF Rendering

Also called: voxel SDF tracing

## Overview
Trace-based SDF Rendering is a method for rendering sparse voxel models by tracing rays through a Signed Distance Field (SDF) to determine hit depth and voxel ID, then retrieving color from the hit voxel's spherical harmonic (SH) coefficients [S1, unofficial]. It differs from standard raymarching by explicitly returning discrete voxel identifiers alongside depth, enabling direct attribute lookup rather than continuous volume integration [S1, unofficial].

## How it works
**Inputs:**
- A sparse voxel model containing geometry (SDF) and color data (SH coefficients) [S1, unofficial].
- A list of camera views [S1, unofficial].
- Parameters: `near_plane` (default 0.02), `vox_geo_mode`, `density_mode` [S1, unofficial].

**Outputs:**
- Rendered color tensor [S1, unofficial].
- Hit depth and hit voxel ID tensors [S1, unofficial].

**Data Structures:**
- `SparseVoxelModel`: Stores voxel geometry and SH coefficients.
- `hit_vox_id`: Tensor of integer voxel IDs. Value `-1` indicates background or non-intersected rays [S1, unofficial].
- `hit_depth`: Tensor of float distances to the surface [S1, unofficial].
- `SH0`: Base color coefficients (degree 0) indexed by voxel ID [S1, unofficial].

**Algorithm:**
1.  For each view in the views list, invoke the voxel model's rendering function with the SDF tracing mode: `render_trace_sdf(view)` [S1, unofficial].
2.  The internal tracer advances the ray using sphere tracing principles (see [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md)) to find the intersection with the voxel geometry [S1, unofficial].
3.  The function returns `hit_depth` and `hit_vox_id` tensors [S1, unofficial].
4.  Retrieve the base color tensor from the voxel model's `SH0` coefficients, indexed by `hit_vox_id` [S1, unofficial].
5.  Rearrange the color tensor axes so the last axis (color channels) is moved to the front: `rendering = voxel_model._sh0[hit_vox_id].moveaxis(-1, 0)` [S1, unofficial].
6.  Mask the rendering by multiplying with a boolean mask where `hit_vox_id` is not `-1`: `rendering *= (hit_vox_id != -1)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_trace_sdf(voxel_model, views):
    """
    Renders views using trace-based SDF method.
    Args:
        voxel_model: SparseVoxelModel instance with _sh0 attribute.
        views: List of camera view objects.
    Returns:
        rendered_images: List of rendered color tensors.
    """
    rendered_images = []
    
    for view in views:
        # Step 1: Trace ray and get hit info
        # The implementation details of the trace are handled by the voxel model
        # based on the parent method's sphere tracing logic.
        hit_depth, hit_vox_id = voxel_model.render_trace_sdf(view)
        
        # Step 2: Retrieve color from SH0 coefficients
        # Indexing with hit_vox_id retrieves colors for all pixels
        # moveaxis moves the color channel axis from last to front
        color_tensor = voxel_model._sh0[hit_vox_id].moveaxis(-1, 0)
        
        # Step 3: Mask background
        # hit_vox_id == -1 represents background/non-intersected rays
        mask = (hit_vox_id != -1)
        
        # Apply mask to color tensor
        # Note: Broadcasting mask to match color tensor dimensions
        rendered_images.append(color_tensor * mask)
        
    return rendered_images
```

**Parameters:**
- `near_plane`: Default 0.02. Used in visibility checks [S1, unofficial].
- `vox_geo_mode`: Configuration for voxel geometry mode [S1, unofficial].
- `density_mode`: Configuration for density mode [S1, unofficial].

**Edge Cases:**
- `hit_vox_id == -1`: Represents background. Must be masked out to avoid invalid color data [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of sphere tracing steps per ray, which is determined by the SDF smoothness and scene complexity.
- **Memory Complexity:** Proportional to the number of voxels and the size of SH coefficients stored in the sparse voxel model [S1, unofficial].
- **Strengths:** Direct access to voxel attributes (color) via ID lookup; efficient for sparse data [S1, unofficial].
- **Limitations:** Requires pre-computed SDF and voxel structure; limited to discrete voxel attributes unless extended [S1, unofficial].
- **When to use:** When rendering sparse voxel models with explicit geometry and color data stored per voxel [S1, unofficial].
- **When not to use:** For continuous volume rendering without discrete voxel structure [S1, unofficial].

## Background
The method is implemented in the `svraster-win10` repository by leonhardrobin [S1, unofficial]. It is part of a pipeline that includes TSDF fusion and adaptive voxel subdivision for 3D reconstruction and rendering [S1, unofficial].

## Sources
[S1] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
