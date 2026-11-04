# McGrids

Also called: Monte Carlo-Driven Adaptive Grids

## Overview
McGrids is an iso-surface extraction method that constructs adaptive, non-uniform tetrahedral grids using an iterative Monte Carlo process to estimate a probability density function (PDF) based on the implicit field, followed by a refinement step and mesh extraction via [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md). It solves the problem of inefficient uniform grid sampling by concentrating grid points near surfaces and geometric details, significantly reducing the number of implicit field queries and memory usage compared to uniform grids. Unlike [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md) or [Subgrid Pipeline](1640-subgrid-pipeline.md) which may rely on fixed or hierarchical structures, McGrids dynamically adapts the grid topology through probabilistic sampling and Centroidal Voronoi Tessellation (CVT) relaxation.

## How it works
**Inputs:**
*   Implicit field function $f(\mathbf{x})$ returning a scalar value.
*   Iso-value $\alpha$ (typically 0).
*   Bounding region (user-defined).
*   Terminating threshold $\tau$ (default $10^{-4}$, typical range $10^{-3}$ to $10^{-5}$).
*   Maximum number of Monte Carlo iterations $N_{iter}$ (optional, used if termination condition is not met).
*   Minimum tetrahedron volume constraint $V_{min}$ (optional, for fractal surfaces).

**Outputs:**
*   A set of vertices $\mathcal{V}$ forming a Delaunay tetrahedralization.
*   A mesh extracted via Marching Tetrahedra.

**Data Structures:**
1.  **Delaunay Tetrahedralization:** Stores grid points as vertices. Tetrahedra $\mathcal{T}$ are formed such that no point lies inside the circumsphere of any tetrahedron.
2.  **Voronoi Diagram (Dual):** One-to-one mapping to McGrid points. Voronoi cells $\Omega_i$ store the implicit values and density information.
3.  **Implicit Values:** Stored in Voronoi cells corresponding to vertices.

**Algorithm Steps:**

1.  **Initialization:**
    *   Define a continuous uniform distribution over the user-defined bounding region.
    *   Initialize an empty Delaunay tessellation.
    *   Insert initial boundary points to define the domain.

2.  **Monte Carlo Grid Generation (Iterative):**
    Repeat for $k = 1$ to $N_{iter}$:
    
    2.1. **PDF Estimation:**
    *   For each vertex $x_i$ with Voronoi cell $\Omega_i$, compute the discrete probability $p(i)$:
        $$ p(i) = d(x_i) \cdot \text{Vol}(\Omega_i) $$
        where $d(x_i)$ is the density value at point $x_i$ (initially uniform or updated from previous steps), and $\text{Vol}(\Omega_i)$ is the volume of the Voronoi cell.
    *   Normalize to create a cumulative distribution:
        $$ p'(i) = \frac{p(i)}{\sum_{j=1}^n p(j)} $$
    *   *Optimization:* When new points are inserted, only update the volumes of Voronoi cells adjacent to the newly inserted cells.

    2.2. **Weighted Random Sampling:**
    *   Sample a uniform random number $u \sim U[0, 1]$.
    *   Select a Voronoi region $\Omega_i$ based on the cumulative distribution $p'(i)$.
    *   **Tessellate $\Omega_i$ into tetrahedra.**
    *   Calculate the volume of each tetrahedron within $\Omega_i$ to form a discrete cumulative distribution.
    *   Sample a uniform number to select a specific tetrahedron $T_{sel} \subset \Omega_i$.
    *   Sample a new point $\mathbf{x}_{new}$ uniformly within $T_{sel}$.

    2.3. **CVT Relaxation:**
    *   Compute the centroid of the Voronoi region $\Omega_{new}$ corresponding to $\mathbf{x}_{new}$.
    *   Move $\mathbf{x}_{new}$ to this centroid position. This is Lloyd's method for stable CVT.

    2.4. **Insertion and Update:**
    *   Insert the relaxed point $\mathbf{x}_{new}$ into the Delaunay tessellation.
    *   Query the implicit field: $v_{new} = f(\mathbf{x}_{new})$.
    *   Store $v_{new}$ in the corresponding Voronoi cell.
    *   Update the PDF (Step 2.1) for the next iteration.

