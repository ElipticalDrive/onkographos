# Adaptive Delaunay Update

Also called: Delaunay Update Frequency

## Overview
Adaptive Delaunay Update is a technique for maintaining the topological validity of a tetrahedral mesh during optimization-based volume rendering, specifically when vertex positions are perturbed. It solves the problem of mesh degeneracy caused by small vertex movements that violate the Delaunay empty-sphere property. Unlike static triangulation methods, it selectively recomputes or updates the connectivity to preserve the geometric constraints required for correct volume integration [S1, unofficial]. It sets itself apart from full re-triangulation by limiting updates to specific iterations, balancing GPU acceleration speed with geometric accuracy [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A valid Delaunay tetrahedral mesh, current vertex positions, and an iteration counter.
- Output: An updated Delaunay tetrahedral mesh where the empty-sphere property is restored for any vertices that have moved.

**Data Structures:**
- **Mesh:** A collection of tetrahedra, each defined by 4 vertex indices.
- **Vertex Attributes:** Position $(x, y, z)$ and other optimization parameters (e.g., color, density).
- **Iteration Counter:** Tracks the number of optimization steps.

**Steps:**
1. **Vertex Update:** During optimization, update the position of vertices by a small displacement $\epsilon$.
2. **Validity Check:** For each moved vertex, check if it has entered the circumsphere of any neighboring tetrahedron.
   - A tetrahedron is Delaunay if its circumsphere contains no other vertices.
   - If a vertex $v$ enters the circumsphere of a neighbor, the local topology is invalid.
3. **Topological Flip (Pachner Move):** If a violation is detected, perform a topological flip to restore the Delaunay property.
   - In 3D, this involves flipping the shared face between two tetrahedra that form a non-convex hull or violate the empty-sphere condition.
   - The flip replaces the two tetrahedra sharing the face with two new tetrahedra formed by the diagonal swap.
4. **Periodic Full Update:** To avoid the computational cost of continuous geometric checks and flips, the full Delaunay triangulation is recomputed from scratch at fixed intervals.
   - **Update Frequency:** Recompute the Delaunay triangulation every 10 iterations [S1, unofficial].
   - This balances the speed of GPU-accelerated rendering with the accuracy of a valid mesh [S1, unofficial].

**Formulas:**
- **Circumsphere Check:** For a tetrahedron with vertices $v_1, v_2, v_3, v_4$, the circumsphere center $c$ and radius $r$ are computed. A vertex $v$ is inside if $\|v - c\| < r$.
- **Displacement:** $v_{new} = v_{old} + \epsilon$, where $\epsilon$ is the optimization step size [S1, unofficial].

**Parameters:**
- **Update Frequency:** 10 iterations [S1, unofficial].
- **Displacement $\epsilon$:** Small amount such that a vertex may enter the circumsphere of a neighboring tetrahedron [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Nearly Cospherical Points:** Flips involve sets of points that are nearly cospherical because the vertex was already nearby the circumsphere boundary prior to the move. The geometric configurations before and after the flip share an almost identical circumsphere, which can lead to numerical instability [S1, unofficial].
- **Precision:** Floating-point precision errors in circumsphere calculations can cause incorrect flip decisions. Guard clauses or epsilon thresholds may be needed [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_delaunay_update(mesh, vertices, iteration):
    """
    Updates the Delaunay mesh after vertex optimization.
    
    Args:
        mesh: List of tetrahedra, each is a tuple of 4 vertex indices.
        vertices: List of vertex positions (x, y, z).
        iteration: Current optimization iteration number.
        
    Returns:
        Updated mesh and vertices.
    """
    # Step 1: Update vertex positions (assumed to be done by optimizer)
    # vertices = optimizer.step(vertices)
    
    # Step 2: Check for periodic full update
    if iteration % 10 == 0:
        # Recompute Delaunay triangulation from scratch
        # This is expensive but ensures global validity
        mesh = compute_delaunay_triangulation(vertices)
        return mesh, vertices
    
    # Step 3: Incremental update for non-multiple-of-10 iterations
    # Identify moved vertices (simplified: assume all vertices might have moved)
    moved_vertices = get_moved_vertices(vertices)
    
    for v in moved_vertices:
        # Find neighboring tetrahedra containing v
        neighbors = find_tetrahedra_containing_vertex(mesh, v)
        
        for tet in neighbors:
            # Compute circumsphere of tet
            center, radius = compute_circumsphere(tet, vertices)
            
            # Check if any other vertex is inside this circumsphere
            for other_v in vertices:
                if other_v == v:
                    continue
                if distance(other_v, center) < radius:
                    # Violation detected: perform Pachner move (flip)
                    mesh = flip_tetrahedron(mesh, tet, other_v, vertices)
                    break # One flip per tet is usually sufficient for local fix
                    
    return mesh, vertices

def compute_circumsphere(tet, vertices):
    """
    Computes the center and radius of the circumsphere of a tetrahedron.
    """
    v1, v2, v3, v4 = [vertices[i] for i in tet]
    # Formula for circumsphere center and radius
    # ... (standard geometric computation)
    pass

def flip_tetrahedron(mesh, tet, violating_vertex, vertices):
    """
    Performs a topological flip (Pachner move) to restore Delaunay property.
    """
    # Identify the face shared by the two tetrahedra involved in the flip
    # Replace the two tetrahedra with two new ones
    # ... (topological update logic)
    pass
```

## Subtopics

### [Radiance Meshes](4929-radiance-meshes.md)
Radiance Meshes represent radiance fields using constant density tetrahedral cells produced via Delaunay tetrahedralization. Unlike Voronoi-based methods (e.g., RadFoam) that require complex polyhedral tessellation, or splatting methods (e.g., 3DGS) that rely on approximate sorting, Radiance Meshes use simple triangles native to hardware rasterizers to enable exact, fast volume rendering. It solves the trade-off between optimization dynamics and rendering speed by decoupling topology (Delaunay mesh) from attribute parameterization (Instant-NGP).

### [Delaunay Triangulation](11773-delaunay-triangulation.md)
Delaunay triangulation constructs a mesh from a set of points $S$ by maximizing the minimum angle of the triangles, thereby avoiding sliver elements and producing optimal meshes for interpolation. It is defined as the straight-line dual of the Voronoi diagram of $S$, where edges connect points whose Voronoi cells share a boundary. It differs from its sibling [Radiance Meshes](4929-radiance-meshes.md) by being a purely geometric connectivity structure derived from point proximity rather than a radiative transfer approximation, and it serves as the foundational connectivity for the parent [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md).

### [Delaunay Cover Refinement](13422-delaunay-cover-refinement.md)
Delaunay Cover Refinement is a mesh generation technique that iteratively subdivides a Delaunay triangulation to ensure all edge lengths are bounded by a user-defined threshold $h$. It solves the problem of generating a uniformly sampled point cloud from an initial point set, which serves as a prerequisite for subsequent voxelization or surface reconstruction steps. Unlike standard Delaunay Triangulation which only optimizes local geometry, this method enforces a global density constraint through recursive splitting.

### Delaunay Edge Flipping (Pachner Moves)
A method to form a Delaunay triangulation by flipping edges of triangles that are not Delaunay, specifically those where the circumsphere contains a vertex. In 3D, these edge-flips are called Pachner moves [S1, unofficial]. During optimization, take a valid Delaunay triangulation and move vertices by a small amount $\epsilon$. If a vertex enters the circumsphere of a neighboring tetrahedron, trigger a topological flip [S1, unofficial]. Flips involve sets of points that are nearly cospherical because the vertex was already nearby the circumsphere boundary prior to the move, and the geometric configurations before and after the flip share an almost identical circumsphere [S1, unofficial].

### Hybrid Rendering with Multi-Resolution Tetrahedral Mesh
Extends multi triangulation to support multiple visualization techniques simultaneously within a single data structure, allowing independent mesh adaptation per technique [S2]. Store one DAG per technique to control refinement and coarsening. Store three faceincidency entries per tetrahedron face (one for each technique). Triple the cross-patch face hash map. The renderer adapts the mesh to each technique independently, enabling direct volume rendering to use fewer tetrahedra than isosurface extraction [S2]. Each tetrahedron stores three faceincidency entries per face. The cross-patch face hash map is tripled. Three separate DAGs are stored, one per visualization technique [S2].

## Operating characteristics
- **Time Complexity:** Full Delaunay triangulation is $O(N \log N)$ or worse depending on the algorithm, where $N$ is the number of vertices. Incremental updates are local but require checking neighbors.
- **Memory Complexity:** $O(N)$ for storing vertices and tetrahedra. Hybrid rendering triples the face hash map and DAG storage [S2].
- **Measured Performance:** On a scene with 1.2 million tetrahedra: 19 ms rendering, 13 ms instant-NGP query, 143 ms backwards pass, 92 ms Delaunay triangulation computation [S1, unofficial].
- **Scaling:** The periodic update strategy (every 10 iterations) allows for efficient GPU acceleration by amortizing the cost of triangulation over multiple rendering steps [S1, unofficial].
- **Strengths:** Maintains geometric validity during optimization; efficient enough for real-time or near-real-time rendering when updates are periodic.
- **Limitations:** Numerical instability due to nearly cospherical points; full recomputation is expensive.
- **When to use:** When optimizing vertex positions in a tetrahedral mesh for volume rendering.
- **When not to use:** When static geometry is sufficient, or when real-time performance is critical and periodic updates are too slow.

## Background
The concept of adaptive Delaunay update is rooted in the need to maintain mesh quality during dynamic optimization processes in computer graphics. The specific application to volume rendering with Radiance Meshes was introduced in the paper "Radiance Meshes for Volumetric Reconstruction" [S1, unofficial]. The technique of using Pachner moves for Delaunay refinement is a standard computational geometry technique [S1, unofficial]. The hybrid rendering approach with multi-resolution meshes was proposed in "Segment-based tetrahedral meshing and rendering" [S2].

## Sources
[S1] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S2] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
