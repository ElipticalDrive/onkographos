# Dual Contouring (2D Grid Implementation)

Also called: 2D Dual Contouring, 2D Dual Contouring in JS, QEF Marching Squares, SimpleGrid Dual Contouring

## Overview
Dual Contouring (2D Grid Implementation) extracts a polygonal mesh from a 2D scalar field defined on a uniform grid by placing a single vertex inside each grid cell that intersects the isosurface, using Quadratic Error Function (QEF) minimization to determine the optimal vertex position [S1, S2, S3, unofficial]. Unlike Marching Squares, which connects intersection points on cell edges with straight lines, this method constructs a dual contour within each cell, allowing for smoother, more accurate representations of curved surfaces [S1, S3, unofficial]. It builds on [Dual Contouring 2D](3668-dual-contouring-2d.md) by providing a concrete, iterative implementation on a simple uniform grid structure.

## How it works
**Inputs:**
- A 2D scalar field defined on a uniform grid of size `gridWidth` x `gridHeight` [S3, unofficial].
- Grid parameters: `WIDTH` (512 pixels), `HEIGHT` (512 pixels), `RESOLUTION` (cell size in world units, typically `WIDTH / 64`) [S1, unofficial].
- Surface function (e.g., noise-modulated circle) to evaluate material index or density at any point [S1, unofficial].

**Outputs:**
- A set of mesh vertices, one per intersected cell, stored in `positions[x][y]` [S3, unofficial].
- A set of mesh edges (faces in 2D), stored in `edges` array, connecting vertices across shared crossed edges [S1, S2, unofficial].

**Data Structures:**
- `values[x][y]`: 2D array storing scalar field values at grid vertices. Values are 0 for empty (air) and 1 for material [S3, unofficial].
- `positions[x][y]`: 2D array storing the computed vertex position for grid cell `(x,y)` as `{x: float, y: float}`. Initialized to `{x:0.5, y:0.5}` [S3, unofficial].
- `verticalEdges[x][y]`: 2D array of objects representing vertical edges. Fields: `p` (float, intersection parameter 0-1), `nx` (float, normal x component), `ny` (float, normal y component). Initialized to `{p:0, nx:1, ny:1}` for no intersection [S3, unofficial].
- `horizontalEdges[x][y]`: 2D array of objects representing horizontal edges. Fields: `p` (float, intersection parameter 0-1), `nx` (float, normal x component), `ny` (float, normal y component). Initialized to `{p:0, nx:1, ny:1}` for no intersection [S3, unofficial].
- `points[ij]`: 2D array storing material indices at grid points [S1, unofficial].
- `cells[i][j]`: 2D array storing cell data, including edges, origin, vertex, and residual [S1, unofficial].
- `faces`: Array storing mesh faces (line segments) [S1, unofficial].
- `ibuffer`: Integer array storing cell indices for each grid vertex, initialized to -1 [S2, unofficial].

**Algorithm Steps:**

1. **Initialize Grid and Arrays:**
   - Calculate `nCells = size / resolution` and `nPoints = nCells + 1` [S1, unofficial].
   - Initialize `values`, `positions`, `verticalEdges`, `horizontalEdges`, `points`, `cells`, and `faces` arrays with zeros or default values [S1, S3, unofficial].
   - For `i` from 0 to `gridWidth-1` and `j` from 0 to `gridHeight-1`:
     - Set `values[i][j] = 0` [S3, unofficial].
     - Set `positions[i][j] = {x:0.5, y:0.5}` [S3, unofficial].
     - Set `verticalEdges[i][j] = {p:0, nx:1, ny:1}` [S3, unofficial].
     - Set `horizontalEdges[i][j] = {p:0, nx:1, ny:1}` [S3, unofficial].

2. **Generate Grid Point Data:**
   - For each grid point `(i,j)`:
     - Calculate world position `xy = origin + [i, j] * resolution` [S1, unofficial].
     - Evaluate `surface.getIndex(xy)` to get material index [S1, unofficial].
     - Store in `points[i][j]` [S1, unofficial].

