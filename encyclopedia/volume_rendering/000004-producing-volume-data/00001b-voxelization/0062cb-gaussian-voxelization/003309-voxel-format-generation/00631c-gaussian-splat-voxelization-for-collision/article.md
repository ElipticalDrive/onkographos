# Gaussian Splat Voxelization for Collision

Also called: Voxalization for Collision, grid-based collision, voxel-based collision, walkalization

## Overview
This method converts 3D Gaussian Splat data into a three-dimensional grid of occupied cells (voxels) to automatically generate a collision structure [S1, unofficial]. It solves the problem of creating watertight collision shells for Gaussian splats, which lack explicit geometry, by allowing the Gaussian density data to build the shell automatically [S1, unofficial]. It differs from manual mesh construction by automating the generation of a collision mesh (e.g., ~10,000 triangles for a 3-million-splat scene) from volumetric data [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A Gaussian Splat file (e.g., `.ply`) and scene parameters (indoor/outdoor mode, voxel size, opacity threshold).
*   **Output:** A collision mesh in GLB format [S1, unofficial]. Intermediate sparse voxel structures may be output as JSON and binary files (`.voxel.json`, `.voxel.bin`) [S1, unofficial].

**Data Structures**
*   **Voxel Grid:** A 3D grid covering the scene bounds. Each cell stores a density value accumulated from nearby Gaussians [S1, unofficial].
*   **Sparse Voxel Structure:** A representation of occupied voxels, stored in JSON (metadata) and binary (data) formats for efficient processing [S1, unofficial].

**Parameters**
*   **Voxel Size:** Determines the resolution of the grid; smaller values yield finer structures [S1, unofficial].
*   **Opacity Threshold:** The density value a voxel must exceed to be considered solid [S1, unofficial].

**Algorithm Steps**
The process varies slightly based on scene type (Indoor vs. Outdoor).

1.  **Grid Initialization:** Lay a 3D grid over the entire scene bounds [S1, unofficial].
2.  **Density Accumulation:** The GPU sweeps through the grid. For each voxel, sum the density contributions of all Gaussians overlapping that voxel [S1, unofficial].
3.  **Binary Classification:** Check if the accumulated density exceeds the opacity threshold. If yes, mark the voxel as solid; otherwise, mark it as air [S1, unofficial].
4.  **Cleanup:** Automatically remove isolated voxels floating in air and fill tiny holes in the solid structure [S1, unofficial].
5.  **Scene-Specific Processing:**
    *   **Indoor Mode:**
        1.  **Exterior Fill:** Fluid flows in from outside the scene bounds, turning everything beyond the walls solid. Walls are temporarily thickened to prevent fluid leaking through small caps or openings [S1, unofficial].
        2.  **Carving:** A seed point is positioned in free space inside the room. A capsule expands from this seed, carving out reachable space. The capsule only moves through open space; areas it cannot reach (e.g., inside walls, narrow gaps) become solid [S1, unofficial].
    *   **Outdoor Mode:**
        1.  **Floor Fill:** Use a special floor fill feature. Walk up every column of voxels from below until hitting the first surface. Everything underneath becomes solid ground. Empty columns at the scene edges become invisible walls [S1, unofficial].
        2.  **Carving:** A seed point is placed in open space slightly above the ground. A capsule expands from this seed. Everything reached becomes the final collision surface [S1, unofficial].
6.  **Mesh Generation:** Merge the resulting voxels into larger rectangles to form a simple collision mesh [S1, unofficial].
7.  **Export:** Save the mesh as a GLB file [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_splat_voxelization_for_collision(gaussian_splats, scene_bounds, voxel_size, opacity_threshold, mode='indoor'):
    """
    Converts Gaussian splats to a collision mesh via voxelization.
    
    Args:
        gaussian_splats: List of Gaussian objects with position, covariance, opacity.
        scene_bounds: Bounding box (min, max) for the scene.
        voxel_size: Size of each voxel in world units.
        opacity_threshold: Density threshold for solid voxels.
        mode: 'indoor' or 'outdoor'.
        
    Returns:
        collision_mesh: A mesh object (e.g., GLB structure).
    """
    # 1. Grid Initialization
    grid_resolution = calculate_grid_resolution(scene_bounds, voxel_size)
    voxel_grid = create_3d_array(grid_resolution, dtype=float)
    
    # 2. Density Accumulation
    for voxel in voxel_grid:
        voxel.density = 0.0
        for gaussian in gaussian_splats:
            if gaussian.overlaps(voxel.center, voxel.size):
                voxel.density += gaussian.density_at(voxel.center)
                
    # 3. Binary Classification
    solid_voxels = []
    for voxel in voxel_grid:
        if voxel.density > opacity_threshold:
            voxel.status = 'solid'
            solid_voxels.append(voxel)
        else:
            voxel.status = 'air'
            
    # 4. Cleanup
    solid_voxels = remove_isolated_voxels(solid_voxels)
    solid_voxels = fill_tiny_holes(solid_voxels)
    
    # 5. Scene-Specific Processing
    if mode == 'indoor':
        # 5a. Exterior Fill
        solid_voxels = exterior_fill(solid_voxels, scene_bounds, voxel_size)
        # 5b. Carving
        seed_point = find_interior_seed(solid_voxels, scene_bounds)
        solid_voxels = carve_interior(solid_voxels, seed_point, capsule_radius=voxel_size * 0.5)
    elif mode == 'outdoor':
        # 5a. Floor Fill
        solid_voxels = floor_fill(solid_voxels, scene_bounds, voxel_size)
        # 5b. Carving
        seed_point = find_outdoor_seed(solid_voxels, scene_bounds)
        solid_voxels = carve_interior(solid_voxels, seed_point, capsule_radius=voxel_size * 0.5)
        
    # 6. Mesh Generation
    collision_mesh = merge_voxels_to_mesh(solid_voxels)
    
    # 7. Export
    return collision_mesh

def exterior_fill(solid_voxels, scene_bounds, voxel_size):
    """
    Fills space outside the scene boundaries with solid voxels.
    Temporarily thickens walls to prevent leakage.
    """
    # Implementation detail: Flood fill from outside bounds
    # Thicken walls by expanding solid voxels by 1-2 voxels in all directions
    # Fill all voxels outside the original scene bounds with solid status
    pass

def carve_interior(solid_voxels, seed_point, capsule_radius):
    """
    Carves out reachable space from a seed point using a capsule expansion.
    """
    # Implementation detail: BFS/DFS from seed_point
    # A voxel is 'carved' (marked as air) if it is reachable by a capsule of given radius
    # from the seed point without intersecting other solid voxels
    reachable_voxels = bfs_reachable(solid_voxels, seed_point, capsule_radius)
    for voxel in solid_voxels:
        if voxel not in reachable_voxels:
            voxel.status = 'solid' # Keep as solid if unreachable
        else:
            voxel.status = 'air' # Carve if reachable
    return solid_voxels

def floor_fill(solid_voxels, scene_bounds, voxel_size):
    """
    Fills ground from below. Walks up each column until hitting a surface.
    """
    for x in range(scene_bounds.min.x, scene_bounds.max.x, voxel_size):
        for z in range(scene_bounds.min.z, scene_bounds.max.z, voxel_size):
            # Walk up y-axis
            for y in range(scene_bounds.min.y, scene_bounds.max.y, voxel_size):
                voxel = get_voxel(x, y, z)
                if voxel.status == 'solid':
                    # Mark all voxels below this as solid ground
                    for y_below in range(scene_bounds.min.y, y, voxel_size):
                        get_voxel(x, y_below, z).status = 'solid'
                    break
    return solid_voxels

def remove_isolated_voxels(solid_voxels):
    """
    Removes voxels that are not connected to the main structure.
    """
    # Implementation detail: Connected component analysis
    # Keep only the largest connected component
    pass

def fill_tiny_holes(solid_voxels):
    """
    Fills small gaps in the solid structure.
    """
    # Implementation detail: Morphological closing or small component removal
    pass

def find_interior_seed(solid_voxels, scene_bounds):
    """
    Finds a point in free space inside the room.
    """
    # Implementation detail: Sample points inside bounds, check if air
    pass

def find_outdoor_seed(solid_voxels, scene_bounds):
    """
    Finds a point in open space above the ground.
    """
    # Implementation detail: Sample points above ground level, check if air
    pass

def merge_voxels_to_mesh(solid_voxels):
    """
    Merges voxels into larger rectangles to form a mesh.
    """
    # Implementation detail: Marching Cubes or similar surface extraction
    # Or direct merging of adjacent voxels into quads/tris
    pass
```

**Edge Cases and Pitfalls**
*   **Precision:** Voxel size must be chosen carefully; too large loses detail, too small increases memory and computation [S1, unofficial].
*   **Boundaries:** In outdoor scenes, Gaussians may extend far into the distance, requiring spherical cropping or careful boundary definition to avoid excessive voxelization [S1, unofficial].
*   **Leakage:** In indoor scenes, small caps or openings can cause exterior fill to leak into the interior if walls are not sufficiently thickened [S1, unofficial].
*   **Narrow Gaps:** The carving phase may fail to carve narrow gaps if the capsule radius is too large, incorrectly marking them as solid [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and voxel grid resolution. The GPU sweep is efficient, but cleanup and carving phases add complexity [S1, unofficial].
*   **Memory Complexity:** Proportional to the voxel grid size. Sparse voxel structures help mitigate this [S1, unofficial].
*   **Performance:** A room with ~3 million Gaussians produces a collision mesh of ~10,000 triangles, less than 200 kilobytes [S1, unofficial].
*   **Strengths:** Automates collision shell generation; produces lightweight meshes suitable for game engines [S1, unofficial].
*   **Limitations:** Requires careful parameter tuning (voxel size, threshold); may not capture fine details if voxel size is too large [S1, unofficial].
*   **When to Use:** When automatic collision generation for Gaussian splats is needed, especially for indoor or outdoor scenes where manual meshing is impractical [S1, unofficial].
*   **When Not to Use:** When high-fidelity collision geometry is required, or when the scene is extremely large and memory is constrained [S1, unofficial].

## Background
The method is implemented in **Splat Transform** (CLI and UI wrapper by Geeken Gifford) and **Super Splat** (web-based, processed by PlayCanvas server) [S1, unofficial]. It leverages the volumetric nature of Gaussian splats to create "walkable" surfaces, analogous to voxels in games like Minecraft [S1, unofficial]. The output GLB format is widely supported in game engines and 3D software [S1, unofficial].

## Sources
[S1] Making Gaussian Splats Solid – How 3DGS Collision Works?, https://www.youtube.com/watch?v=vFnYON7xRqM, fetched 2026-10-27 (unofficial)
