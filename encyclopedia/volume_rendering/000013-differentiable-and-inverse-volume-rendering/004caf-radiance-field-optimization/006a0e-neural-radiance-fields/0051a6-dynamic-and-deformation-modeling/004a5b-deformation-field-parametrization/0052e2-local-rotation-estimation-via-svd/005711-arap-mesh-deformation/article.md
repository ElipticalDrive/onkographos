# ARAP Mesh Deformation

Also called: ARAP energy minimization, As-Rigid-As-Possible deformation

## Overview
ARAP Mesh Deformation is a technique for deforming a triangular mesh while preserving local rigidity, minimizing distortion energy by approximating local rotations. It solves the problem of transferring user-specified boundary edits to the interior of a mesh without excessive stretching or shearing. It differs from global rigid transformations by allowing local rotation estimation per vertex, and from linear blend skinning by avoiding skeletal dependencies, relying instead on the mesh connectivity and cotangent weights [S1].

## How it works
**Inputs:**
- Original mesh $S$ with vertex positions $v_i \in \mathbb{R}^3$ for $i=1 \dots n$.
- Connectivity information (adjacency list $N(i)$ for each vertex).
- Boundary conditions: deformed positions $v'_i$ for a subset of vertices (user edits).

**Outputs:**
- Deformed mesh $S'$ with vertex positions $v'_i$ for all $i$.

**Data Structures:**
- Adjacency list: $N(i)$ contains indices of vertices adjacent to $i$.
- Cotangent weights: $w_{ij}$ stored for each edge $(i,j)$.
- Rotation matrices: $R_i \in \mathbb{R}^{3 \times 3}$ for each vertex $i$.

**Formulas:**
The ARAP energy is defined as [S1]:
$$ E(S') = \sum_{i=1}^{n} \sum_{j \in N(i)} w_{ij} \| (v'_i - v'_j) - R_i(v_i - v_j) \|^2 $$
where:
- $w_{ij} = \frac{1}{2}(\cot \alpha_{ij} + \cot \beta_{ij})$ is the cotangent weight for edge $(i,j)$.
- $\alpha_{ij}, \beta_{ij}$ are the angles opposite to edge $(i,j)$ in the two triangles sharing it.
- $R_i$ is the local rotation matrix at vertex $i$.

**Steps:**
1. **Precompute Weights:** Calculate $w_{ij}$ for all edges using the original mesh geometry.
2. **Initialize:** Set initial rotations $R_i = I$ (identity) for all $i$. Fix boundary vertices $v'_i$ to user-specified positions.
3. **Optimize Rotations:** For each vertex $i$, compute the optimal rotation $R_i$ that minimizes the energy term involving $i$ and its neighbors, given current $v'_j$. This typically involves a Singular Value Decomposition (SVD) of a covariance matrix constructed from vectors $(v_i - v_j)$ and $(v'_i - v'_j)$ [S1, Note: SVD step implied by parent article context and standard ARAP literature, but specific matrix construction not detailed in S1 text. We assume standard ARAP rotation update].
4. **Optimize Positions:** Solve for $v'_i$ for all free vertices by setting the gradient of $E(S')$ with respect to $v'_i$ to zero. This results in a linear system $A v' = b$, where $A$ is a sparse matrix derived from the cotangent weights and $R_i$, and $b$ incorporates boundary conditions and rotation terms.
5. **Iterate:** Repeat steps 3 and 4 until convergence (change in energy or positions below a threshold).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def arap_deformation(mesh, boundary_vertices, boundary_positions, max_iter=100, tol=1e-6):
    """
    mesh: object with vertices V (n x 3), faces, and adjacency list N
    boundary_vertices: list of vertex indices fixed to boundary_positions
    boundary_positions: list of 3D positions for boundary vertices
    """
    n = len(mesh.V)
    V_prime = mesh.V.copy()
    # Initialize rotations to identity
    R = [np.eye(3) for _ in range(n)]
    
    # Precompute cotangent weights
    weights = compute_cotangent_weights(mesh)
    
    for iteration in range(max_iter):
        # Step 1: Optimize Rotations
        for i in range(n):
            # Construct covariance matrix M_i = sum_{j in N(i)} w_ij * (V[i] - V[j]) * (V_prime[i] - V_prime[j]).T
            M_i = np.zeros((3, 3))
            for j in mesh.N[i]:
                diff_orig = mesh.V[i] - mesh.V[j]
                diff_deformed = V_prime[i] - V_prime[j]
                M_i += weights[(i,j)] * np.outer(diff_orig, diff_deformed)
            
            # SVD to find optimal rotation R_i
            U, S, Vt = np.linalg.svd(M_i)
            R[i] = Vt.T @ U.T
            # Handle reflection case if necessary (det(R[i]) < 0)
            if np.linalg.det(R[i]) < 0:
                Vt[-1, :] *= -1
                R[i] = Vt.T @ U.T

        # Step 2: Optimize Positions
        # Solve linear system A * V_prime = b
        # A is constructed from weights and R, b from boundary conditions and R
        A, b = build_linear_system(mesh, weights, R, boundary_vertices, boundary_positions)
        V_prime_flat = np.linalg.solve(A, b)
        V_prime = V_prime_flat.reshape(n, 3)
        
        # Check convergence
        if np.linalg.norm(V_prime - mesh.V) < tol:
            break
            
    return V_prime

