# Voxel Format Generation

Also called: Collision Mesh Generation, Splat Transform Voxelizer

## Overview
Voxel Format Generation converts 3D Gaussian Splats into a sparse voxel octree optimized for collision detection and navigation, outputting metadata (`.voxel.json`) and binary data (`.voxel.bin`), while optionally generating a `.collision.glb` mesh. It distinguishes itself from general-purpose voxelization by integrating scene topology analysis (interior vs. exterior) and navigable space carving directly into the generation pipeline, rather than producing a raw density field.

## How it works
The method processes Gaussian splats through a sequence of filtering, spatial mapping, and topological sealing steps to produce a watertight or navigable collision volume.

### Inputs and Outputs
- **Input**: A Gaussian Splat file (PLY, SPLAT, KSPLAT, SOG) or a point cloud.
- **Outputs**:
  - `.voxel.json`: Metadata containing voxel resolution, opacity cutoff, and octree structure references.
  - `.voxel.bin`: Binary representation of the sparse voxel octree.
  - `.collision.glb` (optional): A mesh derived from the voxel grid for physics engines.

### Data Structures
The output relies on a sparse voxel octree. The binary file (`.voxel.bin`) stores the octree nodes, while the JSON file (`.voxel.json`) stores global parameters and potentially the root node index or bounding box. The specific binary layout of the octree nodes (e.g., child pointers, occupancy flags, attribute storage) is not detailed in the sources, but the structure is implied to be a standard sparse octree suitable for efficient traversal [S1].

### Parameters
- `voxelResolution` (float, default 0.05): Size of each voxel in world units.
- `opacityCutoff` (float, default 0.1): Opacity threshold; voxels with aggregated opacity below this value are treated as empty.
- `navExteriorRadius` (float): Exterior fill radius in world units. Enables exterior fill. Requires `navSeed`.
- `floorFill` (bool, default false): Boolean to fill voxel columns upward from the bottom until hitting a solid voxel. Runs before carving.
- `floorFillDilation` (float, default 0): Dilation radius in world units to identify 'interior' XZ columns. Empty XZ areas larger than `2 * floorFillDilation` from any solid column are treated as exterior.
- `navCapsule` (object `{height, radius}`): Dimensions for the carve capsule. Height 0 disables carve. Requires `navSeed`.
- `seedPos` (vec3): Seed position for voxel fill/carve and cluster filtering. Default: `(0, 0, 0)`.

### Steps
1.  **Filtering**: Run `--filter-cluster` to isolate the central scene and discard stray floaters. This requires a `--seed-pos` to define the center of interest [S1].
2.  **Scene Classification**: Determine if the scene is interior (rooms) or exterior (terrain/objects) to select the appropriate filling strategy [S1].
3.  **Voxelization**: Map the Gaussian splats to the sparse voxel grid.
    -   For **Interior Scenes**:
        -   Apply `--voxel-external-fill` to seal the void around the room interior. This uses `--seed-pos` to verify volume enclosure; if the seed is reachable from outside, the fill is skipped [S1].
        -   Apply `--voxel-carve` to hollow out navigable space using a capsule flood fill from the seed [S1].
    -   For **Exterior Scenes**:
        -   Apply `--voxel-floor-fill` to fill voxel columns upward from the bottom until hitting a solid voxel. This patches XZ areas surrounded by floor within `2 * floorFillDilation` [S1].
        -   Optionally apply `--voxel-carve` to create navigable space above the ground [S1].
