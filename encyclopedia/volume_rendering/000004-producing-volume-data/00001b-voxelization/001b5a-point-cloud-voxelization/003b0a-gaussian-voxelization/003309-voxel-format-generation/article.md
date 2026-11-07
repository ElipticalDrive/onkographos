# Voxel Format Generation

Also called: Collision Mesh Generation, Splat Transform Voxelizer

## Overview
Voxel Format Generation converts a set of 3D Gaussians into a sparse voxel octree optimized for collision detection and navigation, outputting metadata and binary data files. It distinguishes itself from general-purpose voxelization by integrating scene topology analysis (interior vs. exterior) and capsule-based navigation carving directly into the generation pipeline, rather than producing a raw density grid. It builds on [Gaussian Voxelization](8884-gaussian-voxelization.md) by adding semantic processing steps for walkable space identification.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D Gaussian Splat file (e.g., `.ply`, `.ksplat`, `.sog`).
*   **Outputs:**
    *   `.voxel.json`: Metadata describing the octree structure, bounds, and parameters.
    *   `.voxel.bin`: Binary representation of the sparse voxel octree.
    *   `.collision.glb` (optional): A mesh derived from the voxel grid for visualization or physics engines.

**Data Structures:**
The output is a sparse voxel octree. The binary format stores the hierarchy of nodes. The JSON file contains the root node's bounding box, voxel resolution, and flags indicating which processing steps were applied.

**Steps:**
1.  **Filtering:** Run `--filter-cluster` with a `--seed-pos` to isolate the central scene cluster and discard stray floaters (noise far from the seed). [S1]
2.  **Scene Classification & Filling:**
    *   **Interior Scenes:** Apply `--voxel-external-fill` to seal the void surrounding the room interior. This verifies volume enclosure using the seed position; if the seed is reachable from outside, the fill is skipped. [S1]
    *   **Exterior Scenes:** Apply `--voxel-floor-fill` to fill voxel columns upward from the bottom until hitting solid geometry. This identifies "interior" XZ columns by checking if empty XZ areas are within `2 * floorFillDilation` of any solid column. [S1]
3.  **Navigation Carving:** Apply `--voxel-carve` using a capsule flood fill from the `--seed-pos`. The capsule dimensions (`height`, `radius`) define the navigable space. A height of 0 disables carving. [S1]
4.  **Voxelization:** Convert the filtered and processed Gaussians into the sparse octree. Voxels with opacity below `opacityCutoff` are treated as empty. [S1]
5.  **Mesh Generation (Optional):** Generate a `.collision.glb` mesh using `--collision-mesh`.
    *   `smooth`: Uses marching cubes with lossless coplanar merge for smoother geometry. [S1]
    *   `faces`: Generates direct watertight voxel-boundary faces. [S1]

**Parameters:**
*   `voxelResolution`: Size of each voxel in world units. Default: `0.05`. [S1]
*   `opacityCutoff`: Opacity threshold for solid voxels. Default: `0.1`. [S1]
*   `seedPos`: Position `[x, y, z]` for clustering and fill/carve seeds. Default: `0,0,0`. [S1]
*   `navExteriorRadius`: Exterior fill radius in world units. Enables exterior fill. Requires `navSeed`. [S1]
*   `floorFill`: Boolean to fill voxel columns upward from bottom. Default: `false`. [S1]
*   `floorFillDilation`: Dilation radius in world units to identify 'interior' XZ columns. Default: `0`. [S1]
*   `navCapsule`: Object `{height, radius}` for carve dimensions. Default: `height=1.6, radius=0.2`. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_format(gaussian_splats, options):
    # 1. Filter stray floaters
    if options.filter_cluster:
        seed = options.seed_pos
        # Isolate central cluster around seed, discard distant splats
        splats = filter_cluster(gaussian_splats, seed)
    
    # 2. Scene Processing
    if options.voxel_external_fill:
        # Seal void around interior. Skip if seed is reachable from outside.
        splats = voxel_external_fill(splats, seed, options.voxel_resolution)
    
    if options.floor_fill:
        # Fill columns upward from bottom for exterior scenes
        # Identify interior XZ columns based on dilation
        splats = voxel_floor_fill(splats, options.floor_fill_dilation, options.voxel_resolution)
        
    if options.voxel_carve:
        # Carve navigable space using capsule flood fill
        capsule = options.nav_capsule # {height, radius}
        splats = voxel_carve(splats, seed, capsule, options.voxel_resolution)
        
    # 3. Voxelization
    # Convert Gaussians to sparse octree
    # Voxels with opacity < opacity_cutoff are empty
    octree = gaussian_to_octree(splats, options.opacity_cutoff, options.voxel_resolution)
    
    # 4. Output
    write_voxel_json(octree, options)
    write_voxel_bin(octree)
    
    if options.collision_mesh:
        if options.collision_mesh_type == 'smooth':
            mesh = marching_cubes_with_merge(octree)
        elif options.collision_mesh_type == 'faces':
            mesh = voxel_boundary_faces(octree)
        write_collision_glb(mesh)
        
    return octree
```

## Subtopics

### Splat Transform CLI
A command-line tool interface for the Voxel Format Generation method. It supports various input formats (PLY, SPLAT, KSPLAT, SOG, HTML) and exposes all voxelization parameters via flags. It allows chaining transformations (scale, translate, rotate) before voxelization. It also provides statistics generation (`--stats`) to analyze per-column data, including fill ratio (average overdraw layer count), which helps identify degenerate scenes. [S1]

## Operating characteristics
*   **Time Complexity:** Dominated by the clustering and flood-fill operations, which are linear in the number of Gaussians and the volume of the scene, respectively. The octree construction is efficient due to sparsity.
*   **Memory Complexity:** Proportional to the number of non-empty voxels in the octree. The binary format is compact.
*   **Performance:** The tool is designed for pre-processing. The `--filter-cluster` step significantly reduces the number of Gaussians to process by removing noise. The `--stats` option can help tune `voxelResolution` and `opacityCutoff` to balance detail and performance. [S1]
*   **Strengths:** Integrates navigation-aware carving and scene topology handling (interior/exterior) directly into the voxel generation. Produces a compact binary format suitable for real-time collision detection.
*   **Limitations:** Requires a valid `--seed-pos` for clustering and carving. The quality of the collision mesh depends on the `voxelResolution` and the `opacityCutoff`. [S1]

## Background
The method is implemented in the `splat-transform` tool by PlayCanvas. It was developed to provide a robust pipeline for converting 3D Gaussian Splatting data into formats suitable for interactive applications, specifically focusing on collision detection and navigation. The tool supports various splat formats and provides advanced filtering and transformation capabilities. [S1]

## Sources
[S1] playcanvas/splat-transform, https://github.com/playcanvas/splat-transform, fetched 2026-10-27