def compute_cotangent_weights(mesh):
    weights = {}
    for face in mesh.faces:
        i, j, k = face
        # Calculate angles opposite to edges (i,j), (j,k), (k,i)
        # This requires a helper to get opposite vertices and angles
        # Simplified: assume helper function exists
        weights[(i,j)] = 0.5 * (cot(angle_opposite_to_ij) + cot(angle_opposite_to_ij_other_triangle))
        weights[(j,k)] = ...
        weights[(k,i)] = ...
    return weights

def build_linear_system(mesh, weights, R, boundary_vertices, boundary_positions):
    # Construct sparse matrix A and vector b
    # For each free vertex i:
    # sum_{j in N(i)} w_ij * (v'_i - v'_j) = sum_{j in N(i)} w_ij * R_i * (v_i - v_j)
    # Rearrange to form linear equations
    # This is a standard ARAP linear solve step
    # Not fully detailed in S1, but implied by "efficiently solved"
    pass
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Incorrectly fixing boundary vertices can lead to unrealistic deformations.
- **Degenerate Triangles:** Cotangent weights become undefined for degenerate triangles (zero area). [G] Handle by skipping or smoothing weights.
- **Reflections:** SVD may produce a reflection (det(R) = -1). Ensure proper handling to maintain orientation. [G]

## Subtopics

### ARAP Tetrahedral Deformation
Extends ARAP deformation from triangular meshes to tetrahedral meshes. It deforms the tetrahedral mesh $T$ under constraints from the surface mesh deformation. The optimization problem is $\min E(T')$ subject to $A t' = v'$, where $A$ is the barycentric weight matrix and $t'$ are deformed tetrahedron vertices. It involves finding which tetrahedron each triangular mesh vertex is located in and calculating barycentric coordinates. The problem is converted to linear equations using Lagrangian multipliers [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of iterations and the cost of solving the linear system. Each iteration involves SVD for rotations and a sparse linear solve for positions.
- **Memory Complexity:** Proportional to the number of vertices and edges for storing weights, rotations, and the linear system matrix.
- **Strengths:** Preserves local rigidity well, handles complex deformations, no need for skeletons.
- **Limitations:** Can be slow for large meshes due to iterative optimization. Sensitive to boundary conditions.
- **When to Use:** When rigid deformation is desired without skeletal animation, or for transferring deformations from a surface mesh to a volume representation [S1].

## Background
ARAP mesh deformation was introduced in the context of NeRF editing, specifically in the paper "NeRF-Editing: Geometry Editing of Neural Radiance Fields" [S1]. It is used to transfer user edits from a triangular mesh to an implicit volume representation, enabling geometry editing of neural radiance fields.

## Sources
[S1] NeRF-Editing: Geometry Editing of Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.01781, fetched 2026-10-26