3. **Generate Grid Edge Data (Intersections and Normals):**
   - Iterate over all horizontal and vertical edges of the grid.
   - For each edge, get world position `xy` and the two adjacent grid point values `v1` and `v2` [S1, unofficial].
   - If `v1 != v2` (sign change):
     - **Binary Search for Intersection:**
       - Initialize `err = resolution`, `sxy = xy` [S1, unofficial].
       - While `err > MAX_ERR` (0.1):
         - Evaluate `value = surface.getIndex(sxy)` [S1, unofficial].
         - Determine sign: if `value == v2` then `sign = -1` else `sign = 1` [S1, unofficial].
         - Update `sxy = sxy + step * sign * err`, where `step` is the edge direction vector [S1, unofficial].
         - `err /= 2` [S1, unofficial].
       - Store `intercept = sxy - xy` [S1, unofficial].
     - **Estimate Normal:**
       - Calculate `dx = surface.get(sxy + [d,0]) - surface.get(sxy + [-d,0])` and `dy = surface.get(sxy + [0,d]) - surface.get(sxy + [0,-d])` where `d=0.01` [S1, unofficial].
       - Normalize `[dx, dy]` to get normal `[nx, ny]` [S1, unofficial].
     - Store `intercept`, `nx`, `ny` in the corresponding `horizontalEdges` or `verticalEdges` array [S1, S3, unofficial].
   - If no sign change, leave edge data as default (no intersection) [S3, unofficial].

4. **Generate Mesh Vertices (QEF Minimization):**
   - For each cell `(x,y)`:
     - Check if `gridHasSignChange(x,y)` is true (i.e., the cell contains the isosurface) [S3, unofficial].
     - If true:
       - **Collect Intersections:**
         - Create empty `intersections` list [S3, unofficial].
         - For each of the 4 surrounding edges (top, left, bottom, right):
           - If `edgeHasSignChange` is true for that edge:
             - Get intersection point `p` using `getIntersectionPoint(x,y,isVertical,dx,dy)` [S3, unofficial].
             - Get normal `n` from the corresponding `verticalEdges` or `horizontalEdges` array [S3, unofficial].
             - Push `{p: p, n: n}` to `intersections` [S3, unofficial].
       - **Iterative QEF Solution (Simplified Approach from [S3, unofficial]):**
         - Initialize `point = {x:0.5, y:0.5}` [S3, unofficial].
         - For `step` from 0 to `NUMITERATIONS-1` (4):
           - Initialize `move = {x:0, y:0}` [S3, unofficial].
           - For each intersection `ii` in `intersections`:
             - Let `p = intersections[ii].p`, `n = intersections[ii].n` [S3, unofficial].
             - Calculate `projection(n, p, point)` [S3, unofficial].
             - `move = add(move, vec(point, projection(n, p, point)))` [S3, unofficial].
           - `move = mul(move, 0.2)` [S3, unofficial].
           - `point = add(point, move)` [S3, unofficial].
         - Set `positions[x][y] = point` [S3, unofficial].
       - **Alternative QEF Solution (Matrix Approach from [S1, S2, unofficial]):**
         - **Construct QEF Matrix A and Vector b:**
           - `A` is a matrix where each row is the normal of a crossed edge [S1, unofficial].
           - Calculate mean point `m`: `m[0] = mean(intercept[0] for all edges)`, `m[1] = mean(intercept[1] for all edges)` [S1, unofficial].
           - `b` is a vector where each element is `dot(normal, intercept - m)` [S1, unofficial].
         - **Compute $A^T A$ and $A^T b$:**
           - Compute $A^T A$ diagonal elements: $f[2] = A[1]^2 + A[3]^2 + A[5]^2 + A[7]^2$ (and $f[0]$ similarly) [S2, unofficial].
           - Assemble 2x2 symmetric matrix $A^T A$ in row-major layout: $ATA[0]=f[0], ATA[1]=f[1], ATA[2]=f[1], ATA[3]=f[2]$ [S2, unofficial].
           - Compute $A^T b$ vector: $f[3] = A[0]b[0] + A[2]b[1] + A[4]b[2] + A[6]b[3]$, $f[4] = A[1]b[0] + A[3]b[1] + A[5]b[2] + A[7]b[3]$ [S2, unofficial].
         - **Compute Mass Point $p_c$:**
           - $f[5] = cx / k$, $f[6] = cy / k$ (where $cx, cy$ are sums of intersection coordinates, $k$ is number of intersections) [S2, unofficial].
         - **Compute $A^T A p_c$:**
           - $ATAg[0] = ATA[0]f[5] + ATA[1]f[6]$, $ATAg[1] = ATA[2]f[5] + ATA[3]f[6]$ [S2, unofficial].
         - **Compute Residual $d$:**
           - $d[0] = f[3] - ATAg[0]$, $d[1] = f[4] - ATAg[1]$ [S2, unofficial].
         - **Solve for Offset $c$:**
           - Compute pseudo-inverse of $ATA$ using `PseudoInverse2(ATA, pinv)` [S2, unofficial].
           - $c[0] = pinv[0]d[0] + pinv[1]d[1]$, $c[1] = pinv[2]d[0] + pinv[3]d[1]$ [S2, unofficial].
         - **Compute Final Vertex Position $p$:**
           - $px = c[0] + f[5]$, $py = c[1] + f[6]$ [S2, unofficial].
           - Formula: $p = p_c + (A^T A)^{-1} (A^T b - A^T A p_c)$ [S2, unofficial].
         - **Boundary Check:**
           - If any coordinate of relative position is < 0 or > resolution, discard QEF solution and use mean point `m` as vertex position [S1, unofficial].
     - If false (no sign change):
       - Reset `positions[x][y].x=0.5`, `positions[x][y].y=0.5` [S3, unofficial].

