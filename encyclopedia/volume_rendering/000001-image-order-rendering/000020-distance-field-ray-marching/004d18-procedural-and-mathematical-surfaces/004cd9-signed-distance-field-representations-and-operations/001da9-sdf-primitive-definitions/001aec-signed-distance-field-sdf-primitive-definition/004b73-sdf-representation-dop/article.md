# SDF Representation DOP

Also called: Signed Distance Field Representation

## Overview
The SDF Representation DOP creates a volumetric signed distance field (SDF) from geometry to enable fast inside/outside testing for collision detection [S1]. It differs from general volume rendering methods by focusing on the topological correctness of the distance field for rigid body dynamics rather than visual appearance, utilizing specific modes like Laser Scanning to handle non-watertight geometry [S1].

## How it works
**Inputs and Outputs**
*   **Input Geometry:** The parent geometry data is used to create the SDF. A Geometry data attached as subdata overrides the parent, allowing proxy geometry or explicit external volume files [S1].
*   **First Input:** Optional connector to control which simulation objects are modified via the Group parameter [S1].
*   **Other Inputs:** Connectors for data nodes that act as modifiers [S1].
*   **Output:** A signed distance field data structure attached to the specified objects [S1].

**Data Structures**
The SDF is stored on a 3D grid (voxel grid). The grid resolution is defined by the `Divisions` parameter, computed within a bounding box slightly larger than the geometry [S1]. In `Volume Sample` mode, divisions are ignored and computed from the source volume's voxel size [S1]. Each voxel stores a scalar distance value: positive outside, negative inside, zero on the surface [S1].

**Algorithm Steps**
The construction process depends on the selected `Mode` and `Laser Scan` setting:

1.  **Geometry Bounding:** Calculate the bounding box or sphere of the input geometry [S1].
2.  **Grid Initialization:** Initialize a 3D grid based on `Divisions` (or source volume resolution in `Volume Sample` mode) [S1].
3.  **Distance Calculation:**
    *   **Ray Intersect Mode:** Perform ray intersection with the geometry to determine accurate volumetric representation [S1].
    *   **Meta Balls Mode:** Evaluate the metaball field directly; requires `Laser Scan` off [S1].
    *   **Implicit Box/Sphere/Plane:** Create a representation filling the bounding box, sphere, or a plane dividing the bounding box along its smallest axis [S1].
    *   **Minimum Mode:** Compute distance to surface/curve. Positive `Offset` creates an expanded volume [S1].
    *   **Volume Sample Mode:** Sample the raw volume primitive, treating it as an existing SDF [S1].
    *   **Height Field Mode:** Treat the volume as a 2D heightfield; distances are computed orthographically [S1].
    *   **Laser Scan Mode (On):** Send rays along primary axes. Use only the closest and farthest intersections. Voxels between intersections are marked inside; others outside. This handles non-watertight or self-intersecting geometry but misses interior features [S1].
    *   **Laser Scan Mode (Off):** Send rays along primary axes. Find all intersections. Test each segment between pairs of intersections for inside/outside status. Requires manifold, watertight geometry with well-defined normals [S1].
4.  **Sign Correction (Fix Signs):** If enabled, post-process the SDF to correct numerical imprecision and inconsistent signs, plugging leaks and filling holes [S1].
5.  **Boundary Stabilization (Force Bounds):** If enabled, force all voxels on the boundary to be marked as exterior to prevent incorrect stabilization of inverted signs at the SDF boundary [S1].
6.  **Sign Inversion:** If enabled, reverse the inside/outside sense (positive becomes negative) [S1].
7.  **Sign Sweep (Optional):** If `Sign Sweep Threshold` is set, perform additional passes to eliminate large blocks of inconsistent signs. A transition is invalid if the SDF value change exceeds `Threshold * Cell Width`. The original geometry is ray-intersected to determine the correct sign, which is then propagated [S1]. Repeat up to `Max Sign Sweep Count` [S1].

