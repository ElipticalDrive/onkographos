# Point Implicit Contouring (PIC)

Also called: PIC with implicit connectivity, Point with Implicit Connectivity, Point-based implicit contouring

## Overview
Point Implicit Contouring (PIC) is a sparse voxel storage format that represents a 3D object as a dense voxel grid of material indices or signs, combined with a sparse set of points lying on the object's surface within boundary cells [S1]. Unlike formats that store explicit topology or continuous surface definitions, PIC derives mesh connectivity implicitly from the voxel data (signs or material indices at cell corners) using dual triangulation methods [S1]. It is distinguished from its sibling [Ray-Rep](8022-ray-representation.md) by supporting surface smoothing and eliminating the need for post-editing regularization, though it trades off some CSG speed for this flexibility [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A uniform Cartesian grid $C$ of size $n \times n \times n$ cells per block.
*   A dense voxel grid $V$ storing material indices or binary signs at the grid nodes (corners).
*   A sparse set of points $S$ associated with inhomogeneous cells.

**Outputs:**
*   A polygonal mesh where vertices are the points in $S$ and connectivity is determined by the triangulation algorithm applied to the voxel data.

### Data Structures
Each block of the landscape is stored as two arrays:

1.  **Voxel Grid ($V$):** A dense 3D array of integers.
    $$ V = \{ v_{i,j,k} \in \mathbb{Z} \mid i, j, k \in \mathbb{N}, 0 \le i, j, k \le n \} $$
    $v_{i,j,k}$ represents the material index or sign (inside/outside) at the node $(i,j,k)$, which corresponds to the corners of cell $c_{i,j,k}$.

2.  **Point Cloud ($S$):** A sparse 3D array of 3D vectors.
    $$ S = \{ p_{i,j,k} \in \mathbb{R}^3 \mid i, j, k \in \mathbb{Z}, 0 \le i, j, k < n \land c_{i,j,k} \text{ is inhomogeneous} \} $$
    $p_{i,j,k}$ is the position of a sharp vertex located inside cell $c_{i,j,k}$. A cell is **inhomogeneous** if its corners do not have identical materials or signs.

**Memory Layout Details:**
*   **Vertex Quantization:** Sharp vertex positions $p_{i,j,k}$ are quantized relative to the cell boundaries (Axis-Aligned Bounding Box). 8 bits per coordinate component are sufficient [S1].
*   **Ghost Cells:** To handle boundary conditions, layers of dummy cells are added. These "ghost cells" (or halo cells) duplicate data from neighboring blocks and are physically located within the neighbor's memory space [S1].
    *   For seamlessness (no gaps): Overlap adjacent blocks by a layer of thickness equal to one cell [S1].
    *   For smooth normals: Overlap blocks by a layer of thickness equal to two cells [S1].
*   **Optimal Block Size:** For interactive performance, a block size of $32^3$ cells is optimal [S1].

### Algorithm Steps

#### 1. Identification of Boundary Cells
Iterate through all cells $c_{i,j,k}$ in the grid $C$. A cell is marked as a boundary (inhomogeneous) cell if the values $v_{i,j,k}$ at its eight corners are not all identical. Only these cells require a point in $S$ [S1].

#### 2. Triangulation (Mesh Generation)
Connect the stored sharp vertices to form a polygonal mesh. Any cell-based dual triangulation method can be used, such as Dual Contouring (DC) or Dual Marching Cubes (DMC) [S1].
*   If using **DC**, one sharp vertex is created per boundary cell [S1].
*   If using **DMC**, one to four sharp vertices may be created per boundary cell depending on the sign configuration [S1].

**Inter-cell Dependency Handling:**
Dual triangulation methods have inter-cell dependencies. To generate a seamless mesh connecting a block to its neighbors, adjacent boundary cells from neighboring blocks must be included in the triangulation process [S1]. This is achieved using ghost cells.
*   **Pitfall:** Polygons formed by ghost cells are used *only* for boundary conditions (seamlessness and smooth normals) and must **NOT** be included in the final polygonal mesh output [S1].

#### 3. Hermite Data Computation (On-Demand)
Hermite data (intersection points and unit surface normals) is not stored explicitly. It is computed on demand, typically before editing operations [S1].
1.  Create a partial contour (fragment of the polygonal mesh) using the chosen triangulation method (DC/DMC) [S1].
2.  Intersect this partial contour with the active edges of the cells to find intersection points and unit surface normals (Hermite data) [S1].

#### 4. CSG Operations (Boolean Editing)
To perform boolean operations (e.g., union, intersection, difference):
1.  Build a partial contour using a dual triangulation algorithm (DC/DMC) [S1].
2.  Intersect the fragment with active cell edges to obtain Hermite data (segments) [S1].
3.  Perform CSG operations on the voxel grid (points) and the Hermite data (segments) [S1].
4.  Compute new sharp vertex positions based on the result [S1].
*   **Quality:** Even with 8-bit quantization, PIC ensures high-quality reconstruction of sharp edges and corners after boolean operations [S1].

#### 5. Surface Smoothing
PIC supports surface smoothing to remove unwanted sharp edges or improve triangle quality [S1].
1.  Identify vertices in active cells affected by smoothing.
2.  Shift the position of a vertex to the center of mass of vertices from neighboring active cells.
3.  **Constraint:** Average only vertices from neighbors that share an **active face** with the current cell. A cubic cell has up to six active faces [S1].

