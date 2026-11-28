# Occupancy-Based Dual Contouring


## Overview
Occupancy-Based Dual Contouring (ODC) is a surface extraction method designed to convert neural occupancy functions into high-fidelity, manifold meshes without relying on gradient information [S1][S2, unofficial]. Unlike standard dual contouring which uses signed distance gradients for normals, ODC introduces auxiliary 2D points on grid faces to compute local surface normals, enabling accurate vertex placement via the Quadric Error Function (QEF) [S1]. It distinguishes itself from siblings like [Surface Nets](921-surface-nets.md) by resolving topological ambiguities and eliminating self-intersections through a modified polygonization step, making it superior for neural implicit representations where gradients are noisy or unavailable [S1][S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   An implicit function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$ (e.g., occupancy network output) [S2, unofficial].
*   Bounding box $\mathcal{P}$ defined by `min_coord` and `max_coord` [S2, unofficial].
*   Grid resolution `num_grid` (uniform grid cells per axis) [S2, unofficial].
*   Iso-level `isolevel` (typically 0.5 for occupancy, 0.0 for SDF) [S2, unofficial].
*   Boolean `outside` indicating if larger values are outside [S2, unofficial].

**Outputs:**
*   Vertices: Tensor of shape $(N, 3)$ [S2, unofficial].
*   Triangles: Tensor of shape $(M, 3)$ containing vertex indices [S2, unofficial].

### Data Structures
**pfvtab (Polygon-Face-Vertex Table):**
A lookup table for dual contouring vertex configurations. It is a list of 64 entries (indices 0–63). Each entry is a list of one or more lists of integers representing vertex indices. The integers range from 1 to 24, corresponding to the 24 possible edge/face intersection points in the dual contouring topology [S2, unofficial].

### Parameters
*   `min_coord`: `[-0.5, -0.5, -0.5]` [S2, unofficial].
*   `max_coord`: `[0.5, 0.5, 0.5]` [S2, unofficial].
*   `num_grid`: `128` [S2, unofficial].
*   `isolevel`: `0.0` (use `0.5` for occupancy) [S2, unofficial].
*   `imp_func_cplx`: `3` (GPU VRAM usage factor) [S2, unofficial].
*   `outside`: `True` [S2, unofficial].
*   `BINARY_SEARCH`: `15` (iterations for 1D point search) [S2, unofficial].
*   `VERTICAL_RANGE`: `0.8` (range for step 1 of 2D search) [S2, unofficial].
*   `VERTICAL_LINEAR_SEARCH`: `4` (linear iterations for step 1 of 2D search) [S2, unofficial].
*   `VERTICAL_BINARY_SEARCH`: `11` (binary iterations for step 1 of 2D search) [S2, unofficial].
*   `HORIZNTL_RANGE`: `0.71` (range for step 2 of 2D search) [S2, unofficial].
*   `HORIZNTL_LINEAR_SEARCH`: `3` (linear iterations for step 2 of 2D search) [S2, unofficial].
*   `HORIZNTL_BINARY_SEARCH`: `12` (binary iterations for step 2 of 2D search) [S2, unofficial].
*   `ERR`: `3e-7` (error tolerance) [S2, unofficial].
*   `SEP`: `1e-3` (minimum separation of 3D point from grid face) [S2, unofficial].
*   `QEF_REG`: `0.05` (regularization coefficient) [S2, unofficial].

### Algorithm Steps

#### 1. Grid Setup and Configuration Identification
Initialize a uniform grid within $\mathcal{P}$ with resolution `num_grid`. For each grid cell, determine the occupancy state of its 8 vertices. Identify the 22 configurations out of 256 possible configurations (accounting for rotation and mirroring) that involve surface-crossing grid edges [S1]. Partition the surface-crossing grid edges in a grid cell into sets $E_{S_{c,1}}, \dots, E_{S_{c,4}}$, each associated with a different polygonal face [S1].

#### 2. 1D Point Search (Grid Edge Points)
For each grid edge intersecting the isosurface, locate the 1D point $p_{1D}$ where $f(p_{1D}) = \text{isolevel}$.
*   Perform a binary search along the edge.
*   Number of iterations: `BINARY_SEARCH` [S2, unofficial].
*   This step fully exploits the continuous nature of the implicit function, not just grid vertices [S1].

#### 3. 2D Point Search (Grid Face Points)
Define auxiliary 2D points on grid faces. These points are assumed to lie on the same local flat surface as neighboring 1D points [S1].
*   **Local Flatness Assumption:** The surface is locally planar.
*   **Adjacency:** In the primal face polygon cycle, 1D points and 2D points alternate. Each 1D point is adjacent to two 2D points, and each 2D point is adjacent to two 1D points [S1].
*   **Search Algorithm:** For a 2D point $p_{2D}$ on a face, use the two adjacent 1D points to constrain the search. The algorithm guarantees finding points on the same plane as the 1D points using three ray searches [S1].
    *   **Step 1 (Vertical):** Search within `VERTICAL_RANGE` using `VERTICAL_LINEAR_SEARCH` linear steps followed by `VERTICAL_BINARY_SEARCH` binary steps [S2, unofficial].
    *   **Step 2 (Horizontal):** Search within `HORIZNTL_RANGE` using `HORIZNTL_LINEAR_SEARCH` linear steps followed by `HORIZNTL_BINARY_SEARCH` binary steps [S2, unofficial].
*   **Normal Calculation:** Compute the surface normal $\mathbf{n}_{1D}$ at a 1D point using the two adjacent 2D points. The normal is the cross product of the vectors connecting the 1D point to its adjacent 2D points, normalized [S1].

#### 4. 3D Point Identification (Cell Points)
For each grid cell, identify the 3D point $p_{3D}$ inside the cell using the Quadric Error Function (QEF).
*   **QEF Construction:** Use the surface normals $\mathbf{n}_{1D}$ computed in Step 3 and the positions of the 1D points $p_{1D}$ on the cell's edges. The QEF minimizes $\sum (\mathbf{n}_i \cdot (\mathbf{x} - \mathbf{p}_i))^2$ [S1].
*   **Regularization:** Add a regularization term with coefficient `QEF_REG` to ensure stability [S2, unofficial].
*   **Constraint:** Ensure the 3D point is separated from the grid face by at least `SEP` to avoid degeneracy [S2, unofficial].
*   Solve the linear system derived from the QEF to find $p_{3D}$ [S1].

#### 5. Polygonization and Ambiguity Resolution
Generate the mesh polygons based on the 3D points and the configuration table `pfvtab` [S2, unofficial].
*   **Ambiguity Resolution:** Resolve the C12/C18 topological ambiguities using the solution described in [Wenger 2013] to guarantee manifold output [S1]. Note: Without this, coincident C12/C18 ambiguities may produce non-manifold edges [S1].
*   **Self-Intersection Mitigation:** Incorporate the quad splitting technique from Intersection-free Contouring (IC) [Ju and Udeshi 2006] into the final polygonization step. This significantly reduces self-intersecting mesh faces [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def occupancy_dual_contouring(implicit_function, min_coord, max_coord, num_grid, isolevel, outside, 
                              BINARY_SEARCH, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                              HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH, ERR, SEP, QEF_REG):
    """
    Extracts mesh from occupancy function using ODC.
    """
    # 1. Grid Setup
    # Create uniform grid points
    x = torch.linspace(min_coord[0], max_coord[0], num_grid + 1)
    y = torch.linspace(min_coord[1], max_coord[1], num_grid + 1)
    z = torch.linspace(min_coord[2], max_coord[2], num_grid + 1)
    grid_points = torch.stack(torch.meshgrid(x, y, z, indexing='ij'), dim=-1)
    
    # Evaluate occupancy at grid vertices
    # Batched evaluation for GPU efficiency
    batch_size = 100000000
    num_vertices = grid_points.shape[0] * grid_points.shape[1] * grid_points.shape[2]
    grid_occupancy = torch.zeros(num_vertices)
    
    for i in range(0, num_vertices, batch_size):
        end = min(i + batch_size, num_vertices)
        pts = grid_points.view(-1, 3)[i:end]
        grid_occupancy[i:end] = implicit_function(pts)
    
    grid_occupancy = grid_occupancy.view(num_grid + 1, num_grid + 1, num_grid + 1)
    
    # 2. Identify Surface-Crossing Cells and Edges
    # Determine cell configurations (22 relevant ones)
    # Partition edges into sets E_Sc,1 ... E_Sc,4
    # This part relies on the topology defined in Dual Marching Cubes / MDC
    # For each cell, identify intersecting edges
    
    vertices_list = []
    triangles_list = []
    
    # Parallel processing over all cells would be done here in GPU implementation
    # Iterating here for clarity
    for i in range(num_grid):
        for j in range(num_grid):
            for k in range(num_grid):
                # Get 8 vertices of the cell
                v_indices = [
                    (i, j, k), (i+1, j, k), (i+1, j+1, k), (i, j+1, k),
                    (i, j, k+1), (i+1, j, k+1), (i+1, j+1, k+1), (i, j+1, k+1)
                ]
                v_occ = [grid_occupancy[idx] for idx in v_indices]
                
                # Check if cell intersects isosurface
                if not any(o <= isolevel for o in v_occ) or not any(o > isolevel for o in v_occ):
                    continue
                
                # Determine configuration and partition edges
                # Identify 1D points on edges
                edge_points_1D = []
                edge_normals_1D = []
                
                # Define edges of the cube
                edges = [
                    (0,1), (1,2), (2,3), (3,0), # Bottom
                    (4,5), (5,6), (6,7), (7,4), # Top
                    (0,4), (1,5), (2,6), (3,7)  # Vertical
                ]
                
                for e_idx, (v1, v2) in enumerate(edges):
                    o1 = v_occ[v1]
                    o2 = v_occ[v2]
                    
                    # Check if edge crosses isosurface
                    if (o1 <= isolevel) != (o2 > isolevel):
                        # 1D Point Search (Binary Search)
                        p1D = binary_search_1D(v_indices[v1], v_indices[v2], implicit_function, isolevel, BINARY_SEARCH)
                        edge_points_1D.append(p1D)
                        
                        # 2D Point Search for Normal Calculation
                        # Find adjacent 2D points on faces sharing this edge
                        # This requires identifying the faces and performing 2D search
                        # For simplicity, assume we get 2D points p2D_a, p2D_b
                        p2D_a = search_2D_point(face_a, edge_points_1D, implicit_function, isolevel, 
                                                VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                                                HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH)
                        p2D_b = search_2D_point(face_b, edge_points_1D, implicit_function, isolevel, 
                                                VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                                                HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH)
                        
                        # Compute Normal
                        vec1 = p2D_a - p1D
                        vec2 = p2D_b - p1D
                        normal = torch.cross(vec1, vec2)
                        if torch.norm(normal) > 0:
                            normal = normal / torch.norm(normal)
                        else:
                            normal = torch.tensor([0.0, 0.0, 0.0]) # [G] Fallback
                        edge_normals_1D.append(normal)
                
                # 3D Point Identification via QEF
                if len(edge_points_1D) > 0:
                    # Construct QEF matrix A and vector b
                    # Minimize sum (n_i . (x - p_i))^2
                    A = torch.zeros(3, 3)
                    b = torch.zeros(3)
                    for n, p in zip(edge_normals_1D, edge_points_1D):
                        outer = torch.outer(n, n)
                        A += outer
                        b += torch.dot(n, p) * n
                    
                    # Add regularization
                    A += QEF_REG * torch.eye(3)
                    
                    # Solve Ax = b
                    try:
                        p3D = torch.linalg.solve(A, b)
                    except:
                        p3D = torch.mean(torch.stack(edge_points_1D), dim=0) # [G] Fallback
                    
                    # Check separation
                    # Ensure p3D is not too close to grid faces
                    # This is handled by SEP parameter in implementation
                    
                    vertices_list.append(p3D)
                    
                    # 5. Polygonization
                    # Use pfvtab to determine triangles
                    # Resolve C12/C18 ambiguities
                    # Apply IC quad splitting
                    # Add triangles to triangles_list
                    pass

    return torch.stack(vertices_list), torch.tensor(triangles_list)

def binary_search_1D(v1_idx, v2_idx, implicit_function, isolevel, iterations):
    # Get world coordinates of v1 and v2
    # Perform binary search along the segment
    # Return point where f(p) approx isolevel
    pass

def search_2D_point(face, edge_points, implicit_function, isolevel, 
                    vert_range, vert_lin, vert_bin, horiz_range, horiz_lin, horiz_bin):
    # Search for 2D point on the face
    # Use adjacent 1D points to constrain search
    # Step 1: Vertical search
    # Step 2: Horizontal search
    pass
```

## Operating characteristics
*   **Time Complexity:** The method is designed for GPU parallelization across all grid edges, faces, and cells. On an ablation study hardware, it takes approximately 3.5 seconds for a $128^3$ resolution grid [S1]. This is slower than Manifold Dual Contouring (MDC) which takes 2.0 seconds, but faster than methods requiring extensive gradient computation [S1].
*   **Memory Complexity:** Depends on the grid resolution and the complexity of the implicit function query (`imp_func_cplx`). The lookup table `pfvtab` is small (64 entries) [S2, unofficial].
*   **Performance:** ODC achieves the best fidelity compared to prior works, significantly reducing fitting error ($|f - 0.5|$ reduced by factor of 3 with 1D search, and another factor of 3 with 2D search) [S1]. It eliminates self-intersections (0% vs 100% for MDC) [S1].
*   **Strengths:** High fidelity, manifold output, no gradient required, GPU parallelizable.
*   **Limitations:** Slower than MDC due to 2D point searches; requires careful parameter tuning for search ranges and iterations [S1][S2, unofficial].
*   **When to use:** When extracting meshes from neural occupancy functions (e.g., IM-NET, Occupancy Networks, DeepSDF) where gradients are unreliable or unavailable [S2, unofficial].
*   **When not to use:** When computational speed is the primary constraint and gradient information is available (use MDC or standard Dual Contouring) [S1].

## Background
Occupancy-Based Dual Contouring (ODC) was proposed by the KAIST Visual AI Group [S1]. It is based on Manifold Dual Contouring (MDC) [Schaefer et al. 2007] and Dual Marching Cubes [Nielson 2004]. The method was introduced to address the bottleneck in converting neural occupancy functions to meshes, as Marching Cubes produces staircase artifacts and SDF-based methods yield suboptimal results for occupancy data [S1][S2, unofficial]. The code is available at [https://github.com/KAIST-Visual-AI-Group/ODC](https://github.com/KAIST-Visual-AI-Group/ODC) [S2, unofficial].

## Sources
[S1] Occupancy-Based Dual Contouring, https://arxiv.org/abs/2409.13418v1, fetched 2026-10-26
[S2] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
