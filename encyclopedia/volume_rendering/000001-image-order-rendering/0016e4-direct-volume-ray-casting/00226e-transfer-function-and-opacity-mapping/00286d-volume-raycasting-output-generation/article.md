# Volume Raycasting Output Generation

Also called: Final Compositing and Output, RenderOutput Construction

## Overview
This method constructs the final pixel output from a sampled volume value, handling color space conversion, opacity modulation, and depth calculation. It differs from sibling methods by explicitly managing the transition from sampled data to physical colorspace for compositing and computing depth via perspective division of NDC coordinates rather than using raw Z values.

## How it works
**Inputs:**
- `the_coord`: Texture coordinate of the sample.
- `the_value`: Sampled volume value (vec4<f32>).
- `sizef`: Volume dimensions (vec3<f32>).
- `u_wobject.world_transform`: World transformation matrix.
- `u_stdinfo.projection_transform`: Projection matrix.
- `u_stdinfo.cam_transform`: Camera transformation matrix.
- `u_material.opacity`: Material opacity scalar.
- `colorspace`: String, either 'srgb' or other.

**Outputs:**
- `RenderOutput` struct containing:
  - `color`: vec4<f32> (RGBA).
  - `coord`: vec3<f32> (texture coordinate).
  - `depth`: f32 (NDC depth).

**Data Structures:**
- `RenderOutput`:
  - `color`: vec4<f32>
  - `coord`: vec3<f32>
  - `depth`: f32

**Steps:**
1. **Colormapping:** Convert `the_value` to color using `sampled_value_to_color(the_value)`. Let this be `color`.
2. **Colorspace Conversion:**
   - If `colorspace == 'srgb'`, convert `color.rgb` to physical colorspace: `physical_color = srgb2physical(color.rgb)`.
   - Otherwise, `physical_color = color.rgb`.
3. **Opacity Calculation:** Compute `opacity = color.a * u_material.opacity`.
4. **Output Color Construction:** `out_color = vec4<f32>(physical_color, opacity)`.
5. **Position Calculation:**
   - `data_pos = the_coord * sizef - vec3<f32>(0.5, 0.5, 0.5)`
   - `world_pos = u_wobject.world_transform * vec4<f32>(data_pos, 1.0)`
   - `ndc_pos = u_stdinfo.projection_transform * u_stdinfo.cam_transform * world_pos`
6. **Depth Calculation:** `depth = ndc_pos.z / ndc_pos.w`
7. **Output Construction:**
   - `out.color = out_color`
   - `out.coord = the_coord`
   - `out.depth = depth`
   - Return `out`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_raycasting_output_generation(
    the_coord: Vec3,
    the_value: Vec4,
    sizef: Vec3,
    u_wobject_world_transform: Mat4,
    u_stdinfo_projection_transform: Mat4,
    u_stdinfo_cam_transform: Mat4,
    u_material_opacity: float,
    colorspace: str
) -> RenderOutput:
    # Step 1: Colormapping
    color = sampled_value_to_color(the_value)
    
    # Step 2: Colorspace Conversion
    if colorspace == 'srgb':
        physical_color = srgb2physical(color.rgb)
    else:
        physical_color = color.rgb
        
    # Step 3: Opacity Calculation
    opacity = color.a * u_material_opacity
    
    # Step 4: Output Color Construction
    out_color = Vec4(physical_color.x, physical_color.y, physical_color.z, opacity)
    
    # Step 5: Position Calculation
    data_pos = the_coord * sizef - Vec3(0.5, 0.5, 0.5)
    world_pos = u_wobject_world_transform * Vec4(data_pos.x, data_pos.y, data_pos.z, 1.0)
    ndc_pos = u_stdinfo_projection_transform * u_stdinfo_cam_transform * world_pos
    
    # Step 6: Depth Calculation
    depth = ndc_pos.z / ndc_pos.w
    
    # Step 7: Output Construction
    out = RenderOutput()
    out.color = out_color
    out.coord = the_coord
    out.depth = depth
    return out
```

**Parameters:**
- `u_material.opacity`: Material property modulating opacity.
- `colorspace`: String determining sRGB conversion.

**Edge Cases:**
- Division by zero in depth calculation if `ndc_pos.w` is zero.
- Invalid `colorspace` string handling.

## Operating characteristics
- **Time Complexity:** O(1) per sample.
- **Memory Complexity:** O(1) additional memory.
- **Strengths:** Explicit handling of color space and depth ensures accurate compositing.
- **Limitations:** Requires access to transformation matrices and material properties.

## Background
This method is part of the volume raycasting pipeline in pygfx, specifically handling the final output generation after sampling and colormapping.

## Sources
[S1] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
