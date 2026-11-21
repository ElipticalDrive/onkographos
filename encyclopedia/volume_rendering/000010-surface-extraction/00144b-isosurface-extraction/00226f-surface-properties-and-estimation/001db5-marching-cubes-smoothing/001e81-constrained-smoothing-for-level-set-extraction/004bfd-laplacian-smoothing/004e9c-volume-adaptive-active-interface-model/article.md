# Volume-Adaptive Active Interface Model

Also called: Active Contour Model for Volume Fractions, Adaptive Laplacian Smoothing with Volume Constraints

## Overview
The Volume-Adaptive Active Interface Model is a mesh smoothing technique that reconstructs piecewise linear material interfaces from volume fraction data by iteratively balancing internal Laplacian smoothing forces with external volume-adaptive forces [S1, unofficial]. It solves the problem of accurately representing multi-material junctions and complex topologies in volume-of-fluid (VOF) or embedded boundary simulations, where traditional methods like Piecewise Linear Interface Calculation (PLIC) fail due to manifold binary segmentation constraints [S1, unofficial]. Unlike its parent [Laplacian Smoothing](3736-laplacian-smoothing.md), which only reduces noise, this method adaptively pushes the interface to match known material volumes, preventing the shrinkage artifact inherent in pure Laplacian smoothing [S1, unofficial].

## How it works
The method operates on a control point mesh defining the interface within a background grid of volume fraction data. It iteratively updates control point positions to minimize the discrepancy between reconstructed and known material volumes while maintaining surface smoothness.

### Inputs and Outputs
- **Inputs**:
  - A set of control points $P = \{p_i\}$ defining the piecewise linear interface (edges in 2D, triangles in 3D).
  - Volume fraction data for $M$ materials in a background grid.
  - Parameters: $\alpha$ (internal force weight), $\beta$ (external force weight), $\sigma$ (subdivision threshold).
- **Outputs**:
  - Updated control point positions $P'$ representing the reconstructed interface.
  - Updated reconstruction error $\epsilon$ for each material in each cell.

### Data Structures
- **Control Point**: Stores 3D position $p_i$.
- **Neighbor List**: For each $p_i$, a list of indices of neighboring control points (1-ring).
- **Face Material Marking**: Each face $f$ in the mesh is marked with the material ID $A$ it bounds or separates. In multi-material junctions, faces surrounding $p_i$ may be associated with different materials.
- **Volume Error**: $\epsilon(c, A)$, the signed difference between the known volume fraction of material $A$ in cell $c$ and the reconstructed volume.

### Algorithm Steps
1. **Compute Internal Force**: For each control point $p_i$, calculate the average position $d$ of its neighbors in the 1-ring. The internal force is:
   $$ F_{int} = d - p_i $$
   This force acts to smooth the surface [S1, unofficial].

2. **Compute External Force**: For each material $A$ present in the neighborhood of $p_i$:
   - Identify the set of faces $R_A$ surrounding $p_i$ that are marked with material $A$.
   - Calculate the sum of normals $\sum_{f \in R_A} \perp_A(f)$, where $\perp_A(f)$ is the normal of face $f$ oriented into material $A$.
   - Compute the external force for material $A$:
     $$ F_{ext}(A) = \epsilon(c, A) \sum_{f \in R_A} \perp_A(f) $$
     If $\epsilon(c, A) > 0$, material $A$ has too much volume, and the force pushes the interface to reduce it. If $\epsilon(c, A) < 0$, the force pushes to increase it [S1, unofficial].

3. **Select Material**: To handle multi-material junctions stochastically, select a single material $\xi$ randomly from the materials present in the neighborhood of $p_i$ [S1, unofficial].

4. **Compute Total Force**: Combine the internal and selected external forces:
   $$ F = \alpha F_{int} + \beta F_{ext}(\xi) $$
   where $\alpha$ and $\beta$ are weighting parameters [S1, unofficial].

5. **Update Position**: Move the control point:
   $$ p'_i = p_i + F $$

6. **Update Volume Error**: Incrementally update $\epsilon$ by calculating the oriented volume swept by the 1-ring of $p_i$ as it moves to $p'_i$.
   - In 2D, line segments sweep to triangles.
   - In 3D, triangles sweep to tetrahedra.
   - Clip these swept volumes against the local mesh cells.
   - Use the oriented volume of the clipped swept regions to update $\epsilon$ for the relevant materials and cells [S1, unofficial].

