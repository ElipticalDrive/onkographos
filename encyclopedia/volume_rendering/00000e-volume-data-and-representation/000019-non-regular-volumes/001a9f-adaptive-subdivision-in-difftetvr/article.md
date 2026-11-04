# Adaptive Subdivision in DiffTetVR

Also called: Adaptive Mesh Refinement, Adaptive tetrahedral subdivision, Gradient-based mesh refinement, Prism Splitting, Tetrahedral Subdivision

## Overview
Adaptive Subdivision in DiffTetVR is a training strategy that refines a tetrahedral mesh by locally subdividing vertices with high accumulated color gradients, aiming to increase resolution in regions of high-frequency detail while avoiding wasted resolution in empty regions [S1, unofficial]. It distinguishes itself from static mesh approaches by dynamically adjusting the mesh topology during optimization based on rendering error signals, and from general adaptive refinement methods by using a specific prism-splitting algorithm with Constraint Satisfaction Problem (CSP) constraints to maintain mesh validity [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A tetrahedral mesh (initially generated via regular grid or Delaunay triangulation of SfM points [S1, unofficial]), a set of training images with camera poses, and optimization hyperparameters (learning rates, regularization strengths) [S1, unofficial].
*   **Output:** A refined tetrahedral mesh with updated vertex positions and colors, optimized for volume rendering [S1, unofficial].

**Data Structures**
The mesh is represented as an unstructured tetrahedral mesh. Vertices store 3D coordinates and color values. Tetrahedra store indices to their four vertices. Adjacency information (incident tetrahedra for each vertex, shared faces for each edge) is required to identify which tetrahedra to split and to solve the CSP for prism splitting [S1, unofficial]. The CSP solver is implemented in C++ in `FlipSolver.cpp` [S1, unofficial].

**Numbered Steps**
The refinement process is iterative and interleaved with optimization epochs. One epoch is defined as training with 800 renderings from the dataset (e.g., the Tooth dataset) [S1, unofficial].

1.  **Optimize Colors:** For one epoch, optimize only vertex colors [S1, unofficial].
2.  **Optimize Colors and Positions:** For the next epoch, jointly optimize vertex colors and positions [S1, unofficial].
3.  **Acculate Gradients:** For the subsequent epoch, accumulate absolute color gradients at every vertex without performing optimization steps [S1, unofficial]. Absolute gradients are used as they were found to result in better outcomes compared to other gradient metrics [S1, unofficial].
4.  **Select Vertices:** Identify the $\hat{N}$ vertices with the highest absolute color gradient magnitudes. In the reported experiments, $\hat{N}$ corresponds to the 5% of vertices with the highest gradients [S1, unofficial].
5.  **Split Edges:** For each selected vertex, split all incident edges by inserting a new vertex at the midpoint of each edge [S1, unofficial].
6.  **Split Tetrahedra:** Split the incident tetrahedra using the newly inserted vertices. Splitting a tetrahedron incident to vertex $a$ results in:
    *   A new tetrahedron incident with the splitting vertex $a$.
    *   A truncated pyramid in the form of a triangular prism formed by vertices $b, c, d, b', c', d'$ (where $b', c', d'$ are the new midpoints on edges $ab, ac, ad$) [S1, unofficial].
7.  **Solve CSP for Prism Splitting:** The triangular prism is split into exactly three tetrahedra. The split direction of the quad faces of the prism is determined by a Constraint Satisfaction Problem (CSP) to ensure compatibility with adjacent prisms [S1, unofficial].
    *   Split edges are categorized as 'Rising' (R) or 'Falling' (F) based on the direction of the tessellation edge relative to the extruded prism face in a counter-clockwise manner [S1, unofficial].
    *   Two adjacent prisms sharing a quad face are compatible if one uses an R split and the other uses an F split [S1, unofficial].
    *   CSP Constraints:
        1.  The splits for one prism may not be FFF or RRR [S1, unofficial].
        2.  Two adjacent prisms must split a shared quad face using exactly one F and one R split [S1, unofficial].
    *   A lookup table for vertex indices for different splitting patterns (FRR, RFR, FFR, RRF, FRF, RFF) is provided in the source, mapping tetrahedron indices to prism vertex indices [S1, unofficial].
