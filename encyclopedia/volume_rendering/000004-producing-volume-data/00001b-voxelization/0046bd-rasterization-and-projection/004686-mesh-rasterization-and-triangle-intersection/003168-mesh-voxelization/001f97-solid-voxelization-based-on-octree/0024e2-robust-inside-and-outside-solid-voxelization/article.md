# Robust Inside and Outside Solid Voxelization

Also called: Complete Polygonal Scene Voxelization

## Overview
Robust Inside and Outside Solid Voxelization is a method for determining the solid/empty status of octree cells in a voxelized mesh, specifically designed to handle non-watertight meshes with holes and intersecting geometry [S1, unofficial]. Unlike ray casting methods which require watertight meshes, this technique uses a heuristic model based on cube map rendering and visibility propagation to classify cells [S1, unofficial]. It sets itself apart by allowing "seed" cells to propagate their status to visible neighbors, overcoming the topological restrictions of traditional ray casting [S1, unofficial]. It builds on [Solid Voxelization based on Octree](2062-solid-voxelization-based-on-octree.md) by replacing the ray-casting sign test with a GPU-accelerated cube map heuristic.

## How it works
The algorithm operates in three main stages: Octree Creation, Seed Cell Classification, and Status Propagation [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- A triangular mesh [S1, unofficial].
- Maximum octree depth (typically 5; 7 or 8 for thin walls) [S1, unofficial].
- `MIN_INSIDE_FACES`: Integer constant, value 4 [S1, unofficial].
- `MIN_INSIDE_PERCENTAGE`: Float constant, value 0.03 (3%) [S1, unofficial].
- `PROPAGATION_THRESHOLD`: Integer value, example 16 [S1, unofficial].

**Outputs:**
- An octree where each leaf node is classified as `Inside`, `Outside`, or `Intersecting` [S1, unofficial].

### Data Structures
- **Octree Cell**: Contains bounding box, status (`Unknown`, `Inside`, `Outside`, `Intersecting`), and propagation counters [S1, unofficial].
- **Propagation Counters**: For each `Unknown` cell, maintain two integer counters: `inside_count` and `outside_count`, initialized to 0 [S1, unofficial].

### Steps

1. **Create Octree**:
   - Subdivide the root cell containing the mesh until no triangle intersection occurs or the maximum depth is reached [S1, unofficial].
   - During subdivision, test triangle intersection against mesh bounds [S1, unofficial].
   - Immediately mark cells outside mesh bounds as `Outside` without treating them as seed cells [S1, unofficial].
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
   - If either counter reaches `PROPAGATION_THRESHOLD`, commit the status to the target cell and stop processing it [S1, unofficial].
   - **Note**: Only seed cells propagate status. Cells that receive propagated status do not become new seed cells [S1, unofficial].

4. **Repeat Loop**:
   - Repeat steps 2 and 3 until all cells are classified or no further propagation is possible [S1, unofficial].
   - Handle undetermined statuses to prevent infinite loops [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants from evidence
MIN_INSIDE_FACES = 4
MIN_INSIDE_PERCENTAGE = 0.03
PROPAGATION_THRESHOLD = 16

class CellStatus:
    UNKNOWN = 0
    INSIDE = 1
    OUTSIDE = 2
    INTERSECTING = 3

class OctreeCell:
    def __init__(self):
        self.bounding_box = None
        self.status = CellStatus.UNKNOWN
        self.inside_count = 0
        self.outside_count = 0

def robust_voxelization(mesh, max_depth):
    # Standard octree subdivision: recursively split AABB until triangle intersection count is 0 or depth == max_depth
    octree = build_octree(mesh, max_depth)
    for cell in octree.leaves:
        # Standard AABB vs mesh bounding box test
        if not intersects_mesh_bounds(cell):
            cell.status = CellStatus.OUTSIDE
            continue
        # Standard AABB vs triangle intersection test (e.g., Möller-Trumbore)
        if intersects_triangles(cell):
            cell.status = CellStatus.INTERSECTING
            continue
        cell.status = CellStatus.UNKNOWN
        cell.inside_count = 0
        cell.outside_count = 0

    unknown_cells = [c for c in octree.leaves if c.status == CellStatus.UNKNOWN]
    seed_cells = []

    while unknown_cells:
        seed = find_seed_cell(unknown_cells)
        if seed is None:
            break
        seed_cells.append(seed)
        unknown_cells.remove(seed)
        propagate_status(seed, unknown_cells)

    return octree

def find_seed_cell(unknown_cells):
    for cell in unknown_cells:
        cubemap_sides_seeing_inside = 0
        for i in range(6):
            # Render scene from cell center, 6 faces, near plane at cell edge
            render_cube_map_side(i, cell)
            # Read back GPU buffer, count red pixels / total colored pixels
            backface_percentage = calculate_backface_percentage(i)
            if backface_percentage > MIN_INSIDE_PERCENTAGE:
                cubemap_sides_seeing_inside += 1

        if cubemap_sides_seeing_inside >= MIN_INSIDE_FACES:
            cell.status = CellStatus.INSIDE
            return cell
        elif cubemap_sides_seeing_inside == 0:
            cell.status = CellStatus.OUTSIDE
            return cell
    return None

def propagate_status(seed, unknown_cells):
    for cell in unknown_cells:
        if is_visible_from_seed(seed, cell):
            if seed.status == CellStatus.INSIDE:
                cell.inside_count += 1
                if cell.inside_count >= PROPAGATION_THRESHOLD:
                    cell.status = CellStatus.INSIDE
                    unknown_cells.remove(cell)
            elif seed.status == CellStatus.OUTSIDE:
                cell.outside_count += 1
                if cell.outside_count >= PROPAGATION_THRESHOLD:
                    cell.status = CellStatus.OUTSIDE
                    unknown_cells.remove(cell)

def is_visible_from_seed(seed, cell):
    # Project 8 corners of cell into seed's cube map camera space for each face
    for face_idx in range(6):
        for corner in cell.bounding_box.corners:
            screen_pos = project_to_screen(corner, seed.cube_map_view_matrix[face_idx], seed.cube_map_proj_matrix[face_idx])
            depth_val = seed.cube_map_depth_buffer[face_idx][screen_pos.x][screen_pos.y]
            if corner.z < depth_val:
                return True
    return False

# Standard operation definitions
def build_octree(mesh, max_depth):
    root = OctreeCell()
    root.bounding_box = mesh.bounding_box
    queue = [root]
    while queue:
        node = queue.pop(0)
        if node.depth < max_depth and intersects_triangles(node):
            node.children = split_aabb(node.bounding_box)
            queue.extend(node.children)
    return root

def intersects_mesh_bounds(cell):
    return cell.bounding_box.intersects(mesh.bounding_box)

def intersects_triangles(cell):
    return cell.bounding_box.intersects_any_triangle(mesh.triangles)

def render_cube_map_side(face_idx, cell):
    # Standard GPU render: set camera at cell center, look along face normal, near plane at cell edge
    pass # Standard graphics pipeline call

def calculate_backface_percentage(face_idx):
    # Standard GPU readback: count pixels where color == (1,0,0) / total pixels
    pass # Standard GPU readback call

def project_to_screen(corner, view_matrix, proj_matrix):
    # Standard math: clip space = proj * view * corner; perspective divide; viewport transform
    pass # Standard math operation
```

### Edge Cases and Pitfalls
- **Propagation Limitation**: Only seed cells propagate status. Cells that receive propagated status do not become new seed cells and do not propagate further [S1, unofficial].
- **Holes and Intersecting Geometry**: The method handles these cases where ray casting fails [S1, unofficial].
- **Mesh Bounds**: Cells outside mesh bounds are immediately marked `Outside` but do not act as seeds [S1, unofficial].
- **Infinite Loops**: Ensure code handles undetermined statuses to prevent infinite loops [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources [S1, unofficial].
- **Memory Complexity**: Not given in the sources [S1, unofficial].
- **Measured Performance**: Not given in the sources [S1, unofficial].
- **Strengths**: Handles non-watertight meshes with holes and intersecting geometry [S1, unofficial].
- **Limitations**: Only seed cells propagate status, which may limit propagation efficiency [S1, unofficial].
- **When to Use**: Generating simplified occluders for hierarchical Z-buffer occlusion culling in game development tools [S1, unofficial].

## Background
- **Origin**: Introduced in the 'Complete Polygonal Scene Voxelization' paper published in 2002 [S1, unofficial].
- **History**: Developed to replace ray casting based solid voxelization methods that required watertight meshes [S1, unofficial].
- **Usage**: Used for generating simplified occluders for hierarchical Z-buffer occlusion culling in game development tools [S1, unofficial].

## Sources
[S1] Robust Inside and Outside Solid Voxelization, https://jahej.com/alt/2011_09_24_robust-inside-and-outside-solid-voxelization.html, fetched 2026-10-29 (unofficial)