3.  **Refinement (Midpoint Insertion):**
    For each tetrahedron $T \in \mathcal{T}$:
    
    3.1. **Trimming (Void Culling):**
    *   If all four vertices of $T$ have implicit values with the same sign (all $> \alpha$ or all $< \alpha$), the tetrahedron does not intersect the surface.
    *   Set the probability density values of its incident vertices to 0 to prevent further sampling in void regions.

    3.2. **Midpoint Evaluation:**
    *   If vertices have mixed signs (surface intersects $T$):
        *   Perform linear interpolation on edges to find points $\mathbf{p}_j$ where $f(\mathbf{p}_j) = \alpha$.
        *   Calculate the geometric midpoint $\mathbf{m}$ by averaging the positions of these interpolated points:
            $$ \mathbf{m} = \frac{1}{N_{edges}} \sum \mathbf{p}_j $$
        *   Evaluate the true implicit value at the midpoint: $v_m = f(\mathbf{m})$.
        *   Calculate error: $e = |v_m - \alpha|$.

    3.3. **Insertion Condition:**
    *   If $e > \tau$ AND $\text{Vol}(T) > V_{min}$:
        *   Insert $\mathbf{m}$ into the McGrids.
        *   Apply CVT relaxation to $\mathbf{m}$ (move to centroid of its new Voronoi region).
        *   Query $f(\mathbf{m})$ and update the tessellation.
    *   Else, consider $T$ adequately refined.

4.  **Termination:**
    *   Stop refining when all tetrahedra meet the criteria ($e \le \tau$ or $\text{Vol}(T) \le V_{min}$).
    *   In extreme cases (e.g., fractal surfaces), enforce $V_{min}$ to prevent excessive refinement.

5.  **Mesh Extraction:**
    *   Apply the [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) to the final Delaunay tetrahedralization to extract the iso-surface mesh.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcgrids_extraction(implicit_field, alpha, bounding_box, threshold, max_iterations):
    # 1. Initialization
    # Initialize empty Delaunay tetrahedralization and Voronoi diagram
    delaunay = DelaunayTessellation()
    voronoi = VoronoiDiagram()
    
    # Insert boundary points to define the domain
    boundary_points = get_boundary_points(bounding_box)
    for p in boundary_points:
        delaunay.insert(p)
        voronoi.update(p)
        implicit_field.query(p) # Store f(p)

    # 2. Monte Carlo Generation
    for iteration in range(max_iterations):
        # 2.1 PDF Estimation
        # Calculate Voronoi cell volumes and update probabilities
        probabilities = []
        for vertex in delaunay.vertices:
            vol = voronoi.cell_volume(vertex)
            # d(x_i) is density at vertex; initially uniform, updated iteratively
            prob = vertex.density * vol 
            probabilities.append(prob)
        
        # Normalize probabilities
        total_prob = sum(probabilities)
        cum_probs = cumulative_sum(probabilities / total_prob)
        
        # 2.2 Weighted Random Sampling
        # Select a Voronoi region based on cumulative distribution
        r = random.uniform(0, 1)
        region_index = find_bin(cum_probs, r)
        target_voronoi_cell = voronoi.cells[region_index]
        
        # Tessellate the Voronoi region into tetrahedra
        region_tets = tessellate_voronoi_cell(target_voronoi_cell)
        
        # Select a tetrahedron based on volume
        tet_probs = [t.volume for t in region_tets]
        tet_cum_probs = cumulative_sum(tet_probs / sum(tet_probs))
        r_tet = random.uniform(0, 1)
        tet_index = find_bin(tet_cum_probs, r_tet)
        selected_tet = region_tets[tet_index]
        
        # Sample a point uniformly within the selected tetrahedron
        new_point = sample_uniform_in_tet(selected_tet)
        
        # 2.3 CVT Relaxation
        # Move point to centroid of its Voronoi region
        centroid = voronoi.centroid(new_point)
        new_point = centroid
        
        # 2.4 Insertion
        # Insert into Delaunay/Voronoi
        delaunay.insert(new_point)
        voronoi.update(new_point)
        
        # Query implicit field
        new_point.value = implicit_field.query(new_point)
        
        # Update PDF densities for next iteration (simplified: uniform density update or based on error)
        # In practice, density might be updated based on local error or kept uniform for sampling weight
        new_point.density = 1.0 

    # 3. Refinement
    # Identify tetrahedra intersecting the surface
    for tet in delaunay.tetrahedra:
        signs = [vertex.value > alpha for vertex in tet.vertices]
        
        if all(signs) or not any(signs):
            # Trimming: No intersection
            for vertex in tet.vertices:
                vertex.density = 0.0
            continue
            
        # Midpoint Insertion
        # Find intersection points on edges
        edge_intersections = []
        for edge in tet.edges:
            p1, p2 = edge.vertices
            if (p1.value - alpha) * (p2.value - alpha) < 0:
                # Linear interpolation
                t = (alpha - p1.value) / (p2.value - p1.value)
                interp_point = lerp(p1, p2, t)
                edge_intersections.append(interp_point)
        
        if not edge_intersections:
            continue
            
        # Calculate midpoint
        midpoint = sum(edge_intersections) / len(edge_intersections)
        
        # Evaluate error
        midpoint.value = implicit_field.query(midpoint)
        error = abs(midpoint.value - alpha)
        
        if error > threshold and tet.volume > MIN_VOLUME:
            # Insert midpoint
            delaunay.insert(midpoint)
            voronoi.update(midpoint)
            
            # CVT Relaxation for new point
            centroid = voronoi.centroid(midpoint)
            # Note: Re-inserting/adjusting might require re-triangulation
            # Simplified: move and re-insert
            midpoint.position = centroid
            delaunay.reinsert(midpoint)
            voronoi.update(midpoint)

    # 4. Mesh Extraction
    mesh = marching_tetrahedra(delaunay, alpha)
    return mesh
