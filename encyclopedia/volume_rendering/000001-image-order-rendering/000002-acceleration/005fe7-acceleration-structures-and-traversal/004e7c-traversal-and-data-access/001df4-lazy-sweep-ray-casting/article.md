# Lazy Sweep Ray Casting

Also called: Lazy Sweep, Sweep Plane Ray Casting

## Overview
Lazy Sweep Ray Casting is a fast algorithm for rendering general irregular grids, including disconnected and nonconvex domains with holes [S2, unofficial]. It solves the problem of efficiently traversing unstructured meshes by exploiting spatial coherence between scanlines, replacing expensive 3D ray-casting with simpler 2D regular grid ray-casting [S1, unofficial]. It is distinguished from its siblings by its use of a sweep plane orthogonal to the viewing plane to maintain an Active Cell List (ACL), allowing it to handle complex connectivity without explicit storage [S1, unofficial].

## How it works
**Inputs:** An unstructured irregular grid (mesh) defined by vertices and cells; a viewing direction (e.g., along the y-axis); image resolution (pixel rows and columns) [S1, unofficial].
**Outputs:** A rendered image [S1, unofficial].

**Data Structures:**
1.  **Active Cell List (ACL):** A dynamic list storing the set of cells currently intersected by the sweep plane [S1, unofficial].
2.  **Regular Grid (Hash Table):** A 2D structure mapping pixel columns to the subset of cells in the ACL that are traversed at the current sweep position [S1, unofficial].

**Steps:**
1.  **Event Determination:** Identify all "events" in the sweep. Events are defined by vertices in the dataset and y-values corresponding to pixel rows in the image [S1, unofficial]. Sort these events by their y-coordinate [S1, unofficial].
2.  **Sweep and Update:** Iterate through the sorted events [S1, unofficial].
    *   If the event is a **vertex**: Update the ACL by adding cells that become intersected by the sweep plane as it passes the vertex, and removing cells that are no longer intersected [S1, unofficial].
    *   If the event is a **pixel row**:
        1.  Construct the Regular Grid (Hash Table) for the current ACL, storing the ACL elements for traversal [S1, unofficial].
        2.  For each pixel column $x$ in the row: cast a ray corresponding to the x-value through the Regular Grid to retrieve the intersecting cells from the ACL [S1, unofficial].
        3.  Perform volume rendering (e.g., ray-marching or cell projection) for the ray segment through the retrieved cells [S1, unofficial].
3.  **Repeat:** Continue until all pixel rows are processed [S1, unofficial].

**Formulas:**
*   Ray parameterization: $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the direction [G].
*   Intersection with cell boundaries: Determined by the geometry of the irregular grid cells [G].
*   Sweep plane position: $y = y_{event}$ [S1, unofficial].
*   Image resolution: Determines the spacing of pixel row events [S1, unofficial].