#### 6. Post-Editing Update
After editing operations, the ghost cells of all affected blocks must be updated (halo exchange) before triangulation to maintain consistency across block boundaries [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(block_voxels, block_points, ghost_cells):
    """
    Generates a polygonal mesh from PIC data.
    
    Args:
        block_voxels: Dense 3D array V of size (n+1)x(n+1)x(n+1) containing material indices/signs.
        block_points: Sparse map of points S, keyed by cell index (i,j,k) for inhomogeneous cells.
        ghost_cells: Dictionary or array containing voxel data from neighboring blocks.
        
    Returns:
        List of triangles (faces).
    """
    faces = []
    n = block_voxels.shape[0] - 1  # Number of cells per dimension
    
    # 1. Identify inhomogeneous cells and prepare for triangulation
    # Note: Ghost cells are included in the voxel data context for boundary checks
    for i in range(n):
        for j in range(n):
            for k in range(n):
                cell_idx = (i, j, k)
                
                # Check if cell is inhomogeneous
                if not is_inhomogeneous(block_voxels, i, j, k, ghost_cells):
                    continue
                
                # 2. Triangulate using Dual Contouring or DMC
                # The specific algorithm (DC/DMC) determines how many vertices 
                # are generated and how they are connected.
                # Here we assume a generic dual triangulation function that 
                # uses the stored sharp vertex (block_points[cell_idx]) 
                # and the corner signs (block_voxels).
                
                cell_faces = dual_triangulate(
                    voxel_data=block_voxels,
                    sharp_vertex=block_points[cell_idx],
                    cell_index=cell_idx,
                    ghost_cells=ghost_cells
                )
                
                # 3. Filter out ghost-generated faces
                # Polygons formed by ghost cells are for boundary conditions only
                valid_faces = []
                for face in cell_faces:
                    if not is_ghost_face(face, i, j, k, n):
                        valid_faces.append(face)
                
                faces.extend(valid_faces)
                
    return faces

def is_inhomogeneous(voxels, i, j, k, ghost_cells):
    """
    Checks if the corners of cell (i,j,k) have different material indices or signs.
    Uses ghost_cells for boundary checks.
    """
    # Retrieve corner values, handling boundaries with ghost cells
    corners = []
    for dx in [0, 1]:
        for dy in [0, 1]:
            for dz in [0, 1]:
                val = get_voxel_value(voxels, i+dx, j+dy, k+dz, ghost_cells)
                corners.append(val)
    
    # If all corners are identical, the cell is homogeneous
    return not all(c == corners[0] for c in corners)

def dual_triangulate(voxel_data, sharp_vertex, cell_index, ghost_cells):
    """
    Performs dual triangulation (e.g., DC or DMC) for a single cell.
    Returns a list of triangles.
    """
    # Implementation depends on the specific dual method chosen.
    # For DC: The sharp_vertex is the vertex of the dual mesh.
    # For DMC: Vertices are interpolated on edges based on signs.
    # This function connects the sharp_vertex to the intersection points
    # on the cell edges defined by the voxel signs.
    pass

def is_ghost_face(face, i, j, k, n):
    """
    Determines if a face was generated using ghost cell data and should be excluded.
    """
    # Logic to check if any vertex of the face lies outside the primary block
    # or was derived from ghost data.
    pass

def update_ghost_cells(affected_blocks):
    """
    Halo exchange: Update ghost cells of all affected blocks after editing.
    """
    for block in affected_blocks:
        # Copy boundary data from neighbors to this block's ghost layer
        # and vice versa
        pass

def smooth_surface(block_points, block_voxels, ghost_cells):
    """
    Smooths vertex positions by moving them to the center of mass
    of vertices from neighboring active cells sharing an active face.
    """
    for cell_idx, vertex in block_points.items():
        if not is_boundary_cell(block_voxels, cell_idx, ghost_cells):
            continue
            
        neighbors = get_active_neighbors_sharing_face(cell_idx, block_voxels, ghost_cells)
        if not neighbors:
            continue
            
        # Calculate center of mass of neighbor vertices
        avg_pos = sum(block_points[n] for n in neighbors) / len(neighbors)
        
        # Shift vertex
        block_points[cell_idx] = avg_pos
```

## Operating characteristics
*   **Time Complexity:** Triangulation speed is lower than [Ray-Rep](8022-ray-representation.md) but higher than other voxel formats [S1]. CSG operations are slightly slower than Ray-Rep [S1].
*   **Memory Complexity:** Medium uncompressed memory usage [S1]. The dense voxel grid dominates memory, while the point cloud is sparse.
*   **Performance:** Tested on Intel Core i7-2600K @ 3.40 GHz with 16 GB RAM. Optimal block size for interactivity is $32^3$ cells [S1].
*   **Strengths:**
    *   Supports surface smoothing [S1].
    *   No need for regularization after editing (unlike Ray-Rep) [S1].
    *   High-quality reconstruction of sharp edges/corners after boolean operations [S1].
    *   Wide functionality for voxel terrain design [S1].
*   **Limitations:**
    *   Inter-cell dependencies require ghost cells and halo exchanges, complicating parallelization [S1].
    *   Seams and gaps may appear between adjacent blocks after editing if data is not periodically synchronized [S1].
    *   Lower CSG speed compared to Ray-Rep [S1].

## Background
Point Implicit Contouring was proposed as a representation for voxel terrains in virtual reality systems. It was introduced in a 2019 study comparing various voxel storage formats, highlighting its advantages in editing and smoothing capabilities over ray-based representations [S1]. The method builds on the concept of sparse voxel grids but adds explicit point storage for surface vertices while retaining implicit connectivity [S1].

## Sources
[S1] REPRESENTATION AND STORAGE OF VOXEL TERRAIN FOR DESIGNING VIRTUAL REALITY SYSTEMS, https://doi.org/10.26102/2310-6018/2019.24.1.019, fetched 2026-10-26