5. **Generate Mesh Faces (Edges):**
   - Iterate over all cells [S1, unofficial].
   - For each cell `(i,j)` with a vertex:
     - Check right neighbor `(i+1, j)` and bottom neighbor `(i, j+1)` [S1, unofficial].
     - If neighbor exists, has a vertex, and the shared edge is crossed:
       - Create a face (line) between the two vertices [S1, unofficial].
       - Store vertex coordinates in output array `p` at index `2*pi`, and store cell index `pi` in `ibuffer[bi]` [S2, unofficial].
       - Store vertex pairs in `edges` flat integer array [S2, unofficial].

**Pseudocode (Simplified Iterative Approach from [S3, unofficial]):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_arrays(grid_width, grid_height):
    values = [[0 for _ in range(grid_height)] for _ in range(grid_width)]
    positions = [[{'x': 0.5, 'y': 0.5} for _ in range(grid_height)] for _ in range(grid_width)]
    vertical_edges = [[{'p': 0, 'nx': 1, 'ny': 1} for _ in range(grid_height)] for _ in range(grid_width)]
    horizontal_edges = [[{'p': 0, 'nx': 1, 'ny': 1} for _ in range(grid_height)] for _ in range(grid_width)]
    return values, positions, vertical_edges, horizontal_edges

def get_value(values, x, y, grid_width, grid_height):
    if x < 0 or x >= grid_width or y < 0 or y >= grid_height:
        return 0
    return values[x][y]

def grid_has_sign_change(values, x, y, grid_width, grid_height):
    sgn = get_value(values, x, y, grid_width, grid_height)
    for i in range(2):
        for j in range(2):
            if sgn != get_value(values, x + i, y + j, grid_width, grid_height):
                return True
    return False

def edge_has_sign_change(values, x, y, is_vertical, grid_width, grid_height):
    if is_vertical:
        return get_value(values, x, y, grid_width, grid_height) != get_value(values, x, y + 1, grid_width, grid_height)
    else:
        return get_value(values, x, y, grid_width, grid_height) != get_value(values, x + 1, y, grid_width, grid_height)

