# Volume Ones Mask

Also called: Volume Mask, volume onesmask

## Overview
`volume onesmask` creates a binary voxel grid where all voxels are set to 1 within a region bounded by a specified surface, and 0 outside. It solves the problem of generating a spatial mask from geometric surfaces for use in volume rendering or segmentation. It differs from `volume mask` [S1] by outputting a uniform value of 1 rather than copying data from an existing map, though it shares the same underlying masking logic and parameters.

## How it works
**Inputs:**
- A surface specification (`surface-spec`) defining the boundary geometry [S1].
- Optional grid parameters: `onGrid` (reference map), `border`, `spacing`, `valueType` [S1].
- Masking options: `fullMap`, `pad`, `extend`, `slab`, `invertMask`, `axis`, `sandwich`, `fillOverlap` [S1].

**Outputs:**
- A new map (voxel grid) with values set to 1 inside the masked region and 0 outside [S1].

**Data Structures:**
- The output is a 3D grid of voxels. The layout is defined by `spacing` (grid resolution) and `border` or `onGrid` (extent).
- `valueType` determines the storage format: `int8`, `int16`, `int32`, `uint8`, `uint16`, `uint32`, `float32`, or `float64` [S1]. Default is `int8` if `onGrid` is not specified, otherwise it matches the `onGrid` map type [S1].

**Parameters:**
- `fullMap` (Boolean): If `true`, output dimensions match the original map (if `onGrid` is used). If `false` (default), the grid is the smallest possible bounding box enclosing the surface [S1].
- `pad` (Distance): Moves the surface along normals by a distance (Å or nm) before masking. Positive expands, negative shrinks [S1].
- `extend` (Integer N): Moves the surface outward by N voxels (grid units along X, Y, Z) before masking. This avoids self-intersection issues caused by `pad` [S1].
- `slab` (Width or d1,d2): Extracts a slab of data around the surface. Two displaced surfaces are computed.
    - Single value `width`: Offsets surfaces by $\pm \frac{1}{2} \times \text{width}$ along normals [S1].
    - Two values `d1,d2`: Offsets surfaces independently by `d1` and `d2` [S1].
- `invertMask` (Boolean): If `true`, inverts the result (1 outside, 0 inside) [S1].
- `axis` (Vector): Projection axis for determining "inside" vs "outside" relative to holes. Default is y [S1].
- `sandwich` (Boolean): If `true`, includes only voxels strictly between paired surface layers. If `false`, includes voxels projected along the axis beyond a single layer [S1].
- `fillOverlap` (Boolean): If `true`, retains the union of values for intersecting/nested surfaces. If `false`, returns only the shell between nested surfaces (e.g., between two concentric spheres) [S1].
- `onGrid` (Map ID): Places output on the grid of an existing map. `fullMap` defaults to `true` here [S1].
- `border` (Distance B): If `onGrid` is omitted, sets the distance from the bounding surface to the output map edge in $\pm X, \pm Y, \pm Z$ [S1].
- `spacing` (S or Sx,Sy,Sz): Grid spacing in physical units. Default is 1% of the maximum X, Y, Z dimensions of the bounding surfaces. Single value applies to all axes; three comma-separated values apply to X, Y, Z respectively [S1].

**Algorithm Steps:**
1. **Parse Surface:** Identify the geometric surface(s) from `surface-spec` [S1].
2. **Determine Grid Extent:**
    - If `onGrid` is provided:
        - Use the grid of the referenced map.
        - If `fullMap` is `true`, use the full extents of the referenced map.
        - If `fullMap` is `false`, compute the smallest bounding box enclosing the surface within that grid [S1].
    - If `onGrid` is not provided:
        - Compute the bounding box of the surface.
        - Expand the box by `border` B in all six directions ($\pm X, \pm Y, \pm Z$) [S1].
3. **Determine Grid Resolution:**
    - Use `spacing` if provided.
    - If `spacing` is not provided, calculate default spacing as $0.01 \times \max(\text{dim}_X, \text{dim}_Y, \text{dim}_Z)$ of the bounding surfaces [S1].
4. **Apply Surface Modifications:**
    - If `pad` is specified, displace the surface normals by the `pad` distance [S1].
    - If `extend` is specified, displace the surface outward by `extend` voxels along grid axes [S1].
    - If `slab` is specified, compute two additional surfaces displaced from the original.
        - If single `width`: offsets are $\pm \frac{1}{2} \times \text{width}$ [S1].
        - If `d1,d2`: offsets are `d1` and `d2` [S1].
5. **Determine Masked Region:**
    - Define the region based on `axis`, `sandwich`, and `fillOverlap`:
        - Project along `axis` (default y) to resolve holes [S1].
        - If `sandwich` is `true`, select voxels strictly between the paired displaced surfaces (if `slab` is used) or between the surface and its projection [S1].
        - If `fillOverlap` is `true` for nested surfaces, take the union of the masked regions of each surface [S1].
        - If `fillOverlap` is `false` for nested surfaces, take the difference (shell) [S1].
