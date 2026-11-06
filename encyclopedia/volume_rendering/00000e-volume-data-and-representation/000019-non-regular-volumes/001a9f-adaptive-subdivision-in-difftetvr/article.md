# Adaptive Subdivision in DiffTetVR

Also called: Adaptive Mesh Refinement, Adaptive tetrahedral subdivision, Gradient-based mesh refinement, Prism Splitting, Tetrahedral Subdivision

## Overview
Adaptive Subdivision in DiffTetVR is a training strategy that refines a tetrahedral mesh by locally subdividing elements around vertices exhibiting high color gradients, aiming to increase resolution in regions of high-frequency detail while avoiding wasted resolution in empty regions [S1, unofficial]. It distinguishes itself from uniform refinement by using accumulated absolute color gradients to select vertices for splitting and employs a Constraint Satisfaction Problem (CSP) solver to ensure topological compatibility when splitting the resulting triangular prisms into tetrahedra [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh (initially generated via regular grid conversion or Delaunay triangulation of SfM points [S1, unofficial]), accumulated absolute color gradients at each vertex, and a target number of tetrahedra or subdivision iterations.
*   **Output:** A refined tetrahedral mesh with higher density in high-gradient regions.

**Data Structures:**
The implementation relies on a mesh data structure (e.g., OpenVolumeMesh) storing vertices and tetrahedra. The CSP solver is implemented in C++ in `FlipSolver.cpp` [S1, unofficial]. The solver manages the splitting patterns of triangular prisms.

**Parameters:**
*   $\hat{N}$: The number of vertices with the largest gradient magnitudes selected for subdivision. In the reference implementation, this is fixed at 5% of the total vertices [S1, unofficial].
*   Epoch definition: One epoch consists of training with 800 renderings from the dataset (e.g., Tooth dataset) [S1, unofficial].

**Steps:**
The method operates within a coarse-to-fine training loop:

1.  **Color Optimization:** Optimize only vertex colors for one epoch.
2.  **Joint Optimization:** Jointly optimize vertex colors and positions for one epoch.
3.  **Gradient Accumulation:** Accumulate absolute color gradients at every vertex without performing optimization steps for one epoch.
4.  **Vertex Selection:** Identify the $\hat{N}$ vertices with the highest absolute color gradients.
5.  **Edge Splitting:** For each selected vertex, split all incident edges by inserting a new vertex at the midpoint.
6.  **Prism Formation:** Splitting a tetrahedron incident to a selected vertex $a$ results in:
    *   A new tetrahedron incident with the splitting vertex $a$.
    *   A truncated pyramid in the form of a triangular prism formed by the original non-selected vertices $b, c, d$ and their new midpoints $b', c', d'$ on edges $ab, ac, ad$ [S1, unofficial].
7.  **Prism Tetrahedralization (CSP):** Split each triangular prism into exactly three tetrahedra. The split direction of the quad faces of the prism is determined by a Constraint Satisfaction Problem (CSP) to ensure compatibility with adjacent prisms [S1, unofficial].
    *   **Split Categories:** Split edges are categorized as 'Rising' (R) or 'Falling' (F) based on the direction of the tessellation edge relative to the extruded prism face in a counter-clockwise manner [S1, unofficial].
    *   **Compatibility:** Two adjacent prisms sharing a quad face are compatible if one uses an R split and the other uses an F split [S1, unofficial].
    *   **CSP Constraints:**
        1.  The splits for one prism may not be FFF or RRR.
        2.  Two adjacent prisms must split a shared quad face using exactly one F and one R split [S1, unofficial].
8.  **Iteration:** Repeat steps 1-7 until a target amount of tetrahedra is reached.
9.  **Final Optimization:** In the final iteration, optimize only vertex colors to produce the final result [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_subdivision(mesh, gradients, target_tet_count):
    """
    Performs adaptive subdivision on a tetrahedral mesh based on color gradients.
    
    Args:
        mesh: Tetrahedral mesh object with vertices and tetrahedra.
        gradients: Array of absolute color gradient magnitudes for each vertex.
        target_tet_count: Integer, the target number of tetrahedra.
        
    Returns:
        Refined mesh.
    """
    # Initial setup assumes mesh is already initialized (grid or Delaunay)
    # and gradients are accumulated from previous training epochs.
    
    while mesh.num_tetrahedra() < target_tet_count:
        # Step 1: Color Optimization (One Epoch)
        optimize_vertex_colors(mesh, num_epochs=1)
        
        # Step 2: Joint Optimization (One Epoch)
        optimize_vertex_colors_and_positions(mesh, num_epochs=1)
        
        # Step 3: Gradient Accumulation (One Epoch)
        accumulate_absolute_color_gradients(mesh, num_epochs=1)
        
        # Step 4: Vertex Selection
        # Select top 5% of vertices by gradient magnitude
        num_vertices = mesh.num_vertices()
        num_to_select = int(0.05 * num_vertices)
        sorted_indices = argsort_descending(gradients)
        selected_vertices = sorted_indices[:num_to_select]
        
        # Step 5 & 6: Edge Splitting and Prism Formation
        new_vertices = []
        prisms_to_split = []
        
        for v_idx in selected_vertices:
            incident_tets = mesh.get_incident_tetrahedra(v_idx)
            for tet in incident_tets:
                # Identify the splitting vertex 'a' and others 'b, c, d'
                # Insert midpoints on edges ab, ac, ad
                midpoints = {}
                for neighbor in tet.vertices:
                    if neighbor != v_idx:
                        mid_idx = mesh.insert_vertex_at_midpoint(v_idx, neighbor)
                        midpoints[neighbor] = mid_idx
                
                # Form the new tetrahedron incident with 'a'
                # and the triangular prism formed by b,c,d and their midpoints
                new_tet = create_tetrahedron([v_idx] + [midpoints[v] for v in tet.vertices if v != v_idx])
                mesh.add_tetrahedron(new_tet)
                
                # Identify the prism vertices: b, c, d, b', c', d'
                prism_vertices = [tet.vertices[i] for i in range(1, 4)] + \
                                 [midpoints[tet.vertices[i]] for i in range(1, 4)]
                prisms_to_split.append(prism_vertices)
        
        # Step 7: Prism Tetrahedralization via CSP
        # Solve CSP to determine split directions (R/F) for all prisms
        # ensuring compatibility on shared quad faces.
        # This requires building a dual graph of adjacent prisms.
        split_patterns = solve_csp_for_prisms(prisms_to_split, mesh)
        
        for i, prism in enumerate(prisms_to_split):
            pattern = split_patterns[i]
            # Split prism into 3 tetrahedra based on pattern (R/F)
            # The specific vertex indices for the 3 tets depend on the pattern
            # and the lookup table provided in the source (Table 1 in [S1, unofficial]).
            tetrahedra = split_prism_into_tetrahedra(prism, pattern)
            for tet in tetrahedra:
                mesh.add_tetrahedron(tet)
        
        # Remove the original tetrahedra that were split (handled by mesh structure update)
        # Update gradients array if necessary for next iteration
        
    # Step 9: Final Optimization
    optimize_vertex_colors(mesh, num_epochs=1)
    
    return mesh
```

**Edge Cases and Pitfalls:**
*   **Vertex Position Instability:** Optimization of vertex positions is unstable, especially as the mesh becomes more subdivided and tetrahedra get smaller, increasing the chance of inversion [S1, unofficial]. It is only stable at very low learning rates ($1e-6$ and $5e-6$) [S1, unofficial].
*   **Recursive Subdivision:** Subdivision does not sufficiently lower local error, causing recursive subdivision of the same regions. This eventually halts optimization progress as CSP solving and data structure updates dominate computation time [S1, unofficial].
*   **Incompatible Splits:** Incompatible splits on shared quad faces lead to infinitesimally thin holes in the tet mesh. This occurs if adjacent prisms choose the same split direction (e.g., both R or both F) on a shared face [S1, unofficial].
*   **Performance:** After many epochs, solving CSPs and updating the data structure (e.g., OpenVolumeMesh) dominates computation time [S1, unofficial].

## Subtopics

### [Differentiable Mesh Refinement](984-differentiable-mesh-refinement.md)
Differentiable Mesh Refinement is a post-processing optimization step that improves the geometric fidelity of a triangular mesh by adjusting vertex positions to better fit an underlying implicit occupancy field. It distinguishes itself from topological refinement methods (such as [Recursive Subdivision](1304-recursive-subdivision.md)) by operating directly on the existing mesh topology, using face subsampling to maintain computational efficiency during the optimization loop. This method builds on [Adaptive Subdivision in DiffTetVR](807-adaptive-subdivision-in-difftetvr.md) by providing a final polishing step that decouples geometry adjustment from topological changes.

### [Recursive Subdivision](1304-recursive-subdivision.md)
Recursive Subdivision is a deterministic tetrahedral mesh generation and refinement technique for regular volume data. It recursively bisects tetrahedra by adding edge midpoints, following a fixed sequence of geometric cases (Step 1, Step 2, Step 3) to produce a hierarchical framework. Unlike octree decompositions that require complex stitching to handle resolution boundaries, this method guarantees conforming faces and well-shaped tetrahedra without element quality degradation during subdivision.

### [3-root-2 Subdivision](6931-3-root-2-subdivision.md)
3-root-2 subdivision is a hierarchical volume representation scheme that refines a mesh by doubling the total number of vertices per step, theoretically increasing vertex density by a factor of $\sqrt{2}$ in each spatial dimension. It solves the problem of coarse granularity in standard octree refinement, which multiplies vertex counts by 8 per step, by providing finer resolution control for multiresolution rendering. It is distinguished from its siblings by its specific geometric insertion sequence (centroid, then face centers, then edge midpoints) and its reliance on B-spline wavelets for data value updates.

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the CSP solving and data structure updates during subdivision. As the mesh refines, these operations become increasingly expensive, eventually dominating computation time and halting optimization progress [S1, unofficial].
*   **Memory Complexity:** Depends on the number of tetrahedra. The mesh structure must store vertices, tetrahedra, and adjacency information for the CSP solver.
*   **Measured Performance:** Color-only optimization achieves a peak PSNR of 33.5 on the Tooth dataset. Vertex position optimization with fixed $\beta=100$ and $\lambda=10$ is only stable at learning rates $1e-6$ and $5e-6$, achieving a significantly lower PSNR of approx. 26.9 [S1, unofficial].
*   **Strengths:** Targets high-frequency details efficiently by refining only where necessary. Uses absolute gradients for splitting decisions, which was found to result in better outcomes compared to other gradient metrics [S1, unofficial].
*   **Limitations:** Vertex position optimization is unstable and prone to inversion. Subdivision may not sufficiently lower local error, leading to redundant refinements. Does not model directional lighting or volume scattering [S1, unofficial].
*   **When to Use:** When the scene has sparse high-frequency details and uniform resolution is inefficient.
*   **When Not to Use:** When vertex position stability is critical or when computational resources are limited due to CSP overhead.

## Background
DiffTetVR (Differentiable Tetrahedral Volume Rendering) was introduced by Chris Mile et al. in 2026 [S1, unofficial]. The adaptive subdivision method was proposed to address the issue of wasted resolution in empty regions of the tetrahedral mesh. The prism splitting algorithm is based on the work of Erleben et al. [44], who described an algorithm for splitting a triangular prism mesh into a tetrahedral mesh. The initial tetrahedral meshes can be generated using a regular grid aligned with the scene bounding box (converted from hexahedral cells or using fTetWild/TetGen) or Delaunay triangulation of a sparse point cloud from SfM (e.g., COLMAP) [S1, unofficial].

## Sources
[S1] DiffTetVR: Differentiable Tetrahedral Volume Rendering, https://doi.org/10.48550/arxiv.2601.00114, fetched 2026-10-26 (unofficial)
[S2] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