def get_intersection_point(vertical_edges, horizontal_edges, x, y, is_vertical, dx, dy, grid_width, grid_height):
    if x < 0 or x >= grid_width or y < 0 or y >= grid_height:
        return {'x': dx + 0.5, 'y': dy + 0.5}
    if is_vertical:
        return {'x': dx, 'y': dy + vertical_edges[x][y]['p']}
    else:
        return {'x': dx + horizontal_edges[x][y]['p'], 'y': dy}

def add(a, b):
    return {'x': a['x'] + b['x'], 'y': a['y'] + b['y']}

def mul(v, x):
    return {'x': v['x'] * x, 'y': v['y'] * x}

def vec(from_pt, to_pt):
    return {'x': to_pt['x'] - from_pt['x'], 'y': to_pt['y'] - from_pt['y']}

def projection(plane_normal, plane_point, point):
    # Simplified projection: distance from point to plane defined by normal and point
    # d = dot(normal, point - plane_point)
    # projected_point = point - normal * d
    d = dot(plane_normal, vec(plane_point, point))
    return add(point, mul(plane_normal, -d))

def dot(a, b):
    return a['x'] * b['x'] + a['y'] * b['y']

def update_square(values, positions, vertical_edges, horizontal_edges, x, y, grid_width, grid_height):
    if x < 0 or y < 0:
        return
    if grid_has_sign_change(values, x, y, grid_width, grid_height):
        intersections = []
        if edge_has_sign_change(values, x, y, True, grid_width, grid_height):
            intersections.append({'p': get_intersection_point(vertical_edges, horizontal_edges, x, y, True, 0, 0, grid_width, grid_height), 'n': vertical_edges[x][y]})
        if edge_has_sign_change(values, x, y, False, grid_width, grid_height):
            intersections.append({'p': get_intersection_point(vertical_edges, horizontal_edges, x, y, False, 0, 0, grid_width, grid_height), 'n': horizontal_edges[x][y]})
        if edge_has_sign_change(values, x + 1, y, True, grid_width, grid_height):
            intersections.append({'p': get_intersection_point(vertical_edges, horizontal_edges, x + 1, y, True, 1, 0, grid_width, grid_height), 'n': vertical_edges[x + 1][y]})
        if edge_has_sign_change(values, x, y + 1, False, grid_width, grid_height):
            intersections.append({'p': get_intersection_point(vertical_edges, horizontal_edges, x, y + 1, False, 0, 1, grid_width, grid_height), 'n': horizontal_edges[x][y + 1]})
        
        for ii in range(len(intersections)):
            intersections[ii]['n'] = {'x': intersections[ii]['n']['nx'], 'y': intersections[ii]['n']['ny']}
        
        point = {'x': 0.5, 'y': 0.5}
        num_iterations = 4
        for step in range(num_iterations):
            move = {'x': 0, 'y': 0}
            for i in range(len(intersections)):
                p = intersections[i]['p']
                n = intersections[i]['n']
                move = add(move, vec(point, projection(n, p, point)))
            move = mul(move, 0.2)
            point = add(point, move)
        
        positions[x][y] = point
    else:
        positions[x][y]['x'] = 0.5
        positions[x][y]['y'] = 0.5

def generate_mesh(vertices, edges, values, positions, vertical_edges, horizontal_edges, grid_width, grid_height):
    for x in range(grid_width):
        for y in range(grid_height):
            update_square(values, positions, vertical_edges, horizontal_edges, x, y, grid_width, grid_height)
            # Rerender edges logic omitted for brevity, see [S3, unofficial]
    
    # Generate faces
    for x in range(grid_width - 1):
        for y in range(grid_height - 1):
            if positions[x][y]['x'] != 0.5 or positions[x][y]['y'] != 0.5:
                # Check right neighbor
                if positions[x + 1][y]['x'] != 0.5 or positions[x + 1][y]['y'] != 0.5:
                    if edge_has_sign_change(values, x, y, False, grid_width, grid_height):
                        edges.append([x, x + 1]) # Simplified edge representation
                # Check bottom neighbor
                if positions[x][y + 1]['x'] != 0.5 or positions[x][y + 1]['y'] != 0.5:
                    if edge_has_sign_change(values, x, y, True, grid_width, grid_height):
                        edges.append([y, y + 1]) # Simplified edge representation