**Parameters:**
*   Sweep direction: Typically orthogonal to the viewing plane (e.g., y-axis) [S1, unofficial].
*   Image resolution: Determines the spacing of pixel row events [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Disconnected/Nonconvex Grids:** The method handles these naturally by maintaining the ACL, which tracks all intersecting cells regardless of domain connectivity [S2, unofficial].
*   **Precision:** Care must be taken when determining cell intersections with the sweep plane to avoid gaps or overlaps due to floating-point errors [G].
*   **Hash Table Construction:** The cost of building the Regular Grid for each row must be balanced against the savings in ray casting [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lazy_sweep_ray_casting(mesh, image_height, image_width, view_direction='y'):
    # 1. Determine events: vertices and pixel row y-coordinates
    events = []
    for vertex in mesh.vertices:
        events.append(('vertex', vertex.y, vertex.id))
    for row in range(image_height):
        y_row = (row + 0.5) / image_height  # [G] Standard normalized mapping
        events.append(('row', y_row, row))
    
    # Sort events by y-coordinate
    events.sort(key=lambda e: e[1])
    
    acl = []  # Active Cell List
    image = [[0.0] * image_width for _ in range(image_height)]
    
    for event_type, y, id in events:
        if event_type == 'vertex':
            # Update ACL: add cells starting at this vertex, remove cells ending at this vertex
            acl = update_acl(acl, mesh, id)
        elif event_type == 'row':
            # Build Regular Grid (Hash Table) from ACL
            regular_grid = build_regular_grid(acl, image_width)
            
            # Cast rays for each pixel in the row
            for col in range(image_width):
                cells = regular_grid.get(col, [])
                ray_origin = get_ray_origin(col, row, image_width, image_height)
                ray_direction = get_ray_direction(view_direction)
                color = cast_ray_through_cells(ray_origin, ray_direction, cells)
                image[row][col] = color
                
    return image

def update_acl(acl, mesh, vertex_id):
    # [G] Standard implementation: identify cells connected to the vertex.
    # Add cells whose minimum y equals this vertex y.
    # Remove cells whose maximum y equals this vertex y.
    new_acl = []
    for cell in acl:
        if cell.max_y > vertex.y:  # [G] Keep cells still intersected
            new_acl.append(cell)
    for cell in mesh.get_cells(vertex_id):
        if cell.min_y == vertex.y:  # [G] Add cells starting at this sweep position
            new_acl.append(cell)
    return new_acl

def build_regular_grid(acl, image_width):
    # [G] Standard implementation: map cells to pixel columns they span
    grid = {}
    for cell in acl:
        col_start = max(0, int(cell.x_min * image_width))
        col_end = min(image_width - 1, int(cell.x_max * image_width))
        for col in range(col_start, col_end + 1):
            if col not in grid:
                grid[col] = []
            grid[col].append(cell)
    return grid

def get_ray_origin(col, row, width, height):
    # [G] Standard mapping from pixel indices to world coordinates
    return (col / width, row / height, 0.0)

def get_ray_direction(view_direction):
    # [G] Standard viewing direction vector
    if view_direction == 'y':
        return (0.0, 1.0, 0.0)
    return (0.0, 0.0, 1.0)

def cast_ray_through_cells(origin, direction, cells):
    # [G] Standard volume rendering accumulation
    color = [0.0, 0.0, 0.0, 0.0]
    for cell in cells:
        # [G] Intersect ray with cell, sample, and composite
        pass
    return color
```

## Subtopics

### [Garrity Ray Casting](7132-garrity-ray-casting.md)
Garrity Ray Casting is a method for rendering unstructured meshes by exploiting intrinsic mesh connectivity to optimize ray traversal. It solves the problem of efficiently traversing non-convex domains where rays may exit and re-enter the mesh multiple times.

## Operating characteristics
*   **Time Complexity:** Substantially faster (by up to two orders of magnitude) than other software-based algorithms for irregular grids [S2, unofficial]. The complexity depends on the number of cells and the spatial coherence [G].
*   **Memory Complexity:** Low memory consumption, as it does not require explicit connectivity storage or large resampled volumes [S1, unofficial][S2, unofficial].
*   **Strengths:** Generality, depth-sorting accuracy, simplicity of implementation, portability (no hardware dependencies), and exploitation of spatial coherence [S1, unofficial][S2, unofficial].
*   **Limitations:** Performance may degrade if spatial coherence is low (e.g., highly disconnected grids) [G]. The cost of maintaining the ACL and Regular Grid for each row must be managed [G].
*   **When to use:** Ideal for rendering general irregular grids, especially when explicit connectivity is not available or when hardware acceleration is not used [S1, unofficial][S2, unofficial].
*   **When not to use:** If the grid is highly structured and a simpler DDA or texture-based method is applicable [G].

## Background
Lazy Sweep Ray Casting was introduced by Cláudio T. Silva and Joseph S. B. Mitchell in 1997 [S2, unofficial]. It is based on the sweep-plane paradigm, which has been used in other volume rendering methods, often with hardware assistance [S1, unofficial]. The method was developed to address the limitations of resampling irregular grids into regular volumes, which is computationally expensive, memory-intensive, and time-consuming [S1, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