4.  **Mesh Generation (Optional)**: Generate a collision mesh using `--collision-mesh`.
    -   `smooth`: Uses marching cubes with lossless coplanar merge for smoother geometry.
    -   `faces`: Provides direct watertight boundaries from the voxel data [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_format(input_splat_path, output_prefix, options):
    # 1. Load and Filter
    splats = load_splats(input_splat_path)
    if options.filter_cluster:
        # Isolate central scene based on seed position
        splats = filter_cluster(splats, options.seed_pos)
    
    # 2. Voxelization Setup
    voxel_size = options.voxel_resolution if options.voxel_resolution else 0.05
    opacity_cutoff = options.opacity_cutoff if options.opacity_cutoff else 0.1
    
    # Initialize sparse octree
    octree = SparseVoxelOctree(voxel_size=voxel_size)
    
    # 3. Aggregate Gaussians into Voxels
    for gaussian in splats:
        # Determine affected voxels based on Gaussian covariance/scale
        affected_voxels = get_affected_voxels(gaussian, octree)
        for voxel in affected_voxels:
            # Weighted aggregation of opacity and color
            weight = gaussian.opacity * gaussian_kernel_distance(gaussian.center, voxel.center)
            voxel.opacity += weight
            voxel.color += gaussian.color * weight
            voxel.total_weight += weight

    # 4. Normalize and Threshold
    for voxel in octree.all_voxels():
        if voxel.total_weight > 0:
            voxel.opacity = voxel.opacity / voxel.total_weight
        if voxel.opacity < opacity_cutoff:
            voxel.opacity = 0.0
            voxel.color = (0, 0, 0, 0)

    # 5. Topological Processing (Fill/Carve)
    if options.voxel_external_fill:
        # Seal void around interior
        # Check if seed is reachable from outside; if so, skip fill
        if not is_seed_reachable_from_outside(octree, options.seed_pos):
            fill_external_void(octree, options.seed_pos)
            
    if options.voxel_floor_fill:
        # Fill columns upward from bottom
        floor_dilation = options.floor_fill_dilation if options.floor_fill_dilation else 0
        fill_floor_columns(octree, dilation_radius=2 * floor_dilation)
        
    if options.voxel_carve:
        # Carve navigable space using capsule flood fill
        capsule_height = options.nav_capsule.height if options.nav_capsule else 1.6
        capsule_radius = options.nav_capsule.radius if options.nav_capsule else 0.2
        carve_navigable_space(octree, options.seed_pos, capsule_height, capsule_radius)

    # 6. Output
    save_voxel_json(octree, output_prefix + ".voxel.json", {
        "voxelResolution": voxel_size,
        "opacityCutoff": opacity_cutoff
    })
    save_voxel_bin(octree, output_prefix + ".voxel.bin")
    
    if options.collision_mesh:
        mesh_type = options.collision_mesh_type if options.collision_mesh_type else "smooth"
        if mesh_type == "smooth":
            mesh = marching_cubes_with_coplanar_merge(octree)
        else:
            mesh = voxel_boundary_faces(octree)
        save_glb(mesh, output_prefix + ".collision.glb")
```

## Subtopics

### Splat Transform CLI
A command-line tool implementation of Voxel Format Generation. It supports various input formats (PLY, SPLAT, KSPLAT, SOG, HTML) and provides specific flags for voxel generation. Key parameters include `--voxel-size` (default 0.05), `--voxel-opacity` (default 0.1), `--voxel-floor-fill` (default size 1.6), `--voxel-carve` (default height 1.6, radius 0.2), and `--collision-mesh` (default 'smooth'). It also supports `--seed-pos` for defining the scene center and navigable space seed [S1].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources. The tool uses a memory budget capped at 48 GiB or half the machine's RAM, whichever is lower, for residency policies during decimation and rendering [S1].
- **Performance**: Not given in the sources.
- **Strengths**: Integrates collision mesh generation directly with voxelization; supports both smooth (marching cubes) and watertight (voxel boundary) mesh outputs; handles both interior and exterior scene topologies automatically.
- **Limitations**: Requires a seed position for accurate clustering and carving; performance on degenerate scenes with high overdraw may be impacted (though statistics are available for validation) [S1].

## Background
Voxel Format Generation is part of the `splat-transform` toolchain developed by PlayCanvas. It is designed to bridge the gap between 3D Gaussian Splatting representations and traditional collision detection systems used in game engines and navigation applications. The method was introduced as part of the broader `splat-transform` library, which provides tools for converting, filtering, and transforming Gaussian splat data [S1].

## Sources
[S1] playcanvas/splat-transform, https://github.com/playcanvas/splat-transform, fetched 2026-10-27