```

## Subtopics

### CVT Relaxation
A technique used within McGrids to enhance grid regularity. It moves sampled points to the centroid of their Voronoi regions (Lloyd's method), reducing interpolation error and accelerating convergence compared to non-relaxed Monte Carlo sampling.

### Midpoint Refinement
A strategy to densify grids near surfaces. It evaluates the implicit value at the geometric midpoint of surface-intersecting tetrahedra and inserts it if the error exceeds a threshold, providing a curvature-like approximation without explicit normal information.

## Operating characteristics
*   **Time Complexity:** Dominated by the iterative Monte Carlo process and Delaunay/Voronoi updates. Each iteration involves sampling, tessellation, and relaxation. The refinement step adds overhead proportional to the number of surface-intersecting tetrahedra.
*   **Memory Complexity:** Proportional to the number of grid points and tetrahedra. Significantly lower than uniform grids for the same accuracy, as points are concentrated near the surface.
*   **Measured Performance:** On the 'Lucy' model with threshold $10^{-3}$, McGrids with CVT required 15 iterations and 228,472 queries, while without CVT it required 58 iterations and 319,185 queries [S1]. On a dataset of complex meshes (resolution $256^3$ equivalent), McGrids (threshold $10^{-3}$) achieved Chamfer Distance (CD) 0.54, Normal Consistency (NC) 0.94, Time 2.12s, Memory 0.91G, compared to Marching Cubes (MC) (12.85s, 2.57G) and Neural Dual Contouring (NDC) (76.89s, 21.76G) [S1].
*   **Strengths:** Reduces implicit field queries and memory usage; produces high-quality meshes with rich geometric details; adaptive to complex shapes.
*   **Limitations:** Slower runtime than fixed-grid methods (like standard DMT or FlexiCubes) due to dynamic grid computation; does not assume normal information, potentially requiring denser grids than Dual Contouring for sharp features; incurs overhead for grid construction [S1].
*   **When to use:** When high accuracy and detail are required, and implicit field queries are expensive. Suitable for neural implicit fields and complex analytical SDFs.
*   **When not to use:** For low-resolution meshes or when implicit field queries are extremely cheap; conventional methods like Marching Cubes may be more efficient [S1].

## Background
McGrids was proposed in the paper "McGrids: Monte Carlo-Driven Adaptive Grids for Iso-Surface Extraction" [S1]. It addresses the limitations of uniform grids in iso-surface extraction by formulating grid construction as a probability sampling problem. The method leverages Delaunay tetrahedralization and Voronoi diagrams to guide the adaptive sampling process. It has been applied in differentiable multiview reconstruction pipelines, replacing mesh extraction components in methods like DMT and FlexiCubes to achieve more accurate reconstruction results [S1].

## Sources
[S1] McGrids: Monte Carlo-Driven Adaptive Grids for Iso-Surface Extraction, https://link.springer.com/chapter/10.1007/978-3-031-72998-0_8, fetched 2026-10-28
