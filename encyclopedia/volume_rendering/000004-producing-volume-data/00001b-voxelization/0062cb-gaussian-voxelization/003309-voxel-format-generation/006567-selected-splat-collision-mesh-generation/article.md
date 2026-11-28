# Selected-splat Collision Mesh Generation

Also called: Lichtfeld Collision Mesh, Smooth remesh

## Overview
Selected-splat Collision Mesh Generation converts a user-selected subset of 3D Gaussian splats into a watertight collision mesh featuring perimeter walls and a ceiling, solving the problem of creating navigable or collidable geometry from sparse, non-manifold neural radiance field data [S1, unofficial]. It differs from sibling methods by rasterizing only the selected splat centers into a sparse 3D grid to preserve floating blobs and overhangs, then explicitly constructing solid boundary walls and a unified ceiling over the horizontal footprint, rather than relying solely on implicit surface extraction [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A scene containing Gaussian splat nodes; a binary selection mask identifying specific splats (typically ground/footprint and floating objects); parameters for mesh detail, wall height, smoothing, and opacity filtering.
*   **Output:** A collision mesh in OBJ, GLB, or PlayCanvas voxel format (.voxel.json + .voxel.bin) [S1, unofficial].

**Data Structures**
*   **Selection Mask:** A boolean array matching the total count of Gaussian splats in the scene nodes.
*   **Sparse 3D Grid:** A dictionary mapping 3D voxel coordinates $(x, y, z)$ to occupancy status, or a hierarchical octree structure for binary export [S1, unofficial].
*   **Voxel Format (PlayCanvas):**
    *   `.voxel.json`: Metadata including version '1.1', grid bounds, scene bounds, voxel resolution, leaf size (4), tree depth, node counts, and leaf data counts [S1, unofficial].
    *   `.voxel.bin`: A binary array of 32-bit integers representing a compacted octree. Nodes store pointers to children or leaf masks. Leaves store 64-bit occupancy masks packed into two 32-bit integers [S1, unofficial].

**Parameters**
*   **Mesh Detail:** Presets in meters: 0.05, 0.1 (default), 0.2, 0.5, 1.0 [S1, unofficial].
*   **Wall Height:** Default 3.0 meters [S1, unofficial].
*   **Smoothing Amount:** Default 0.35 voxels, range 0.0 to 0.45 [S1, unofficial].
*   **Minimum Splat Opacity:** Default 0.15 [S1, unofficial].
*   **Ignore Transparent Splats:** Boolean, enabled by default [S1, unofficial].

**Steps**
1.  **Scene Validation:** Verify a scene is loaded and training is paused [S1, unofficial].
2.  **Selection Aggregation:**
    *   Retrieve all splat nodes.
    *   Apply the global selection mask to identify active splats.
    *   Iterate through nodes. For each node, extract the raw means ($\mathbf{xyz}$) and raw opacity logits.
    *   If `Ignore transparent splats` is enabled:
        *   Filter splats where the individual opacity (derived from raw logits via a stable threshold comparison) is below `Minimum splat opacity` [S1, unofficial].
        *   Note: This uses per-splat opacity, not accumulated rendered opacity, and does not check connectivity [S1, unofficial].
    *   Transform the remaining selected centers to world coordinates using the node's world transform matrix $\mathbf{M}$: $\mathbf{p}_{world} = \mathbf{p}_{local} \mathbf{M}_{3\times3}^T + \mathbf{M}_{3\times1}$ [S1, unofficial].
3.  **Voxelization:**
    *   Rasterize the world-space centers into a sparse 3D grid.
    *   The grid preserves separate height layers for floating blobs and overhangs [S1, unofficial].
    *   Determine the horizontal footprint (unique $(x, z)$ pairs).
4.  **Boundary Construction:**
    *   Identify the perimeter of the horizontal footprint.
    *   For each $(x, z)$ in the footprint:
        *   Determine the vertical extent $[y_{min}, y_{max}]$ based on the selected splat centers at that location.
        *   If $(x, z)$ is on the perimeter (adjacent to an empty cell in the footprint), fill the voxel column from $y_{min}$ up to $y_{ceiling}$ [S1, unofficial].
        *   If $(x, z)$ is internal, fill from $y_{min}$ to $y_{max}$ [S1, unofficial].
    *   Calculate $y_{ceiling}$: The maximum $y$ coordinate of any selected splat center plus `Wall Height`, measured along the native -Y axis (up) [S1, unofficial].
    *   Disconnected footprint components remain separate [S1, unofficial].
5.  **Mesh Generation (Smooth Remesh):**
    *   Apply a light Gaussian filter to the 3D occupancy field with sigma `Smoothing Amount` [S1, unofficial].
    *   Extract the surface using Lewiner's Marching Cubes algorithm on the filtered field [S1, unofficial].
    *   Limit: Dense grid capped at 32 million samples; surface capped at 2 million triangles [S1, unofficial].
6.  **Export:**
    *   **OBJ/GLB:** Output the generated mesh. GLB defaults to Lichtfeld world coordinates (native -Y up) [S1, unofficial].
    *   **Voxel:** Quantize the ceiling to voxel size. Binary roofs may extend less than one voxel beyond the exact mesh ceiling [S1, unofficial]. Encode the occupancy grid into the compacted octree format [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import struct
from collections import deque
import math

def generate_collision_mesh(scene, selection_mask, params):
    """
    Generate collision mesh from selected splats.
    params: dict with keys 'detail_idx', 'wall_height', 'smoothing', 
            'ignore_transparent', 'opacity_cutoff'
    """
    # 1. Validate Scene
    if scene is None or scene.is_training:
        raise ValueError('Load a scene and pause training first.')

    # 2. Aggregate Selected Centers
    nodes = [n for n in scene.get_nodes() if n.type == 'SPLAT']
    total_gaussians = sum(n.gaussian_count for n in nodes)
    
    if len(selection_mask) != total_gaussians:
        raise ValueError('Selection layout does not match this scene.')

    world_centers = []
    
    offset = 0
    for node in nodes:
        if not scene.is_node_effectively_visible(node.id):
            offset += node.gaussian_count
            continue
            
        count = node.gaussian_count
        node_mask = selection_mask[offset:offset+count]
        offset += count
        
        if not node_mask.any():
            continue
            
        # Get raw data
        data = node.splat_data()
        means_raw = np.asarray(data.means_raw.cpu().numpy()).reshape(-1, 3)
        opacity_logits_raw = np.asarray(data.opacity_raw.cpu().numpy()).reshape(-1)
        
        # Handle deleted mask if present
        if data.has_deleted_mask():
            deleted = np.asarray(data.deleted.cpu().numpy()).reshape(-1).astype(bool)
            node_mask &= ~deleted
            
        # Filter transparent splats
        if params['ignore_transparent']:
            candidates = np.flatnonzero(node_mask)
            logits = opacity_logits_raw[candidates]
            # Stable threshold comparison for sigmoid-like activation
            # Assuming standard logistic function: 1 / (1 + exp(-x))
            # Threshold check: sigmoid(x) >= cutoff
            # x >= log(cutoff / (1 - cutoff))
            cutoff = params['opacity_cutoff']
            if 0 < cutoff < 1:
                threshold = math.log(cutoff / (1 - cutoff))
                keep = logits >= threshold
            else:
                keep = np.ones_like(logits, dtype=bool)
                
            node_mask[candidates[~keep]] = False
            
        if not node_mask.any():
            continue
            
        # Transform to world space
        selected_means = means_raw[node_mask]
        matrix = np.asarray(node.world_transform).reshape(4, 4)
        # Apply rotation and translation
        transformed = selected_means @ matrix[:3, :3].T + matrix[:3, 3]
        world_centers.append(transformed)

    if not world_centers:
        raise ValueError('No selected splats remain. Lower the opacity cutoff or disable the transparent-splat filter.')

    all_centers = np.concatenate(world_centers, axis=0)
    if len(all_centers) == 0:
        raise ValueError('No centers to voxelize.')

    # 3. Voxelization and Boundary Construction
    # Grid resolution based on detail preset
    details = [0.05, 0.1, 0.2, 0.5, 1.0]
    voxel_size = details[params['detail_idx']]
    
    # Determine bounds
    min_coords = np.min(all_centers, axis=0)
    max_coords = np.max(all_centers, axis=0)
    
    # Quantize to grid
    # Note: Native -Y is up. We treat Y as the vertical axis.
    # Grid indices
    min_grid = np.floor(min_coords / voxel_size).astype(int)
    max_grid = np.ceil(max_coords / voxel_size).astype(int)
    
    # Sparse grid to store max height per (x, z) and min height per (x, z)
    # Using a dict for sparsity
    footprint = {} # (x, z) -> {'min_y': float, 'max_y': float}
    
    for p in all_centers:
        x = int(math.floor(p[0] / voxel_size))
        y = int(math.floor(p[1] / voxel_size)) # Y is vertical
        z = int(math.floor(p[2] / voxel_size))
        
        key = (x, z)
        if key not in footprint:
            footprint[key] = {'min_y': y, 'max_y': y}
        else:
            footprint[key]['min_y'] = min(footprint[key]['min_y'], y)
            footprint[key]['max_y'] = max(footprint[key]['max_y'], y)
            
    # Calculate Ceiling Height
    # Highest selected splat center + wall height, measured upward along native -Y
    # In grid coordinates, "up" is increasing Y index if Y is up.
    # However, the source says "measured upward along native -Y". 
    # If native -Y is up, then increasing -Y means decreasing Y coordinate? 
    # Let's stick to the code logic: "highest selected splat center".
    # Usually "highest" means max Y coordinate.
    # The code calculates: max(values) + wall_height.
    # Let's assume standard Cartesian Y-up for calculation, then map to engine.
    # Source: "Ceiling height is highest selected splat center + wall height"
    
    global_max_y = np.max(all_centers[:, 1])
    ceiling_height_world = global_max_y + params['wall_height']
    ceiling_grid_y = int(math.floor(ceiling_height_world / voxel_size))
    
    # Identify Perimeter
    # A cell is on perimeter if any of its 4 horizontal neighbors are not in footprint
    perimeter_cells = set()
    for (x, z) in footprint:
        neighbors = [(x-1, z), (x+1, z), (x, z-1), (x, z+1)]
        if not any(n in footprint for n in neighbors):
            perimeter_cells.add((x, z))
            
    # Generate Voxels
    voxels = set()
    max_voxels = 5000000
    
    for (x, z), bounds in footprint.items():
        is_perimeter = (x, z) in perimeter_cells
        
        # Determine vertical range
        # Internal: fill from min to max
        # Perimeter: fill from min to ceiling
        low_y = bounds['min_y']
        
        if is_perimeter:
            high_y = ceiling_grid_y
        else:
            high_y = bounds['max_y']
            
        # Add voxels
        for y in range(low_y, high_y + 1):
            voxels.add((x, y, z))
            
        if len(voxels) > max_voxels:
            raise ValueError('Voxel export exceeds 5 million solid voxels. Choose coarser mesh detail.')
            
    # 4. Export (Voxel Format Example)
    # This section demonstrates the binary encoding logic from the source
    if not voxels:
        raise ValueError('No collision voxels to export.')
        
    origin = [min(p[a] for p in voxels) for a in range(3)]
    maximum = [max(p[a] for p in voxels) for a in range(3)]
    
    # Quantize origin/maximum to 4-voxel blocks for octree
    origin_q = [v // 4 * 4 for v in origin]
    maximum_q = [(v // 4 + 1) * 4 for v in maximum]
    
    blocks = {}
    for p in voxels:
        q = tuple(p[a] - origin_q[a] for a in range(3))
        key = tuple(v // 4 for v in q)
        bit = q[0]%4 + (q[1]%4)*4 + (q[2]%4)*16
        blocks[key] = blocks.get(key, 0) | (1 << bit)
        
    # ... (Octree construction logic as per source [S1, unofficial])
    # For brevity, the full octree serialization is omitted here as it is 
    # specific to the PlayCanvas voxel format and not the core meshing algorithm.
    
    return voxels, ceiling_height_world
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of selected splats for rasterization. Marching Cubes complexity depends on the volume of the dense grid (capped at 32 million samples) [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of solid voxels (capped at 5 million) and the resulting mesh triangles (capped at 2 million for smooth mode) [S1, unofficial].
*   **Performance:** The method is fast for sparse selections but can hit the 5 million voxel limit on dense, large-area ground selections. Coarser mesh detail (e.g., 0.5m or 1.0m) reduces voxel count quadratically [S1, unofficial].
*   **Strengths:** Handles floating objects and overhangs correctly by preserving vertical layers; explicit ceiling generation ensures watertight enclosures for navigation; smooth remeshing produces clean, manifold geometry suitable for physics engines [S1, unofficial].
*   **Limitations:** Sparse selection can leave floor gaps; the transparency filter may exclude dense low-opacity surfaces while keeping high-opacity floaters; thin geometry may distort during smoothing [S1, unofficial].
*   **When to use:** When a watertight collision mesh is required for navigation or physics in a game engine, specifically for environments with distinct ground and floating elements [S1, unofficial].
*   **When not to use:** When high-fidelity surface detail of the original splats is required (use raw splats or different voxelization); when the scene exceeds the 5 million voxel or 2 million triangle limits [S1, unofficial].

## Background
Selected-splat Collision Mesh Generation is a plugin for Lichtfeld Studio, developed by MisterE123. It was released as an unofficial extension to facilitate the creation of collision geometry for 3D Gaussian Splat scenes [S1, unofficial]. It builds on the concept of voxelizing Gaussian splats but adds specific logic for boundary wall and ceiling generation to create enclosed spaces [S1, unofficial]. The smooth remeshing component uses Lewiner's Marching Cubes implementation from scikit-image [S1, unofficial].

## Sources
[S1] MisterE123/lichtfeld-collision-mesh, https://github.com/MisterE123/lichtfeld-collision-mesh, fetched 2026-10-27 (unofficial)
