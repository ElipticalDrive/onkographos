# Robust Inside and Outside Solid Voxelization

Also called: Complete Polygonal Scene Voxelization

## Overview
Robust Inside and Outside Solid Voxelization is a method for determining the solid/empty status of octree cells in a voxelized mesh, specifically designed to handle non-watertight meshes with holes and intersecting geometry [S1, unofficial]. Unlike ray casting methods which require watertight meshes, this technique uses a heuristic model based on cube map rendering and visibility propagation to classify cells [S1, unofficial]. It sets itself apart by allowing "seed" cells to propagate their status to visible neighbors, overcoming the topological restrictions of traditional ray casting [S1, unofficial].

## How it works
The algorithm operates in three main stages: Octree Creation, Seed Cell Classification, and Status Propagation [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- A triangular mesh (OBJ format or similar).
- Maximum octree depth (typically 5; 7 or 8 for thin walls) [S1, unofficial].
- `MIN_INSIDE_FACES`: Integer constant, value 4 [S1, unofficial].
- `MIN_INSIDE_PERCENTAGE`: Float constant, value 0.03f (3%) [S1, unofficial].
- `Propagation Threshold`: Integer value, example 16 [S1, unofficial].

**Outputs:**
- An octree where each leaf node is classified as `Inside`, `Outside`, or `Intersecting` [S1, unofficial].

### Data Structures
- **Octree Cell**: Contains bounding box, status (`Unknown`, `Inside`, `Outside`, `Intersecting`), and propagation counters.
- **Propagation Counters**: For each `Unknown` cell, maintain two integer counters: `inside_count` and `outside_count`, initialized to 0 [S1, unofficial].

### Steps

1. **Create Octree**:
   - Subdivide the root cell containing the mesh until no triangle intersection occurs or the maximum depth is reached [S1, unofficial].
   - During subdivision, test triangle intersection against mesh bounds. Immediately mark cells outside mesh bounds as `Outside` without treating them as seed cells [S1, unofficial].
   - Mark cells intersecting triangles as `Intersecting` [S1, unofficial].

2. **Find Seed Cell**:
   - Select an `Unknown` cell to test as a potential seed [S1, unofficial].
   - Render a cube map centered in the cell with the near plane at the cell edge [S1, unofficial].
   - Use a single-pass pixel shader to color front faces blue and back faces red:
     ```glsl
     void main() {
         gl_FragColor = gl_FrontFacing ? vec4(0,0,1,1) : vec4(1,0,0,1);
     }
     ```
     [S1, unofficial]
   - For each of the 6 cube map sides, read back the GPU buffer and calculate the percentage of red (backface) pixels relative to total colored pixels [S1, unofficial].
   - Count sides where `backfacePercentage > MIN_INSIDE_PERCENTAGE` [S1, unofficial].
   - If `cubemap_sides_seeing_inside >= MIN_INSIDE_FACES`, classify the cell as `Inside` [S1, unofficial].
   - If `cubemap_sides_seeing_inside == 0`, classify the cell as `Outside` [S1, unofficial].
   - If neither, the cell cannot be solved exactly and remains `Unknown` [S1, unofficial].

3. **Propagate Seed Cell**:
   - For the classified seed cell, test every `Unknown` cell against the depth map and frustum of each cube map surface [S1, unofficial].
   - Project the 8 corners of the octree cell into the camera space of each cube face [S1, unofficial].
   - If any corner is closer than the depth value at that pixel, the cell is visible from the seed [S1, unofficial].
   - If visible, increment the corresponding counter (`inside_count` if seed is `Inside`, `outside_count` if seed is `Outside`) for the target cell [S1, unofficial].
   - If either counter reaches `Propagation Threshold`, commit the status to the target cell and stop processing it [S1, unofficial].
   - **Note**: Only seed cells propagate status. Cells that receive propagated status do not become new seed cells [S1, unofficial].

4. **Repeat Loop**:
   - Repeat steps 2 and 3 until all cells are classified or no further propagation is possible [S1, unofficial].
   - Handle undetermined statuses to prevent infinite loops [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def robust_voxelization(mesh, max_depth):
    octree = build_octree(mesh, max_depth)
    # Mark cells outside mesh bounds as Outside
    for cell in octree.leaves:
        if not intersects_mesh_bounds(cell):
            cell.status = CellStatus.Outside
            continue
        if intersects_triangles(cell):
            cell.status = CellStatus.Intersecting
            continue
        cell.status = CellStatus.Unknown
        cell.inside_count = 0
        cell.outside_count = 0

    seed_cells = []
    unknown_cells = [c for c in octree.leaves if c.status == CellStatus.Unknown]

    while unknown_cells:
        # Find Seed Cell
        seed = find_seed_cell(unknown_cells)
        if seed is None:
            break # No more seeds can be found
        
        seed_cells.append(seed)
        unknown_cells.remove(seed)
        
        # Propagate Seed Cell
        propagate_status(seed, unknown_cells)

    return octree

def find_seed_cell(unknown_cells):
    for cell in unknown_cells:
        cubemap_sides_seeing_inside = 0
        for i in range(6):
            render_cube_map_side(i, cell)
            backface_percentage = calculate_backface_percentage(i)
            if backface_percentage > MIN_INSIDE_PERCENTAGE:
                cubemap_sides_seeing_inside += 1
        
        if cubemap_sides_seeing_inside >= MIN_INSIDE_FACES:
            cell.status = CellStatus.Inside
            return cell
        elif cubemap_sides_seeing_inside == 0:
            cell.status = CellStatus.Outside
            return cell
    return None

def propagate_status(seed, unknown_cells):
    for cell in unknown_cells:
        if is_visible_from_seed(seed, cell):
            if seed.status == CellStatus.Inside:
                cell.inside_count += 1
                if cell.inside_count >= Propagation_Threshold:
                    cell.status = CellStatus.Inside
                    unknown_cells.remove(cell)
            elif seed.status == CellStatus.Outside:
                cell.outside_count += 1
                if cell.outside_count >= Propagation_Threshold:
                    cell.status = CellStatus.Outside
                    unknown_cells.remove(cell)

def is_visible_from_seed(seed, cell):
    # Project 8 corners of cell into seed's cube map camera space
    # Check if any corner is closer than depth value at that pixel
    # Return True if visible
    pass
```

### Edge Cases and Pitfalls
- **Propagation Limitation**: Only seed cells propagate status. Cells that receive propagated status do not become new seed cells and do not propagate further [S1, unofficial].
- **Holes and Intersecting Geometry**: The method handles these cases where ray casting fails [S1, unofficial].
- **Mesh Bounds**: Cells outside mesh bounds are immediately marked `Outside` but do not act as seeds [S1, unofficial].
- **Infinite Loops**: Ensure code handles undetermined statuses to prevent infinite loops [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Strengths**: Handles non-watertight meshes with holes and intersecting geometry [S1, unofficial].
- **Limitations**: Only seed cells propagate status, which may limit propagation efficiency [S1, unofficial].
- **When to Use**: Generating simplified occluders for hierarchical Z-buffer occlusion culling in game development tools [S1, unofficial].
- **When Not to Use**: When watertight meshes are guaranteed and ray casting is preferred for simplicity [S1, unofficial].

## Background
- **Origin**: Introduced in the 'Complete Polygonal Scene Voxelization' paper published in 2002 [S1, unofficial].
- **History**: Developed to replace ray casting based solid voxelization methods that required watertight meshes [S1, unofficial].
- **Usage**: Used for generating simplified occluders for hierarchical Z-buffer occlusion culling in game development tools [S1, unofficial].

## Sources
[S1] Robust Inside and Outside Solid Voxelization, https://jahej.com/alt/2011_09_24_robust-inside-and-outside-solid-voxelization.html, fetched 2026-10-29 (unofficial)
