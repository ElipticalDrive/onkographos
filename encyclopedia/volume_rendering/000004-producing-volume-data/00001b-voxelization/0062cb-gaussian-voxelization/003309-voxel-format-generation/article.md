# Voxel Format Generation

Also called: Collision Mesh Generation, Splat Transform Voxelizer

## Overview
Voxel Format Generation converts 3D Gaussian Splats into a sparse voxel octree optimized for collision detection, outputting metadata and binary data files. It distinguishes itself from general-purpose voxelizers by integrating scene topology analysis (interior vs. exterior) and generating watertight collision meshes directly from the voxel grid, rather than just visual representation.

## How it works
The method processes 3D Gaussian primitives to produce a sparse voxel octree structure. It involves filtering, spatial discretization, topological filling/carving, and optional mesh extraction.

**Inputs and Outputs:**
*   **Input:** A 3D Gaussian Splat file (e.g., `.ply`, `.sog`).
*   **Outputs:**
    *   `.voxel.json`: Metadata describing the octree structure.
    *   `.voxel.bin`: Binary data of the sparse voxel octree.
    *   `.collision.glb`: (Optional) A mesh derived from the voxel grid.

**Data Structures:**
The output is a sparse voxel octree. The binary format stores the octree nodes, while the JSON stores metadata. Specific internal bit-packing of the octree nodes is not detailed in the sources [S1].

**Parameters:**
*   `voxelResolution`: Size of each voxel in world units. Default: `0.05` [S1].
*   `opacityCutoff`: Opacity threshold for solid voxels. Voxels with opacity below this are treated as empty. Default: `0.1` [S1].
*   `navExteriorRadius`: Exterior fill radius in world units. Enables exterior fill mode. Requires `navSeed` [S1].
*   `floorFill`: Boolean to fill voxel columns upward from the bottom until hitting a solid voxel. Runs before carving. Default: `false` [S1].
*   `floorFillDilation`: Dilation radius in world units to identify 'interior' XZ columns. Empty XZ areas larger than $2 \times \text{floorFillDilation}$ from any solid column are treated as exterior. Default: `0` [S1].
*   `navCapsule`: Object `{height, radius}` defining the dimensions for carving. A height of `0` disables carving. Requires `navSeed` [S1].
*   `seedPos`: A known walkable point (x,y,z) inside the scene, used by filtering and fill/carve passes [S1].

**Steps:**
1.  **Filtering:** Run `--filter-cluster` with `--seed-pos` to isolate the central scene geometry and discard stray floaters [S1].
2.  **Scene Classification & Filling:**
    *   For **interior scenes** (rooms): Apply `--voxel-external-fill` to seal the void around the room interior [S1].
    *   For **exterior scenes** (terrain): Apply `--voxel-floor-fill` to fill ground beneath surfaces. This may use `floorFillDilation` to distinguish interior from exterior columns [S1].
3.  **Carving:** Apply `--voxel-carve` using `navCapsule` dimensions to hollow out navigable space from the solid voxels [S1].
4.  **Mesh Generation (Optional):** Generate a collision mesh using `--collision-mesh`. Choose shape `'smooth'` (marching cubes with lossless coplanar merge) or `'faces'` (direct watertight voxel-boundary faces) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_format(input_ply, output_voxel_json, args):
    # 1. Load and Filter
    splats = load_ply(input_ply)
    if args.filter_cluster:
        # Isolate central cluster around seed_pos, discard others
        splats = filter_cluster(splats, args.seed_pos)
    
    # 2. Voxelization
    # Discretize splats into a sparse voxel octree based on voxelResolution
    # and opacityCutoff.
    octree = voxelize(splats, args.voxelResolution, args.opacityCutoff)
    
    # 3. Topological Processing
    if args.voxel_external_fill:
        # Seal void around interior
        octree = fill_external_void(octree, args.seed_pos)
    
    if args.floor_fill:
        # Fill columns upward from bottom
        octree = fill_floor(octree, args.floor_fill_dilation)
        
    if args.voxel_carve:
        # Hollow out navigable space using capsule dimensions
        octree = carve_navigable_space(octree, args.nav_capsule, args.seed_pos)
        
    # 4. Output
    save_metadata(octree, output_voxel_json)
    save_binary(octree, output_voxel_json.replace('.json', '.bin'))
    
    if args.collision_mesh:
        if args.collision_mesh_shape == 'smooth':
            mesh = marching_cubes(octree)
            mesh = merge_coplanar_faces(mesh)
        elif args.collision_mesh_shape == 'faces':
            mesh = extract_voxel_faces(octree)
        save_mesh(mesh, output_voxel_json.replace('.json', '.glb'))
```

## Subtopics

### [Splat Transform CLI](6927-splat-transform-cli.md)
A command-line utility for converting between various Gaussian splat formats and generating voxel/collision data. It distinguishes itself by supporting out-of-core processing pipelines for large datasets, ensuring scalability for scenes exceeding available system RAM.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Integrates scene topology analysis (interior/exterior) and generates watertight collision meshes. Supports out-of-core processing for large scenes via the CLI [S1].
*   **Limitations:** Requires manual selection of scene type (interior vs. exterior) and parameters like `seedPos` and `navCapsule` for correct carving [S1].

## Background
The method is implemented in the `splat-transform` tool by PlayCanvas [S1]. It is designed for preparing 3D Gaussian Splat data for physics and collision detection in interactive applications.

## Sources
[S1] playcanvas/splat-transform, https://github.com/playcanvas/splat-transform, fetched 2026-10-27