**Parameters**
*   `Mode`: String. Options: `Ray Intersect`, `Meta Balls`, `Implicit Box`, `Implicit Sphere`, `Implicit Plane`, `Minimum`, `Volume Sample`, `Height Field` [S1].
*   `Divisions`: Integer. Grid resolution. Minimum feature size must be larger than grid spacing to avoid detail loss [S1].
*   `Offset`: Float. Expands (positive) or shrinks (negative) the volume [S1].
*   `Tolerance`: Float. Scale-invariant tolerance for ray intersections, multiplied by geometry size [S1].
*   `Laser Scan`: Boolean. On: uses closest/farthest intersections (robust to non-watertight). Off: uses all intersections (requires watertight) [S1].
*   `Fix Signs`: Boolean. Enables post-processing for sign consistency [S1].
*   `Force Bounds`: Boolean. Forces boundary voxels to exterior [S1].
*   `Invert Sign`: Boolean. Reverses inside/outside [S1].
*   `Sign Sweep Threshold`: Float. Threshold for invalid sign transitions relative to cell width [S1].
*   `Max Sign Sweep Count`: Integer. Maximum iterations for sign sweep [S1].
*   `Data Sharing`: Enum. `Do Not Share Data`, `Share Data Across All Time`, `Share Data In One Timestep` [S1].
*   `Activation`: Expression/Float. Non-zero attaches data; zero removes it [S1].
*   `Group`: String. Subset of objects to affect [S1].
*   `Data Name`: String. Name for attaching data, supports subdata paths (e.g., "Forces/Fan") [S1].
*   `Unique Data Name`: Boolean. Ensures unique data name to prevent overwriting [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_sdf(geometry, params):
    # 1. Determine geometry source
    if params['subdata_geometry']:
        source_geom = params['subdata_geometry']
    else:
        source_geom = geometry

    # 2. Determine grid bounds and resolution
    if params['mode'] == 'Volume Sample':
        # Use source volume divisions
        grid = create_grid_from_volume(source_geom)
    else:
        bbox = calculate_bounding_box(source_geom)
        grid = create_grid(bbox, params['divisions'])

    # 3. Compute distances based on mode
    if params['mode'] == 'Ray Intersect':
        compute_ray_intersect_sdf(grid, source_geom, params['tolerance'])
    elif params['mode'] == 'Meta Balls':
        compute_metaball_sdf(grid, source_geom)
    elif params['mode'] == 'Implicit Box':
        fill_grid_with_bbox(grid, source_geom)
    elif params['mode'] == 'Implicit Sphere':
        fill_grid_with_bounding_sphere(grid, source_geom)
    elif params['mode'] == 'Implicit Plane':
        create_plane_sdf(grid, source_geom)
    elif params['mode'] == 'Minimum':
        compute_minimum_distance_sdf(grid, source_geom, params['offset'])
    elif params['mode'] == 'Volume Sample':
        sample_volume_sdf(grid, source_geom)
    elif params['mode'] == 'Height Field':
        compute_heightfield_sdf(grid, source_geom)
    else:
        # Default Laser Scan logic
        if params['laser_scan']:
            compute_laser_scan_sdf(grid, source_geom, params['tolerance'])
        else:
            compute_full_ray_sdf(grid, source_geom, params['tolerance'])

    # 4. Fix Signs
    if params['fix_signs']:
        correct_signs(grid, source_geom)
        if params['force_bounds']:
            force_boundary_exterior(grid)
        
        # 5. Sign Sweep
        if params['sign_sweep_threshold'] > 0:
            for i in range(params['max_sign_sweep_count']):
                if not sweep_signs(grid, params['sign_sweep_threshold'], source_geom):
                    break

    # 6. Invert Sign
    if params['invert_sign']:
        invert_sdf(grid)

    return grid
```

**Edge Cases and Pitfalls**
*   **Non-watertight Geometry:** Use `Laser Scan` mode to avoid reliance on normals [S1].
*   **Interior Features:** `Laser Scan` mode cannot represent interior features [S1].
*   **Sign Inversion Stabilization:** `Fix Signs` may stabilize inverted signs at the boundary; use `Force Bounds` to mitigate [S1].
*   **Convergence:** Too low a `Sign Sweep Threshold` may prevent convergence [S1].
*   **Hollow Objects:** Use `Invert Sign` on an inner box for robust hollow representation [S1].

## Operating characteristics
*   **Time Complexity:** Depends on grid resolution (`Divisions`) and geometry complexity. Laser scanning is generally faster than full ray intersection for complex non-watertight geometry [S1]. Sign sweeps add iterative cost [S1].
*   **Memory Complexity:** Proportional to the number of voxels in the grid. Data sharing options (`Share Data Across All Time`, `Share Data In One Timestep`) can significantly reduce memory footprint for multiple objects [S1].
*   **Strengths:** Fast inside/outside testing; robust handling of non-watertight geometry via Laser Scanning; flexible modes for different geometry types [S1].
*   **Limitations:** Precomputation cost; memory usage for high-resolution grids; potential for sign errors requiring post-processing [S1].
*   **When to Use:** Rigid body collision detection in simulations [S1].
*   **When Not to Use:** When visual fidelity of the volume is the primary goal rather than collision topology; when geometry is extremely complex and grid resolution cannot capture details [S1].

## Background
The SDF Representation DOP is part of Houdini's dynamics (DOP) network, designed to integrate signed distance fields into simulation workflows for collision detection [S1]. It builds on the concept of SDFs as functions evaluating distance to surface with sign indicating inside/outside [S1].

## Sources
[S1] SDF Representation - SideFX, https://www.sidefx.com/docs/houdini/nodes/dop/volume.html, fetched 2026-10-27
