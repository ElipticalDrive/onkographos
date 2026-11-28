# Algorithm SD (Surface Detection)
Also called: Boundary surface extraction, Surface tracking algorithm
## Overview
Algorithm SD computes a boundary surface from a scene grid by tracking faces between voxels classified as inside or outside an object [S1]. It outputs a list of square faces, each defined by a center and a normal vector [S1]. It differs from its parent, Grid Boundary Detection, which only identifies boundary voxels, by actively tracking the specific interface and computing precise geometric properties (center location and normal) for rendering [S1].
## How it works
**Inputs:** A scene grid $G_R$, a segmentation procedure $P$, a vector function $f$, and an initial boundary face $F_0$ [S1].
**Outputs:** A list of square faces, where each face $F$ has a center $C$ and a normal vector $N$ [S1].

**Data Structures:**
*   **Scene Grid $G_R$:** Contains density values or feature data at voxel vertices [S1].
*   **Face $F$:** Defined by its center $C$ and normal $N$ [S1].
*   **Voxel $X$:** Identified by integer coordinates [S1].

**Steps:**
1.  **Initialization:** Start with a known boundary face $F_0$ [S1].
2.  **Neighbor Identification:** Given a voxel $X$, a face $F$, and a direction, determine neighboring voxels $X_1$ and $X_2$ using lookup functions $N_1$ and $N_2$ [S1].
3.  **Feature Evaluation:** Evaluate the feature vector $f(X)$ at point $X$. If using trilinear interpolation, identify the 8 vertices of the smallest parallelepiped containing $X$ in $G_R$ and interpolate densities [S1].
4.  **Center Calculation:** Compute the center $C$ location along the line segment connecting the centers of $X_1$ and $X_2$. The location is determined by finding where the decision boundary of $P$ lies [S1].
5.  **Normal Calculation:** Compute normal $N$ using six points equidistant from $C$: four at the centers of the edges of $F$, and two along the line segment connecting $X_1$ and $X_2$ [S1].
6.  **Density Sampling:** Determine the density at each of the six points using densities of 16 voxels in $G_R$ (8 with $x_3$ greater than the point's $x_3$, 8 with less) [S1].
7.  **Rendering:** Apply standard visibility and shading calculations to the generated surface faces [S1].

**Formulas:**
*   **Center Distance:** For a comparator threshold $T$ and linear interpolant $f$, the distance $d$ from $X_1$ to center $C$ is:
    $$ d = \frac{T - f(X_1)}{f(X_2) - f(X_1)} $$
    [S1]
*   **Shading:** The shading associated with a face $F$ is given by:
    $$ S(F) = k [k_d f_d(N, L) + k_s f_s(N, L)] e(d_F) + A $$
    where $k, k_d, k_s, A$ are constants, $f_d$ and $f_s$ are functions determining diffuse and specular components, and $e(d_F)$ is an attenuation factor [S1].

**Parameters:**
*   **Threshold $T$:** Used in comparator-based segmentation [S1].
*   **Interpolation:** Trilinear is common; higher-degree interpolants (e.g., cubic-spline) require larger subsets of points [S1].

**Edge Cases and Pitfalls:**
*   **Binary Scenes:** For binary scenes with different grid systems, use a scalar function $f$ estimating shortest distance to boundary (negative outside, positive inside). Classification $P$ checks the sign of $f(X)$ [S1].
*   **Interpolation Accuracy:** The choice of classifier $P$ and function $f$ is critical; faces must truly form a boundary surface for the algorithm to work correctly [S1].
*   **Normal Computation:** Up to 96 voxels may contribute to the computation of $N$ due to the 16-voxel requirement per sample point [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def algorithm_sd(scene_grid, segmentation_procedure, feature_function, initial_face):
    """
    Computes a boundary surface from a scene grid.
    
    Args:
        scene_grid: Grid G_R containing density/feature data.
        segmentation_procedure: Function P to classify voxels.
        feature_function: Function f to evaluate features at a point.
        initial_face: Face F_0 to start tracking.
        
    Returns:
        List of faces, each with center C and normal N.
    """
    faces = []
    queue = [initial_face]
    visited_faces = set()
    
    while queue:
        face = queue.pop(0)
        face_id = get_face_id(face)
        
        if face_id in visited_faces:
            continue
        visited_faces.add(face_id)
        
        # Step 2: Identify neighboring voxels
        # N1 and N2 are table lookups based on voxel X, face F, and direction
        voxel_x = get_voxel_from_face(face)
        direction = get_face_normal_direction(face)
        x1, x2 = lookup_neighbors(voxel_x, face, direction)
        
        # Step 3: Evaluate feature function
        # For trilinear interpolation, identify 8 vertices of parallelepiped
        f_x1 = evaluate_feature(scene_grid, x1, feature_function)
        f_x2 = evaluate_feature(scene_grid, x2, feature_function)
        
        # Step 4: Compute center C
        # Determine where decision boundary of P lies
        # If P is comparator with threshold T and f is linear interpolant:
        threshold = get_threshold(segmentation_procedure)
        d = (threshold - f_x1) / (f_x2 - f_x1)
        center_c = interpolate_center(x1, x2, d)
        
        # Step 5 & 6: Compute normal N
        # Six points: 4 at edge centers of F, 2 along segment X1-X2
        sample_points = get_equidistant_points(center_c, face, x1, x2)
        densities = []
        for point in sample_points:
            # Density at point based on 16 voxels in G_R
            # 8 with x3 > point.x3, 8 with x3 < point.x3
            density = compute_density_from_16_voxels(scene_grid, point)
            densities.append(density)
        
        normal_n = compute_normal_from_densities(densities)
        
        # Add face to output
        faces.append({
            'center': center_c,
            'normal': normal_n,
            'id': face_id
        })
        
        # Add neighboring faces to queue for tracking
        # Implementation detail: determine adjacent faces to current face
        # and add unvisited ones to queue
        add_adjacent_faces_to_queue(queue, face, visited_faces)
        
    return faces

def evaluate_feature(grid, voxel, feature_func):
    """
    Evaluates feature function f at a voxel or interpolated point.
    If trilinear, interpolates 8 vertices.
    """
    # Implementation depends on specific interpolant
    # For trilinear:
    vertices = get_parallelepiped_vertices(voxel)
    densities = [grid[v] for v in vertices]
    return trilinear_interpolate(vertices, densities, voxel)

def compute_density_from_16_voxels(grid, point):
    """
    Computes density at a point using 16 surrounding voxels.
    8 with x3 > point.x3, 8 with x3 < point.x3.
    """
    # Identify 16 voxels based on point coordinates
    voxels = get_16_surrounding_voxels(grid, point)
    # Interpolate density from these 16 voxels
    return interpolate_from_16_voxels(voxels, grid)
```
## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Can compute surfaces on a grid coarser or finer than the input scene grid [S1]. Uses standard visibility and shading techniques [S1].
*   **Limitations:** Accuracy depends heavily on the choice of classifier $P$ and function $f$ [S1]. Normal computation can involve up to 96 voxels, potentially impacting performance [S1].
*   **When to use:** When a precise boundary surface representation is needed for rendering, especially when the input and output grid systems differ [S1].
*   **When not to use:** When the input and output grid systems are identical and simpler methods suffice, or when the choice of $P$ and $f$ is not well-defined [S1].
## Background
Algorithm SD was introduced in the context of surface and volume rendering in three-dimensional imaging [S1]. It is designed to work with both scenes and binary scenes, regardless of whether the input and output grid systems are different or identical [S1]. The algorithm relies on a segmentation procedure $P$ and a vector function $f$ to classify voxels and evaluate features [S1].
## Sources
[S1] Surface and volume rendering in three-dimensional imaging: A comparison, https://doi.org/10.1007/bf03168161, fetched 2026-10-26
