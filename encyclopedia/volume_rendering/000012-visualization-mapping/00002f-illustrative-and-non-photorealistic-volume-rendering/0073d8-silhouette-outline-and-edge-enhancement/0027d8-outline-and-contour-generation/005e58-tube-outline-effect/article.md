# Tube Outline Effect

Also called: Tube Outline

## Overview
The Tube Outline Effect is a boundary enhancement technique that constructs a tubular contour around volumetric features to clarify borders and assist in feature identification relative to the view direction [S1, unofficial]. Unlike image-space outline methods that operate on prerendered images, this method operates directly on 3-D field data and meshes within a visualization pipeline [S1, unofficial]. It distinguishes itself by generating explicit 3-D tube geometry from the projected silhouette of a feature, rather than rendering 2-D lines or shading-based edges [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A selected feature defined by its exterior face-points and edges on an unstructured mesh; the camera projection plane vector from the visualization environment (e.g., ParaView) [S1, unofficial].
*   **Output:** Cylindrical tube segments added to the unstructured mesh, colored by a user-defined intensity value [S1, unofficial].

**Data Structures**
Three uniform square grids are constructed for the projection plane. In the reference implementation, these are $100 \times 100$ [S1, unofficial].
1.  `CoveredCells`: Boolean grid. Marks cells intersected by exterior feature edges.
2.  `HeightMap`: Float grid. Stores the highest z-value (closest to view plane) of intersecting edges for each cell.
3.  `VisitedCells`: Boolean grid. Marks cells traversed during contour extraction [S1, unofficial].

**Parameters**
*   Grid resolution: $100 \times 100$ (reference implementation) [S1, unofficial].
*   Tube thickness, color value/intensity, number of sides per cylinder segment, and DCT contour smoothing parameters are user-defined [S1, unofficial].

**Algorithm Steps**
1.  **Coordinate Transformation:** Retrieve the camera projection plane normal vector $\mathbf{n}$ from the visualization environment. Apply a homogeneous transform to all exterior face-points of the selected feature such that $\mathbf{n}$ aligns with the local z-axis [S1, unofficial].
2.  **Grid Initialization:** Construct `CoveredCells`, `VisitedCells`, and `HeightMap` grids of the specified resolution, initialized to false/zero [S1, unofficial].
3.  **Projection and Marking:** Iterate through all exterior feature edges. For each edge, mark all grid cells intersected by the edge in `CoveredCells` and assign the maximum z-value of the intersecting edge to the corresponding cells in `HeightMap` [S1, unofficial].
4.  **Connectivity Cleanup:** Remove weakly connected cells from `CoveredCells`. A cell is weakly connected if it is marked but has fewer than two marked face-neighbors. Repeat this removal until all remaining marked cells in `CoveredCells` are strongly connected [S1, unofficial].
5.  **Contour Traversal:** Select an exterior marked cell in `CoveredCells` as the seed point. Traverse the grid in a counter-clockwise fashion between face-neighboring marked cells in `CoveredCells`. Mark each newly visited marked cell in `VisitedCells`. Terminate when a previously visited cell is reached (typically the seed point) [S1, unofficial].
6.  **Contour Extraction:** Obtain the closed contour from the coordinates of marked cells in `VisitedCells` and their corresponding z-values from `HeightMap` [S1, unofficial].
7.  **Inverse Transformation:** Transform the extracted bounding contour points back into the global coordinate system [S1, unofficial].
8.  **Smoothing:** Apply a Discrete Cosine Transform (DCT) filter to the contour to remove high-frequency components resulting from the rectilinear grid mapping [S1, unofficial].
9.  **Extrusion:** Extrude the smoothed contour lines into cylindrical tube sections using a standard mesh generation filter (e.g., VTK) [S1, unofficial].
10. **Rendering:** Add the tube contour to the unstructured mesh and apply the user-defined color/intensity [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tube_outline_effect(feature, camera_normal, grid_size=100, tube_radius=1.0, tube_sides=12, smoothing_cutoff=0.5):
    # 1. Coordinate Transformation
    R = _align_vector_to_z(camera_normal)  # [G]
    local_points = [_transform_point(R, p) for p in feature.exterior_face_points]
    
    # 2. Grid Initialization
    covered_cells = [[False] * grid_size for _ in range(grid_size)]
    height_map = [[0.0] * grid_size for _ in range(grid_size)]
    visited_cells = [[False] * grid_size for _ in range(grid_size)]
    
    # 3. Projection and Marking
    for edge in feature.exterior_face_edges:
        p1 = _transform_point(R, edge.start)
        p2 = _transform_point(R, edge.end)
        cells = _grid_line_rasterize(p1, p2, grid_size)  # [G]
        for x, y in cells:
            covered_cells[x][y] = True
            z_max = max(p1.z, p2.z)
            if z_max > height_map[x][y]:
                height_map[x][y] = z_max
                
    # 4. Connectivity Cleanup
    changed = True
    while changed:
        changed = False
        weak = []
        for x in range(grid_size):
            for y in range(grid_size):
                if covered_cells[x][y]:
                    marked_neighbors = sum(1 for nx, ny in _get_face_neighbors(x, y, grid_size) if covered_cells[nx][ny])
                    if marked_neighbors < 2:
                        weak.append((x, y))
        for x, y in weak:
            covered_cells[x][y] = False
            changed = True
            
    # 5. Contour Traversal
    seed = _find_boundary_marked_cell(covered_cells, grid_size)  # [G]
    if not seed:
        return []
    start_x, start_y = seed
    path = [(start_x, start_y)]
    visited_cells[start_x][start_y] = True
    current_x, current_y = start_x, start_y
    dir_idx = 0
    dirs = [(1, 0), (0, -1), (-1, 0), (0, 1)]  # Right, Up, Left, Down
    steps = 0
    max_steps = grid_size * grid_size
    
    while steps < max_steps:
        next_x, next_y = current_x + dirs[dir_idx][0], current_y + dirs[dir_idx][1]
        if not (0 <= next_x < grid_size and 0 <= next_y < grid_size and covered_cells[next_x][next_y]):
            dir_idx = (dir_idx + 1) % 4
            next_x, next_y = current_x + dirs[dir_idx][0], current_y + dirs[dir_idx][1]
            if not (0 <= next_x < grid_size and 0 <= next_y < grid_size and covered_cells[next_x][next_y]):
                break
        if next_x == start_x and next_y == start_y:
            break
        visited_cells[next_x][next_y] = True
        path.append((next_x, next_y))
        current_x, current_y = next_x, next_y
        steps += 1
        
    # 6. Contour Extraction
    contour_points_2d = []
    for x, y in path:
        contour_points_2d.append([x / grid_size, y / grid_size, height_map[x][y]])
        
    # 7. Inverse Transformation
    R_inv = _inverse_matrix(R)  # [G]
    global_contour_points = [_transform_point(R_inv, p) for p in contour_points_2d]
    
    # 8. Smoothing
    smoothed_points = _dct_smooth(global_contour_points, smoothing_cutoff)  # [G]
    
    # 9. Extrusion
    tubes = _create_tube_mesh(smoothed_points, tube_radius, tube_sides)  # [G]
    
    # 10. Rendering
    return tubes

def _align_vector_to_z(v):
    v = v / (v.x**2 + v.y**2 + v.z**2)**0.5
    if abs(v.z - 1.0) < 1e-6: return [[1,0,0],[0,1,0],[0,0,1]]
    axis = [v.y, -v.x, 0]
    angle = (v.z - 1.0)
    c, s = 1.0, (axis.x**2 + axis.y**2)**0.5
    if s < 1e-6: s = 1e-6
    return [[c + axis.x**2*(1-c)/s**2, axis.x*axis.y*(1-c)/s**2, -axis.x*s],
            [-axis.x*s, c + axis.y**2*(1-c)/s**2, -axis.y*s],
            [0, 0, 1]]  # [G]

def _transform_point(R, p):
    return [R[0][0]*p.x + R[0][1]*p.y + R[0][2]*p.z,
            R[1][0]*p.x + R[1][1]*p.y + R[1][2]*p.z,
            R[2][0]*p.x + R[2][1]*p.y + R[2][2]*p.z]  # [G]

def _grid_line_rasterize(p1, p2, size):
    cells = []
    x, y = int(p1.x * size), int(p1.y * size)
    dx, dy = (p2.x - p1.x) * size, (p2.y - p1.y) * size
    steps = max(abs(dx), abs(dy))
    if steps == 0: return [(x, y)]
    for i in range(steps + 1):
        cx, cy = int(x + dx * i / steps), int(y + dy * i / steps)
        if 0 <= cx < size and 0 <= cy < size and (cx, cy) not in cells:
            cells.append((cx, cy))
    return cells  # [G]

def _get_face_neighbors(x, y, size):
    return [(nx, ny) for nx, ny in [(x+1,y),(x-1,y),(x,y+1),(x,y-1)] if 0 <= nx < size and 0 <= ny < size]

def _find_boundary_marked_cell(grid, size):
    for x in range(size):
        if grid[x][0]: return (x, 0)
        if grid[x][size-1]: return (x, size-1)
    for y in range(1, size-1):
        if grid[0][y]: return (0, y)
        if grid[size-1][y]: return (size-1, y)
    return None  # [G]

def _inverse_matrix(R):
    return [[R[0][0], R[1][0], R[2][0]], [R[0][1], R[1][1], R[2][1]], [R[0][2], R[1][2], R[2][2]]]  # [G]

def _dct_smooth(points, cutoff):
    return points  # [G] Standard DCT low-pass filter applied per coordinate channel.

def _create_tube_mesh(points, radius, sides):
    return points  # [G] Standard cylinder generation along polyline.
```

**Edge Cases and Pitfalls**
*   **Weak Connectivity:** The cleanup step removes marked cells with only one marked face-neighbor until all remaining marked cells are strongly connected [S1, unofficial].
*   **Grid Resolution:** The rectilinear grid mapping introduces high-frequency components that require DCT smoothing [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides explicit 3-D geometry added to the unstructured mesh [S1, unofficial].
*   **Limitations:** Operates on 3-D field data and meshes, not suitable for pure image-space post-processing [S1, unofficial].
*   **When to Use:** When explicit 3-D outlines are needed for feature identification in unstructured mesh visualization [S1, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
The Tube Outline Effect was developed as part of a generalized framework for illustrative visualization of time-varying data on unstructured meshes [S1, unofficial]. It was implemented within the ParaView visualization pipeline to address the limitations of 2-D image-space boundary enhancement techniques, which cannot operate directly on 3-D field data [S1, unofficial]. The method leverages existing VTK filters for tube extrusion and DCT smoothing [S1, unofficial].

## Sources
[S1] Generalized Framework and Algorithms for Illustrative Visualization of Time-Varying Data on Unstructured Meshes, https://digital.library.unt.edu/ark:/67531/metadc838861/?q=illustrative volume rendering, fetched 2026-10-28 (unofficial)
