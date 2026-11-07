# TRELLIS.2 QEF for Active Cube Vertex Positioning

Also called: TRELLIS.2 preprocessing QEF

## Overview
TRELLIS.2 QEF is a vertex positioning method used in the preprocessing stage of the TRELLIS.2 framework [S1]. It solves the problem of determining the optimal position for dual vertices within "active cubes" identified by intersecting an input mesh, enabling the subsequent extraction of high-quality meshes via Dual Contouring. It distinguishes itself from standard Dual Contouring QEF solvers by explicitly incorporating boundary edge constraints and regularization terms to handle sharp features and intricate internal structures [S1].

## How it works
**Inputs:**
*   An input ground-truth (GT) mesh.
*   A grid of cubes (voxels) covering the domain.
*   Parameters: $\lambda_{\text{bound}}$ (boundary weight), $\lambda_{\text{reg}}$ (regularization weight).

**Outputs:**
*   A set of active cubes.
*   For each active cube, an edge intersection flag vector $\in \{0, 1\}^3$ indicating surface intersections along the $x, y, z$ axes.
*   For each active cube, a dual vertex position $v$ computed by minimizing the QEF.

**Data Structures:**
*   **Active Cube:** A cube in the grid where at least one edge intersects the input mesh.
*   **Edge Intersection Flag:** A 3-bit integer or boolean vector $(f_x, f_y, f_z)$, where $f_k=1$ if the cube has a surface intersection along axis $k$, and $0$ otherwise.
*   **Intersection Data:** For each intersection point $\mathbf{p}_i$ on the cube boundary, store the surface normal $\mathbf{n}_i$.
*   **Boundary Edge Data:** For each boundary edge of the cube that intersects the mesh, store the edge origin $\mathbf{o}_j$ and direction $\mathbf{d}_j$.

**Algorithm Steps:**

1.  **Identify Active Cubes:**
    Iterate over all cubes in the grid. For each cube, detect intersections between its 12 edges and the input mesh. If any intersection is found, mark the cube as active.

2.  **Assign Edge Intersection Flags:**
    For each active cube, determine the presence of surface intersections along the three principal axes ($x, y, z$). Assign a flag vector $\in \{0, 1\}^3$ to the cube. This flag indicates which axes have intersections, facilitating subsequent mesh polygonization [S1].

3.  **Compute Dual Vertex Position:**
    For each active cube, solve for the dual vertex position $v$ by minimizing the following Quadric Error Function (QEF) [S1]:

    $$
    \min_{v \in \text{voxel}} e(v) = \sum_i d_{\text{II},i}^2 + \lambda_{\text{bound}} \sum_j d_{L,j}^2 + \lambda_{\text{reg}} d_q^2
    $$

    Where:
    *   $v$ is the dual vertex position.
    *   The first term is the **plane alignment error**:
        $$
        d_{\text{II},i}^2 = (\mathbf{n}_i \cdot (v - \mathbf{p}_i))^2
        $$
        Here, $\mathbf{p}_i$ is the intersection point on the cube boundary, and $\mathbf{n}_i$ is the surface normal at $\mathbf{p}_i$. This term measures the squared distance from $v$ to the tangent plane defined by $\mathbf{p}_i$ and $\mathbf{n}_i$.
    *   The second term is the **boundary constraint**:
        $$
        d_{L,j}^2 = \| (v - \mathbf{o}_j) - ((v - \mathbf{o}_j) \cdot \mathbf{d}_j)\mathbf{d}_j \|^2
        $$
        Here, $\mathbf{o}_j$ is the origin and $\mathbf{d}_j$ is the direction of the $j$-th boundary edge. This term measures the squared distance from $v$ to the $j$-th boundary edge. It is weighted by $\lambda_{\text{bound}}$.
    *   The third term is the **regularization term**:
        $$
        d_q^2 = \| v - \bar{v} \|^2
        $$
        Here, $\bar{v}$ is the centroid (or mass point) of the intersection points $\{\mathbf{p}_i\}$ [S1]. This term penalizes deviation from the centroid and is weighted by $\lambda_{\text{reg}}$.

    The minimization yields the optimal $v$ for the active cube.