6. **Invert (Optional):**
    - If `invertMask` is `true`, swap the inside/outside classification [S1].
7. **Initialize Grid:**
    - Create a new grid with dimensions derived from steps 2 and 3.
    - Set all voxel values to 0 [G].
8. **Set Mask Values:**
    - For each voxel in the grid, check if it lies within the computed masked region.
    - If yes, set the voxel value to 1.
    - If no, leave it as 0 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_ones_mask(surface, options):
    # 1. Parse surface geometry
    surface_geom = parse_surface(surface)
    
    # 2. Determine grid parameters
    if options.onGrid:
        ref_grid = get_grid(options.onGrid)
        if options.fullMap:
            grid_dims = ref_grid.dims
            grid_origin = ref_grid.origin
            grid_spacing = ref_grid.spacing
        else:
            # Compute smallest bounding box for surface within ref_grid context
            bbox = compute_bounding_box(surface_geom)
            grid_dims, grid_origin, grid_spacing = fit_to_ref_grid(bbox, ref_grid)
    else:
        bbox = compute_bounding_box(surface_geom)
        # Expand by border
        if options.border:
            bbox.expand(options.border)
        # Determine spacing
        if options.spacing:
            grid_spacing = parse_spacing(options.spacing)
        else:
            max_dim = max(bbox.max - bbox.min)
            grid_spacing = 0.01 * max_dim
        grid_dims, grid_origin = compute_grid_from_bbox(bbox, grid_spacing)
        
    # 3. Modify surface for masking
    modified_surface = surface_geom
    if options.pad:
        modified_surface = displace_normals(modified_surface, options.pad)
    if options.extend:
        # Extend by N voxels along grid axes
        modified_surface = extend_grid_surface(modified_surface, options.extend, grid_spacing)
        
    slab_surfaces = None
    if options.slab:
        width = options.slab
        if ',' in width:
            d1, d2 = parse_comma_values(width)
            slab_surfaces = [displace_normals(modified_surface, d1), 
                             displace_normals(modified_surface, d2)]
        else:
            half_w = 0.5 * width
            slab_surfaces = [displace_normals(modified_surface, -half_w), 
                             displace_normals(modified_surface, half_w)]
            
    # 4. Determine voxel inclusion
    # Initialize output grid with 0s
    output_grid = create_grid(grid_dims, grid_origin, grid_spacing, options.valueType)
    
    axis = options.axis if options.axis else [0, 1, 0] # Default y
    
    for voxel in output_grid:
        inside = is_inside_surface(voxel, modified_surface, axis, options.sandwich)
        
        if options.slab:
            # Check if voxel is between slab_surfaces
            if options.sandwich:
                inside = is_between_surfaces(voxel, slab_surfaces, axis)
            else:
                # Project beyond single layer
                inside = is_projected_inside(voxel, slab_surfaces, axis)
                
        if options.fillOverlap and len(surface) > 1:
            # Handle nested surfaces: union
            # This requires checking each surface individually and ORing results
            # Simplified for single surface logic in basic implementation
            pass 
            
        if options.invertMask:
            inside = not inside
            
        if inside:
            output_grid.set(voxel, 1)
            
    return output_grid
```

**Edge Cases and Pitfalls:**
- **Self-Intersection:** If `pad` causes the surface to intersect itself, the masked volume will not include the intersection region [S1]. Use `extend` to avoid this [S1].
- **Holes:** The `axis` direction matters when the surface has holes, as it determines the projection direction for "inside" determination [S1].
- **Nested Surfaces:** `fillOverlap` controls whether nested surfaces (e.g., concentric spheres) result in a solid sphere (`true`) or a hollow shell (`false`) [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels in the output grid, $O(N_{voxels})$, for the final assignment step. Surface displacement and ray-casting for "inside" tests depend on surface complexity.
- **Memory Complexity:** Linear with respect to the output grid size, $O(N_{voxels} \times \text{bytes\_per\_voxel})$ [S1].
- **Strengths:** Efficient generation of binary masks for volume rendering; flexible surface modification options (`pad`, `extend`, `slab`) [S1].
- **Limitations:** `pad` can cause self-intersection artifacts; `axis` dependency can lead to incorrect masking for complex topologies with holes if not chosen carefully [S1].

## Background
`volume onesmask` is a command in UCSF ChimeraX, part of the `volume` command suite. It is designed for structural biology and visualization tasks where binary masks are needed for segmentation or masking volume data. It is related to `volume mask` [S1], `volume multiply` [S1], and `shape` [S1].

## Sources
[S1] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