8.  **Repeat:** Repeat steps 1-7 until a target amount of tetrahedra is reached [S1, unofficial].
9.  **Final Optimization:** In the final iteration, optimize only vertex colors to produce the final result [S1, unofficial].

**Formulas**
*   **Midpoint Calculation:** For an edge between vertices $u$ and $v$, the new midpoint vertex $m$ is calculated as:
    $$ m = \frac{u + v}{2} $$
*   **Gradient Selection:** Vertices $v_i$ are selected if their absolute gradient magnitude $|g_i|$ is among the top $\hat{N}$ values, where $\hat{N} = 0.05 \times V$ (for 5% selection) [S1, unofficial].

**Parameters**
*   **$\hat{N}$:** The number of vertices with the largest gradient magnitudes selected for subdivision. In experiments, this was set to 5% of the total vertices [S1, unofficial].
*   **Learning Rate for Vertex Positions ($l_{rp}$):** Must be very low (e.g., $1e-6$ or $5e-6$) to avoid instability. Higher rates cause degenerate tetrahedra [S1, unofficial].
*   **Regularization Strength ($\lambda$):** Used in vertex position optimization. Set to 10 in experiments [S1, unofficial].
*   **Softplus Parameter ($\beta$):** Used in regularization. Set to 100 in experiments [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Instability:** Optimization of vertex positions is unstable, especially as the mesh becomes more subdivided and tetrahedra get smaller, increasing the chance of inversion [S1, unofficial].
*   **Recursive Subdivision:** Subdivision does not sufficiently lower local error, causing recursive subdivision of the same regions. This eventually halts optimization progress as CSP solving and data structure updates dominate computation time [S1, unofficial].
*   **Incompatible Splits:** Incompatible splits on shared quad faces lead to infinitesimally thin holes in the tet mesh. This occurs if adjacent prisms choose the same split direction (e.g., both R or both F) on a shared face [S1, unofficial].
*   **Performance:** Unlike color-only optimization (PSNR ~33.5), vertex position optimization with fixed $\beta=100$ and $\lambda=10$ achieves a significantly lower PSNR of approx. 26.9 and is only stable at learning rates $1e-6$ and $5e-6$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_subdivision(mesh, images, cameras, target_tet_count, gradient_percent=0.05):
    """
    Performs adaptive subdivision on a tetrahedral mesh based on accumulated color gradients.
    
    Args:
        mesh: Initial tetrahedral mesh object with vertices and tetrahedra.
        images: List of training images.
        cameras: List of camera poses corresponding to images.
        target_tet_count: Target number of tetrahedra to reach.
        gradient_percent: Percentage of vertices with highest gradients to subdivide.
        
    Returns:
        Refined mesh with optimized vertex colors and positions.
    """
    # Initialize vertex colors and positions if not already done
    # mesh.vertices: list of [x, y, z, r, g, b]
    # mesh.tetrahedra: list of [v0, v1, v2, v3] indices
    
    current_tet_count = len(mesh.tetrahedra)
    
    while current_tet_count < target_tet_count:
        # Step 1: Optimize Colors
        optimize_colors(mesh, images, cameras)
        
        # Step 2: Optimize Colors and Positions
        optimize_colors_and_positions(mesh, images, cameras, lr_pos=1e-6, lambda_reg=10, beta=100)
        
        # Step 3: Accumulate Gradients
        gradients = accumulate_absolute_gradients(mesh, images, cameras)
        
        # Step 4: Select Vertices
        num_vertices_to_subdivide = int(len(mesh.vertices) * gradient_percent)
        sorted_indices = argsort_descending(gradients)
        selected_vertex_indices = sorted_indices[:num_vertices_to_subdivide]
        
        # Step 5 & 6: Split Edges and Tetrahedra
        new_vertices = []
        vertex_to_midpoint = {} # Map original vertex index to new midpoint vertex index for incident edges
        
        for v_idx in selected_vertex_indices:
            incident_edges = get_incident_edges(mesh, v_idx)
            for edge in incident_edges:
                u, w = edge
                # Create midpoint if not already created for this edge
                edge_key = tuple(sorted([u, w]))
                if edge_key not in vertex_to_midpoint:
                    mid_v = create_midpoint_vertex(mesh, u, w)
                    vertex_to_midpoint[edge_key] = mid_v
                    new_vertices.append(mid_v)
                else:
                    mid_v = vertex_to_midpoint[edge_key]
            
            # Split incident tetrahedra
            incident_tets = get_incident_tetrahedra(mesh, v_idx)
            for tet in incident_tets:
                # Identify the splitting vertex 'a' and others 'b', 'c', 'd'
                # 'a' is v_idx
                # 'b', 'c', 'd' are the other three vertices
                # Create new midpoints for edges ab, ac, ad
                # Split tetrahedron into one new tet and one prism
                # Split prism into three tets using CSP
                split_tetrahedron_csp(mesh, tet, v_idx, vertex_to_midpoint)
        
        # Update mesh data structures
        mesh.vertices.extend(new_vertices)
        # Remove old tetrahedra and add new ones
        # Update adjacency information
        
        current_tet_count = len(mesh.tetrahedra)
    
    # Step 9: Final Optimization
    optimize_colors(mesh, images, cameras)
    
    return mesh

def optimize_colors(mesh, images, cameras):
    """Optimize only vertex colors."""
    # Gradient descent on vertex colors using rendering loss
    pass

def optimize_colors_and_positions(mesh, images, cameras, lr_pos, lambda_reg, beta):
    """Jointly optimize vertex colors and positions."""
    # Gradient descent on vertex colors and positions
    # Apply regularization with lambda_reg and beta
    pass

def accumulate_absolute_gradients(mesh, images, cameras):
    """Accumulate absolute color gradients at every vertex."""
    gradients = zeros(len(mesh.vertices))
    for img, cam in zip(images, cameras):
        # Render and compute gradients
        # Accumulate absolute gradients
        pass
    return gradients

def create_midpoint_vertex(mesh, u, w):
    """Create a new vertex at the midpoint of edge (u, w)."""
    u_pos = mesh.vertices[u][:3]
    w_pos = mesh.vertices[w][:3]
    mid_pos = (u_pos + w_pos) / 2.0
    mid_color = (mesh.vertices[u][3:6] + mesh.vertices[w][3:6]) / 2.0
    new_idx = len(mesh.vertices)
    mesh.vertices.append([*mid_pos, *mid_color])
    return new_idx

def split_tetrahedron_csp(mesh, tet, split_vertex_idx, vertex_to_midpoint):
    """
    Split a tetrahedron incident to split_vertex_idx.
    Results in one new tetrahedron and one prism.
    The prism is split into three tetrahedra using CSP.
    """
    # Identify vertices: a=split_vertex_idx, b, c, d
    a = split_vertex_idx
    b, c, d = [v for v in tet if v != a]
    
    # Get midpoints for edges ab, ac, ad
    edge_ab = tuple(sorted([a, b]))
    edge_ac = tuple(sorted([a, c]))
    edge_ad = tuple(sorted([a, d]))
    
    ab_mid = vertex_to_midpoint[edge_ab]
    ac_mid = vertex_to_midpoint[edge_ac]
    ad_mid = vertex_to_midpoint[edge_ad]
    
    # New tetrahedron incident with 'a'
    new_tet_1 = [a, ab_mid, ac_mid, ad_mid]
    
    # Prism vertices: b, c, d, ab_mid, ac_mid, ad_mid
    # Split prism into three tetrahedra using CSP
    # Determine split directions (R/F) for each quad face of the prism
    # Solve CSP to ensure compatibility with adjacent prisms
    # Add the three resulting tetrahedra to mesh
    pass
```

## Subtopics

### [Differentiable Mesh Refinement](984-differentiable-mesh-refinement.md)
Differentiable Mesh Refinement is a post-processing optimization step that improves the geometric fidelity of a triangular mesh by adjusting vertex positions to better fit an underlying implicit occupancy field. It distinguishes itself from topological refinement methods (such as [Recursive Subdivision](1304-recursive-subdivision.md)) by operating directly on the existing mesh topology, using face subsampling to maintain computational efficiency during the optimization loop. This method builds on [Adaptive Subdivision in DiffTetVR](807-adaptive-subdivision-in-difftetvr.md) by providing a final polishing step that decouples geometry adjustment from topological changes.

### [Recursive Subdivision](1304-recursive-subdivision.md)
Recursive Subdivision is a deterministic tetrahedral mesh generation and refinement technique used within the DiffTetVR framework to create a hierarchical volume representation. It solves the problem of inefficient resolution distribution by recursively bisecting tetrahedra based on specific geometric cases, ensuring that the resulting mesh maintains high element quality without degradation. Unlike general adaptive refinement which may produce irregular or skewed elements, this method guarantees that recursively splitting a tetrahedron three times yields smaller tetrahedra with exactly the same shape and quality as the initial one, resulting in only well-shaped tetrahedra.

## Operating characteristics
*   **Time Complexity:** The refinement process involves optimization epochs, gradient accumulation, and CSP solving. CSP solving and data structure updates can dominate computation time after many subdivisions, causing optimization progress to halt [S1, unofficial].
*   **Memory Complexity:** Depends on the number of tetrahedra and vertices. As the mesh is subdivided, memory usage increases linearly with the number of elements [S1, unofficial].
*   **Measured Performance:** On the Tooth dataset, color-only optimization achieves a peak PSNR of 33.5. Vertex position optimization with fixed $\beta=100$ and $\lambda=10$ achieves a significantly lower PSNR of approx. 26.9 and is only stable at learning rates $1e-6$ and $5e-6$ [S1, unofficial].
*   **Scaling:** The method scales with the number of training images and camera poses. The number of subdivisions is controlled by the target tetrahedron count or gradient percentage [S1, unofficial].
*   **Strengths:** Increases resolution in regions of high-frequency detail, avoiding wasted resolution in empty regions [S1, unofficial].
*   **Limitations:** Optimization of vertex positions is unstable, especially as the mesh becomes more subdivided. Subdivision does not sufficiently lower local error, leading to recursive subdivision of the same regions. Incompatible splits can lead to infinitesimally thin holes [S1, unofficial].
*   **When to Use:** When high-frequency detail is present in specific regions of the volume and a coarse initial mesh is used.
*   **When Not to Use:** When vertex position optimization is required for stability, or when computational resources are limited due to CSP solving overhead [S1, unofficial].

## Background
DiffTetVR was introduced in a preprint titled "DiffTetVR: Differentiable Tetrahedral Volume Rendering" [S1, unofficial]. The method builds on DiffDVR by Weiss and Westermann [S1, unofficial]. The prism splitting algorithm is based on work by Erleben et al. [S1, unofficial]. Initial tetrahedral meshes can be generated using regular grids or Delaunay triangulation of sparse point clouds from SfM algorithms like COLMAP [S1, unofficial]. The CSP solver is implemented in C++ in `FlipSolver.cpp` [S1, unofficial].

## Sources
[S1] DiffTetVR: Differentiable Tetrahedral Volume Rendering, https://doi.org/10.48550/arxiv.2601.00114, fetched 2026-10-26 (unofficial)
[S2] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
