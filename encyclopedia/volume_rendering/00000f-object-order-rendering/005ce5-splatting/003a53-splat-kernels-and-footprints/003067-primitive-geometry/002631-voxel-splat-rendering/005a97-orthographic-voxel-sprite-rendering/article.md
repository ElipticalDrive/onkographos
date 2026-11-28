# Orthographic Voxel Sprite Rendering

Also called: Flat Billboard Voxel Rendering, Volumetric Ortho-Sprite Preview, VoxelOrthoSprite

## Overview
Orthographic Voxel Sprite Rendering projects a 3D voxel model onto a 2D sprite plane using parallel rays to create a flat billboard appearance [S1, unofficial]. It solves the problem of per-eye pixel shimmer in VR by ensuring both eyes march identical rays, rendering the model as a single static image that appears identical from both eyes [S1, unofficial]. Unlike [Image-Space Sheet-Buffered Splatting Pipeline](8961-image-space-sheet-buffered-splatting-pipeline.md), which uses complex sheet buffers, this method uses a single fragment pass with a chunky virtual-pixel grid and outline detection [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Model bounding box (`box_center`, `box_size`), `sigma`, `quad_size`, `ray_dir_local`, `sprite_up_local`, `light_dir`, voxel texture sampler, `get_material_color`, `march_box` [S1, unofficial].
- **Output:** Fragment color (`ALBEDO`) with palette color, per-face lighting, and a one-pixel black outline on misses [S1, unofficial].

**Data Structures:**
- **Bounding Box:** Defined by `box_center` (vec3) and `box_size` (vec3) in voxel units [S1, unofficial].
- **Virtual Pixel Grid:** Defined by `sigma` (pixels per voxel) [S1, unofficial].
- **Coordinate Space:** Godot Y-up voxel space (one unit per voxel), matching the local space of the `VoxelOrthoSprite` node and the DDA in `voxel_storage.gdshaderinc` [S1, unofficial].

**Parameters:**
- `sigma`: Virtual pixels per voxel. Default: 4.0. Higher values produce finer resolution; a value of 1 makes every voxel one chunky pixel [S1, unofficial].
- `quad_size`: Size of the proxy quad in voxel units. The quad is centred on the model's bounding box centre. Default: `vec2(1.0)` [S1, unofficial].
- `light_dir`: Unit vector from the model towards the light, in voxel space. Default: `vec3(0.57735)` [S1, unofficial].

**Steps:**

1.  **Quad Rasterization:**
    Rasterize a camera-facing QuadMesh sized to the model's orthographic footprint plus a margin for the outline [S1, unofficial]. Each fragment knows its position on the quad (`quad_uv`), which represents the position on the unit quad (-0.5..0.5 on each axis), interpolated perspective-correctly [S1, unofficial].

2.  **Coordinate Conversion and Snapping:**
    Convert fragment position to voxel units [S1, unofficial]. Calculate `delta_px` as `1.0 / sigma`, representing the width of one virtual pixel in voxel units [S1, unofficial]. Snap the `quad_uv` offset to the nearest virtual pixel centre:
    $$ offset = \text{round}\left(\frac{quad\_uv \times quad\_size}{delta\_px}\right) \times delta\_px $$
    [S1, unofficial].

3.  **Ray Marching:**
    Start a ray with the shared direction `ray_dir_local` at the snapped position and march through the voxels using `march_box` [S1, unofficial]. Calculate `sprite_right` as `cross(ray_dir_local, sprite_up_local)` [S1, unofficial]. Calculate `box_min` as `box_center - box_size * 0.5` and `box_max` as `box_center + box_size * 0.5` [S1, unofficial]. Calculate `max_dim` as `max(max(box_size.x, box_size.y), box_size.z)` [S1, unofficial]. Calculate `origin` as `box_center + offset.x * sprite_right + offset.y * sprite_up_local - 2.0 * max_dim * ray_dir_local` [S1, unofficial]. Call `march_box(origin, ray_dir_local, box_min, box_max, normal, hit_pos)` [S1, unofficial].

4.  **Lighting and Color:**
    If `material > 0u`, calculate lighting as `max(dot(normal, light_dir), 0.0) * 0.7 + 0.3` [S1, unofficial]. Set `ALBEDO` to `get_material_color(material) * lighting` [S1, unofficial].

5.  **Outline Detection:**
    If the march results in a miss, test the four cardinal neighbour pixels [S1, unofficial]. Trace rays at `offset - vec2(delta_px, 0.0)`, `offset + vec2(delta_px, 0.0)`, `offset + vec2(0.0, delta_px)`, and `offset - vec2(0.0, delta_px)` [S1, unofficial]. If any hit (`material > 0u`), set `ALBEDO` to `vec3(0.0)` [S1, unofficial]. Otherwise, discard the fragment [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_orthographic_voxel_sprite(quad_uv, quad_size, sigma, box_center, box_size, ray_dir_local, sprite_up_local, light_dir, get_material_color, march_box):
    # 1. Calculate virtual pixel parameters
    delta_px = 1.0 / sigma
    
    # 2. Snap quad_uv to virtual pixel grid
    # quad_uv is in range [-0.5, 0.5]
    offset = round((quad_uv * quad_size) / delta_px) * delta_px
    
    # 3. Calculate ray origin and boundaries
    sprite_right = cross(ray_dir_local, sprite_up_local)
    box_min = box_center - box_size * 0.5
    box_max = box_center + box_size * 0.5
    max_dim = max(max(box_size.x, box_size.y), box_size.z)
    
    origin = box_center + offset.x * sprite_right + offset.y * sprite_up_local - 2.0 * max_dim * ray_dir_local
    
    # march_box and get_material_color are provided by the voxel storage system [S1, unofficial]
    normal, hit_pos, material = march_box(origin, ray_dir_local, box_min, box_max)
    
    # 4. Handle hit
    if material > 0:
        lighting = max(dot(normal, light_dir), 0.0) * 0.7 + 0.3
        albedo = get_material_color(material) * lighting
        return albedo
    
    # 5. Handle miss: Outline detection
    neighbors = [
        offset - vec2(delta_px, 0.0),
        offset + vec2(delta_px, 0.0),
        offset + vec2(0.0, delta_px),
        offset - vec2(0.0, delta_px)
    ]
    
    for neighbor_offset in neighbors:
        neighbor_origin = box_center + neighbor_offset.x * sprite_right + neighbor_offset.y * sprite_up_local - 2.0 * max_dim * ray_dir_local
        _, _, neighbor_material = march_box(neighbor_origin, ray_dir_local, box_min, box_max)
        if neighbor_material > 0:
            return vec3(0.0) # Black outline
            
    return None # Discard fragment
```

**Edge cases and pitfalls:**
- Using screen position for the grid causes per-eye pixel shimmer in VR because each eye sees the same card from a slightly different place [S1, unofficial].
- The ray direction and quad orientation come from the HEAD position (midpoint between eyes), not the per-eye position, so both eyes march identical rays, ensuring the sprite reads as a flat billboard [S1, unofficial].

## Subtopics

### [Square-based voxel rendering](19692-square-based-voxel-rendering.md)
Square-based voxel rendering is a technique for small voxel sets that maps 3D voxels to 2D squares (tiles) on the screen to achieve real-time performance with visual compromises, avoiding the GPU load of dense polygon sets while maintaining voxel destructibility. It differs from its parent, Orthographic Voxel Sprite Rendering, by projecting the voxel as a flat 2D square rather than a full 3D sprite billboard, sacrificing volumetric depth cues for rendering speed and simplicity.

## Operating characteristics
- **Performance:** Single fragment pass makes it efficient for preview purposes [S1, unofficial].
- **Scaling:** Performance scales with `sigma` (higher values produce finer resolution) and voxel density [S1, unofficial].
- **Limitations:** Outline detection adds up to 4 additional ray marches per miss [S1, unofficial].
- **When to use:** VR previews, low-poly voxel art, performance-critical applications where visual fidelity is secondary to consistency [S1, unofficial].
- **When not to use:** High-resolution rendering, perspective-correct detailed views, scenes with complex lighting requirements beyond simple diffuse [S1, unofficial].
- **Time and memory complexity:** Not given in the sources.

## Background
- **Origin:** Developed for Godot engine-based voxel editors and viewers as the `VoxelOrthoSprite` node, utilizing `voxel_storage.gdshaderinc` for DDA raymarching [S1, unofficial].
- **History and development:** Not given in the sources.
- **Where it is used:** Godot engine voxel editors and viewers [S1, unofficial].

## Sources
[S1] Voxel2Pixel, https://github.com/BenMcLean/Voxel2Pixel, fetched 2026-10-29 (unofficial)