7. **Subdivision**: If any face size exceeds the threshold $\sigma$ (max line segment length in 2D, max triangle area in 3D), subdivide the face to maintain resolution [S1, unofficial].

8. **Convergence**: Decrease $\alpha$ and $\beta$ monotonically (e.g., proportional to the change in error) until convergence [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_adaptive_active_interface_model(control_points, volume_fractions, alpha, beta, sigma, max_iterations):
    """
    Reconstructs piecewise linear material interfaces from volume fraction data.
    
    Args:
        control_points: List of control point objects with position p_i and neighbor indices.
        volume_fractions: Grid of volume fractions for M materials.
        alpha: Weight for internal smoothing force.
        beta: Weight for external volume-adaptive force.
        sigma: Threshold for face subdivision (max length/area).
        max_iterations: Maximum number of iterations.
        
    Returns:
        Updated control_points.
    """
    # Initialize reconstruction error epsilon for each cell and material
    epsilon = compute_initial_volume_errors(control_points, volume_fractions)
    
    for iteration in range(max_iterations):
        # Decrease weights monotonically based on error change
        alpha, beta = adjust_weights(alpha, beta, epsilon)
        
        for p_i in control_points:
            # 1. Compute Internal Force
            neighbors = get_neighbors(p_i)
            if not neighbors:
                continue
            d = average_position(neighbors)
            F_int = d - p_i.position
            
            # 2. Identify materials in neighborhood
            materials_in_neighborhood = get_materials_around(p_i, volume_fractions)
            if not materials_in_neighborhood:
                continue
                
            # 3. Select one material randomly
            xi = random.choice(materials_in_neighborhood)
            
            # 4. Compute External Force for selected material
            R_xi = get_faces_around(p_i, material=xi)
            sum_normals = sum(face_normal(f, material=xi) for f in R_xi)
            epsilon_c_xi = epsilon[p_i.cell_id][xi]
            F_ext = epsilon_c_xi * sum_normals
            
            # 5. Compute Total Force
            F = alpha * F_int + beta * F_ext
            
            # 6. Update Position
            p_i.position += F
            
            # 7. Update Volume Error Incrementally
            swept_volumes = compute_swept_volumes(p_i, p_i.position) # Old pos to new pos
            update_epsilon(epsilon, swept_volumes, p_i.cell_id)
            
        # 8. Subdivision
        subdivide_faces(control_points, sigma)
        
        if is_converged(epsilon, alpha, beta):
            break
            
    return control_points

def compute_initial_volume_errors(control_points, volume_fractions):
    """
    Computes the initial signed error between known and reconstructed volumes.
    [G] Implementation detail: Iterate over all cells, compute volume of material regions
    defined by the initial mesh, and subtract from known volume fractions.
    """
    epsilon = {}
    for cell in volume_fractions.cells:
        epsilon[cell.id] = {}
        for material in volume_fractions.materials:
            known_vol = volume_fractions[cell.id][material]
            reconstructed_vol = compute_reconstructed_volume(cell, control_points, material)
            epsilon[cell.id][material] = known_vol - reconstructed_vol
    return epsilon

def compute_swept_volumes(p_i, new_pos):
    """
    Computes the oriented volume swept by the 1-ring of p_i moving to new_pos.
    [G] In 2D, sweep edges to triangles. In 3D, sweep triangles to tetrahedra.
    Return a map of (cell_id, material) -> oriented_volume.
    """
    swept = {}
    neighbors = get_neighbors(p_i)
    if is_2d():
        for edge in get_edges_around(p_i, neighbors):
            triangle = sweep_edge_to_triangle(edge, p_i.position, new_pos)
            clip_and_accumulate(swept, triangle)
    else:
        for face in get_faces_around(p_i, neighbors):
            tet = sweep_face_to_tetrahedron(face, p_i.position, new_pos)
            clip_and_accumulate(swept, tet)
    return swept

def clip_and_accumulate(swept, swept_geom):
    """
    Clips swept geometry against local mesh cells and accumulates oriented volume.
    [G] For each cell intersecting swept_geom, compute intersection volume.
    Add to swept[cell_id][material] based on material orientation.
    """
    for cell in intersecting_cells(swept_geom):
        vol = oriented_volume_of_intersection(swept_geom, cell)
        material = get_material_for_intersection(swept_geom, cell)
        if material is not None:
            swept[cell.id][material] += vol
```

### Parameters
- **$\alpha$**: Weighting of the internal force. Typically set proportional to $\beta$ and monotonically decreased relative to the change in error until convergence [S1, unofficial].
- **$\beta$**: Weighting of the external force. Decreased monotonically alongside $\alpha$ [S1, unofficial].
- **$\sigma$**: Threshold parameter controlling maximum surface face size (max line segment length in 2D, max triangle area in 3D). Large values disable subdivision [S1, unofficial].

### Edge Cases and Pitfalls
- **Shrinkage**: Laplacian smoothing tends to shrink curves. This is counter-balanced by the external force, which adaptively pushes the interface to match volume fractions rather than just increasing volume [S1, unofficial].
- **Multi-material Junctions**: The stochastic selection of a single material $\xi$ per iteration handles complex junctions where multiple volumetric forces would otherwise conflict [S1, unofficial].
- **Precision**: Incremental volume updates are used for speed, but clipping swept volumes against local mesh cells is required for accuracy [S1, unofficial].

## Subtopics

### [Discrete Multi-Material Interface Reconstruction](14152-discrete-multi-material-interface-reconstruction.md)
Discrete Multi-Material Interface Reconstruction (MIR) is a method to reconstruct piecewise linear material interfaces from volume fraction data by discretizing cells into subcells, labeling them with material IDs, and optimizing the labeling to minimize interface energy while preserving volume fractions. It improves upon the [Volume-Adaptive Active Interface Model](13766-volume-adaptive-active-interface-model.md) by producing interfaces that are continuous across cell boundaries and suitable for visualization, whereas the parent model and alternatives like Piecewise Linear Interface Calculation (PLIC) are discontinuous across cell boundaries. Unlike PLIC, which is ambiguous for three or more materials due to the ordering of its binary segmentations, MIR explicitly handles $n$-material problems through a discrete labeling optimization.

## Operating characteristics
- **Time Complexity**: Not given in the sources. The incremental update of volume errors is noted as much faster than recomputing the entire volume each update [S1, unofficial].
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**:
  - On the 2D 'A' letter dataset (202 cells, 17.5% mixed), the method achieved an average mixed cell error of 0.3924% [S1, unofficial].
  - On the 2D 4-material junction dataset (7 cells, 42.86% mixed), the method achieved an average mixed cell error of 0.0043% [S1, unofficial].
  - On the SF Bay bathymetry dataset (2582x10 grid, 6.13% mixed), the method achieved 0.7640% error per mixed cell, compared to 43.8410% for Meredith's method [S1, unofficial].
  - On the Swirler dataset (128^3 grid, lower 64^3 octant), the rule-based marching method generated topology for 71% of mixed cells, while the discrete boundary method handled the swirler blades and central screen [S1, unofficial].
- **Strengths**: Handles multi-material junctions and complex topologies better than PLIC; captures volume accuracy and smoothness better than Meredith's method; captures fine details like swirler blades [S1, unofficial].
- **Limitations**: Stochastic selection may lead to non-deterministic results; requires careful tuning of $\alpha$ and $\beta$; subdivision may be needed for large faces [S1, unofficial].
- **When to use**: When accurate volume preservation and smooth interfaces are required for multi-material VOF or embedded boundary simulations, especially in the presence of complex junctions [S1, unofficial].
- **When not to use**: When deterministic topology generation is strictly required without stochastic elements; when computational speed is critical and incremental volume updates are too slow for the specific hardware [S1, unofficial].

## Background
The Volume-Adaptive Active Interface Model was developed as part of research into scientific visualization techniques for volume fraction data and function fields [S1, unofficial]. It builds on the concept of active contour models (snakes) and Laplacian smoothing, adapting them to the specific constraints of volume fraction data in computational fluid dynamics (CFD) and embedded boundary methods [S1, unofficial]. The method was compared against PLIC, Meredith's method, and rule-based marching methods on various synthetic and real-world datasets, including VOF simulations of rising bubbles and low-swirl burners [S1, unofficial].

## Sources
[S1] Scientific visualization techniques for volume fraction data and function fields, https://escholarship.org/uc/item/41x9x1gm, fetched 2026-10-27 (unofficial)