**Pitfalls:**
*   **Floating-Point Precision:** The determination of edge intersection flags is susceptible to floating-point precision errors inherent in ray-intersection tests. This may lead to erroneous intersection judgments, affecting the correctness of the active cube identification and flag assignment [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def trellis2_qef_vertex_positioning(input_mesh, grid_cubes, lambda_bound, lambda_reg):
    """
    Computes active cubes, flags, and dual vertices using TRELLIS.2 QEF.
    
    Args:
        input_mesh: Object representing the input GT mesh, with methods for 
                    ray-mesh intersection.
        grid_cubes: List of cube objects, each with properties:
                    - center: (x, y, z)
                    - size: float (edge length)
                    - edges: List of 12 edge segments (each defined by two endpoints)
        lambda_bound: Weight for boundary constraint.
        lambda_reg: Weight for regularization term.
        
    Returns:
        active_cubes: List of active cube objects.
        flags: Dictionary mapping cube_id to (f_x, f_y, f_z) flag vector.
        vertices: Dictionary mapping cube_id to dual vertex position v.
    """
    active_cubes = []
    flags = {}
    vertices = {}
    
    for cube in grid_cubes:
        # Step 1: Identify Active Cubes
        # Detect intersections between cube edges and input mesh
        intersections = []
        has_intersection = False
        
        for edge in cube.edges:
            # Perform ray-mesh intersection test
            # Returns list of intersection points and normals
            hits = input_mesh.ray_intersect(edge.start, edge.end)
            if hits:
                has_intersection = True
                for hit_point, hit_normal in hits:
                    intersections.append((hit_point, hit_normal))
        
        if not has_intersection:
            continue
            
        active_cubes.append(cube)
        
        # Step 2: Assign Edge Intersection Flags
        # Determine presence of surface intersections along x, y, z axes
        # This requires checking which axes have at least one intersection
        # Note: The source notes flag determination is error-prone due to precision
        flag_x = False
        flag_y = False
        flag_z = False
        
        for hit_point, hit_normal in intersections:
            # Check if intersection lies on an edge parallel to x, y, or z axis
            # This logic depends on how 'intersections' are attributed to specific edges
            # Assuming we can determine the axis of the intersected edge
            intersected_edge_axis = get_edge_axis(cube, hit_point) 
            if intersected_edge_axis == 'x':
                flag_x = True
            elif intersected_edge_axis == 'y':
                flag_y = True
            elif intersected_edge_axis == 'z':
                flag_z = True
                
        flags[cube.id] = (int(flag_x), int(flag_y), int(flag_z))
        
        # Step 3: Compute Dual Vertex Position via QEF Minimization
        # Collect boundary edge data for the cube
        boundary_edges = []
        for edge in cube.edges:
            # Check if this edge intersects the mesh
            hits = input_mesh.ray_intersect(edge.start, edge.end)
            if hits:
                # Parameterize edge by origin and direction
                origin = edge.start
                direction = edge.end - edge.start
                direction = direction / np.linalg.norm(direction)
                boundary_edges.append((origin, direction))
                
        # Compute centroid of intersection points for regularization
        if intersections:
            points = [p for p, n in intersections]
            centroid = np.mean(points, axis=0)
        else:
            # Fallback if no intersections found (should not happen for active cubes)
            centroid = cube.center
            
        # Solve QEF: min e(v) = sum(d_II^2) + lambda_bound * sum(d_L^2) + lambda_reg * d_q^2
        # This is a linear least squares problem: min ||A v - b||^2
        # Construct matrix A and vector b
        
        A = []
        b = []
        
        # Term 1: Plane alignment error
        # d_II,i^2 = (n_i . (v - p_i))^2 = || n_i^T v - n_i^T p_i ||^2
        for p, n in intersections:
            # n is a 3D vector
            A.append(n)
            b.append(np.dot(n, p))
            
        # Term 2: Boundary constraint
        # d_L,j^2 = || (v - o_j) - ((v - o_j) . d_j) d_j ||^2
        # This is the squared distance to the line.
        # The gradient of this term leads to a quadratic form.
        # Alternatively, we can express it as minimizing distance to line.
        # Let M_j = I - d_j d_j^T (projection matrix onto plane perpendicular to d_j)
        # Then d_L,j^2 = || M_j (v - o_j) ||^2 = || M_j v - M_j o_j ||^2
        # So we add rows M_j to A and M_j o_j to b, scaled by sqrt(lambda_bound)
        
        sqrt_lambda_bound = np.sqrt(lambda_bound)
        for o, d in boundary_edges:
            # Projection matrix onto plane perpendicular to d
            # M = I - d * d^T
            M = np.eye(3) - np.outer(d, d)
            # Add M to A, scaled by sqrt(lambda_bound)
            A.append(sqrt_lambda_bound * M[0, :])
            b.append(sqrt_lambda_bound * np.dot(M, o))
            A.append(sqrt_lambda_bound * M[1, :])
            b.append(sqrt_lambda_bound * np.dot(M, o))
            A.append(sqrt_lambda_bound * M[2, :])
            b.append(sqrt_lambda_bound * np.dot(M, o))
            
        # Term 3: Regularization
        # d_q^2 = || v - centroid ||^2
        # Scaled by sqrt(lambda_reg)
        sqrt_lambda_reg = np.sqrt(lambda_reg)
        A.append(sqrt_lambda_reg * np.eye(3)[0, :])
        b.append(sqrt_lambda_reg * centroid[0])
        A.append(sqrt_lambda_reg * np.eye(3)[1, :])
        b.append(sqrt_lambda_reg * centroid[1])
        A.append(sqrt_lambda_reg * np.eye(3)[2, :])
        b.append(sqrt_lambda_reg * centroid[2])
        
        A = np.array(A)
        b = np.array(b)
        
        # Solve least squares: v = (A^T A)^{-1} A^T b
        # Using numpy's least squares solver
        v, residuals, rank, s = np.linalg.lstsq(A, b, rcond=None)
        
        vertices[cube.id] = v
        
    return active_cubes, flags, vertices

def get_edge_axis(cube, point):
    """
    Determine which axis the edge containing 'point' is parallel to.
    This is a helper function to assign flags.
    """
    # Implementation depends on how cube edges are stored
    # For a cube centered at c with size s, edges are parallel to axes
    # Check which coordinate of point is close to the cube boundary
    # and which coordinate varies along the edge
    # This is a placeholder for the actual logic
    # In practice, one would check which edge segment the point lies on
    return 'x' # Placeholder
```

## Subtopics

### Quadric Error Function (QEF) for Sharp Features
Focuses on preserving sharp features during QEF minimization. Choose when sharp features are critical. [Link](1604-quadric-error-function-qef-for-sharp-features.md)

### QEF Data Accumulation
Handles the accumulation of QEF data from multiple intersections. Choose when dealing with complex intersection data. [Link](3712-qef-data-accumulation.md)

### Occupancy-Based Dual Contouring
Uses occupancy information to determine active cubes. Choose when occupancy grids are available. [Link](3799-occupancy-based-dual-contouring.md)

### Mass Point Centering for Dual Contouring
Uses mass points for vertex positioning. Choose when mass point centering is preferred. [Link](3939-mass-point-centering-for-dual-contouring.md)

### Brute-Force QEF Solver
Solves QEF without optimization heuristics. Choose for simplicity or small-scale problems. [Link](4403-brute-force-qef-solver.md)

### QEF Solver Initialization
Provides initial guesses for QEF solvers. Choose when convergence is an issue. [Link](4722-qef-solver-initialization.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Handles sharp features, intricate internal structures, and nonmanifold topology [S1].
*   **Limitations:** Susceptible to floating-point precision errors in edge intersection detection [S1].
*   **When to use:** When preprocessing data for TRELLIS.2 or similar frameworks requiring high-quality mesh extraction from GT meshes.
*   **When not to use:** When floating-point precision errors are critical and cannot be mitigated, or when the input is not a GT mesh.

## Background
TRELLIS.2 was introduced by Xiang et al. in 2025 [S1]. It is part of a framework for 3D AIGC (Artificial Intelligence Generated Content) data preprocessing. The method is used in the data preprocessing stage to generate active cubes and their associated geometric attributes conditioned on input images, which are then used to train a Sparse Voxel Diffusion Transformer. The 3D mesh is extracted using Dual Contouring after this preprocessing step [S1].

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