```

**Parameters:**
- `WIDTH`: Grid width in pixels, constant value 512 [S1, unofficial].
- `HEIGHT`: Grid height in pixels, constant value 512 [S1, unofficial].
- `RESOLUTION`: Grid cell size in world units, `WIDTH / 64` [S1, unofficial].
- `gridWidth`: Width of the uniform grid in cells. Typical value: 22 [S3, unofficial].
- `gridHeight`: Height of the uniform grid in cells. Typical value: 12 [S3, unofficial].
- `MAX_ERR`: Maximum allowed error for binary search intersection, constant value 0.1 [S1, unofficial].
- `NUMITERATIONS`: Number of iterative steps for QEF convergence. Value: 4 [S3, unofficial].
- `dh`: Finite difference step size for gradient estimation. Value: `1e-5f` [S2, unofficial].
- `eps`: Epsilon threshold for root-finding convergence. Value: `1e-6f` [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Handling:** `getValue` returns 0 (air) for coordinates outside the grid bounds [0, `gridWidth`) x [0, `gridHeight`) [S3, unofficial]. `getIntersectionPoint` returns a default point `{x:dx+0.5, y:dy+0.5}` for out-of-bounds edges [S3, unofficial].
- **Vertex Outside Cell:** If any coordinate of the relative vertex position is < 0 or > resolution, discard QEF solution and use mean point `m` as vertex position [S1, unofficial].
- **Clamping:** Clamping computed point coordinates to [0,1] is not required in this implementation [S3, unofficial].
- **Tile Boundaries:** For tiled implementations, boundary vertices at tile edges must be handled carefully to avoid duplicate edges or missing connections between adjacent tiles [S2, unofficial].

## Subtopics

### [Dual Contouring with NDC Constraint](4410-dual-contouring-with-ndc-constraint.md)
Enforces vertex positions within $[0,1]$ bounds using iterative scale adjustment and binary search, unlike standard clamping. Choose when strict NDC compliance is required.

### [Dual Contouring 2D (CUDA)](3671-dual-contouring-2d-cuda.md)
GPU-accelerated version using tiles and shared memory for parallel processing. Choose for high-performance rendering on large grids.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$ for grid sampling and edge detection, where $N$ is grid width and $M$ is grid height. QEF minimization is $O(K)$ per cell, where $K$ is the number of intersected edges (typically 2-4). Total time is linear with respect to the number of cells.
- **Memory Complexity:** $O(N \cdot M)$ for storing grid values, edges, and positions.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with grid resolution. Quadtree subdivision can be used to adaptively refine regions of interest, reducing memory and computation for homogeneous areas [S1, unofficial].
- **Strengths:** Produces smoother, more accurate meshes than Marching Squares by placing vertices optimally within cells. Handles complex topologies well.
- **Limitations:** More computationally expensive than Marching Squares due to QEF solving. Requires careful handling of boundary conditions and tile synchronization in parallel implementations.
- **When to Use:** When high-quality, smooth 2D meshes are needed from scalar fields, and performance is not the primary constraint.
- **When Not to Use:** For real-time applications with very large grids where performance is critical; consider Marching Squares or CUDA-accelerated Dual Contouring instead.

## Background
Dual Contouring was introduced by Frank Peterson in 2004 [S1, unofficial]. The 2D grid implementation described here is a simplified version of the original algorithm, adapted for uniform grids and implemented in JavaScript for educational purposes [S1, S3, unofficial]. The CUDA version [S2, unofficial] extends the method to GPU for parallel processing. It is used in computer graphics for surface extraction from implicit functions, such as in terrain generation or medical imaging.

## Sources
[S1] AdamStone/dual-contouring-2d-grid, https://github.com/AdamStone/dual-contouring-2d-grid, fetched 2026-10-26 (unofficial)
[S2] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
[S3] radeusgd/2dJSdualcontour, https://github.com/radeusgd/2dJSdualcontour, fetched 2026-10-26 (unofficial)
