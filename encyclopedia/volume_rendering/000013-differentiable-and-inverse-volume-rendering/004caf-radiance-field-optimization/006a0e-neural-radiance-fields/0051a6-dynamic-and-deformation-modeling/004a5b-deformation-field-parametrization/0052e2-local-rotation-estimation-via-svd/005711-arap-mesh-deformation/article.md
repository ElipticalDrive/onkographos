# ARAP Mesh Deformation

Also called: ARAP energy minimization, As-Rigid-As-Possible (ARAP) deformation

## Overview
ARAP Mesh Deformation is a technique for deforming a triangular mesh while preserving local rigidity, minimizing distortion energy by approximating local rotations [S1]. It solves the problem of transferring user-specified boundary edits to the interior of a mesh without excessive stretching or shearing [S1]. It differs from global rigid transformations by allowing local rotation estimation per vertex, and from linear blend skinning by avoiding skeletal dependencies, relying instead on mesh connectivity and cotangent weights [S1, G].

## How it works
**Inputs and Outputs:**
- Original mesh $S$ with vertex positions $v_i \in \mathbb{R}^3$ for $i=1 \dots n$ [S1].
- Connectivity information (adjacency list $N(i)$ for each vertex) [S1].
- Boundary conditions: deformed positions $v'_i$ for a subset of vertices (user edits) [S1].
- Deformed mesh $S'$ with vertex positions $v'_i$ for all $i$ [S1].

**Data Structures:**
- Adjacency list: $N(i)$ contains indices of vertices adjacent to $i$ [S1].
- Cotangent weights: $w_{ij}$ stored for each edge $(i,j)$ [S1].
- Rotation matrices: $R_i \in \mathbb{R}^{3 \times 3}$ for each vertex $i$ [S1].

**Formulas:**
The ARAP energy is defined as [S1]:
$$ E(S') = \sum_{i=1}^{n} \sum_{j \in N(i)} w_{ij} \| (v'_i - v'_j) - R_i(v_i - v_j) \|^2 $$
where:
- $w_{ij} = \frac{1}{2}(\cot \alpha_{ij} + \cot \beta_{ij})$ is the cotangent weight for edge $(i,j)$ [S1].
- $\alpha_{ij}, \beta_{ij}$ are the angles opposite to edge $(i,j)$ in the two triangles sharing it [S1].
- $R_i$ is the local rotation matrix at vertex $i$ [S1].

**Steps:**
1. **Precompute Weights:** Calculate $w_{ij}$ for all edges using the original mesh geometry [S1].
2. **Initialize:** Set initial rotations $R_i = I$ (identity) for all $i$ [G]. Fix boundary vertices $v'_i$ to user-specified positions [S1].
3. **Optimize Rotations:** For each vertex $i$, compute the optimal rotation $R_i$ that minimizes the energy term involving $i$ and its neighbors, given current $v'_j$ [S1]. This typically involves a Singular Value Decomposition (SVD) of a covariance matrix constructed from vectors $(v_i - v_j)$ and $(v'_i - v'_j)$ [G].
4. **Optimize Positions:** Solve for $v'_i$ for all free vertices by setting the gradient of $E(S')$ with respect to $v'_i$ to zero [S1]. This results in a linear system $A v' = b$, where $A$ is a sparse matrix derived from the cotangent weights and $R_i$, and $b$ incorporates boundary conditions and rotation terms [S1].
5. **Iterate:** Repeat steps 3 and 4 until convergence (change in energy or positions below a threshold) [S1].

**Parameters:**
- $w_{ij}$: The cotangent weight for edge $(i,j)$, calculated as $0.5 \times (\cot(\alpha_{ij}) + \cot(\beta_{ij}))$ [S1].
- $R_i$: The local rotation matrix at vertex $i$ [S1].
- Fixed points for optimization: All Gaussian kernels whose distance from the active handle is greater than $0.5s$, where $s$ is the scale of the scene's bounding box [S2].
- Handle displacement magnitude: Scaled to $0.2s$, where $s$ is the scale of the scene's bounding box [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def compute_cotangent_weights(vertices, faces):
    """Compute cotangent weights for all edges in a triangular mesh."""
    weights = {}
    for face in faces:
        i, j, k = face
        p_i, p_j, p_k = vertices[i], vertices[j], vertices[k]
        
        def angle_at(p1, p2, p3):
            v1 = p1 - p2
            v2 = p3 - p2
            cos_a = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
            cos_a = np.clip(cos_a, -1.0, 1.0)
            return math.acos(cos_a)
        
        alpha = angle_at(p_j, p_i, p_k)
        beta = angle_at(p_k, p_j, p_i)
        gamma = angle_at(p_i, p_k, p_j)
        
        cot_alpha = 1.0 / math.tan(alpha) if alpha != 0 else 0.0
        cot_beta = 1.0 / math.tan(beta) if beta != 0 else 0.0
        cot_gamma = 1.0 / math.tan(gamma) if gamma != 0 else 0.0
        
        edge_ij = (min(i, j), max(i, j))
        edge_jk = (min(j, k), max(j, k))
        edge_ki = (min(k, i), max(k, i))
        
        weights[edge_ij] = weights.get(edge_ij, 0.0) + 0.5 * cot_alpha
        weights[edge_jk] = weights.get(edge_jk, 0.0) + 0.5 * cot_beta
        weights[edge_ki] = weights.get(edge_ki, 0.0) + 0.5 * cot_gamma
        
    return weights

def arap_deformation(vertices, faces, adjacency, boundary_indices, boundary_positions, max_iter=100, tol=1e-6):
    n = len(vertices)
    v = np.array(vertices)
    v_prime = v.copy()
    R = [np.eye(3) for _ in range(n)]
    
    weights = compute_cotangent_weights(v, faces)
    fixed_mask = np.zeros(n, dtype=bool)
    fixed_mask[boundary_indices] = True
    
    for iteration in range(max_iter):
        # Step 1: Optimize Rotations
        for i in range(n):
            M_i = np.zeros((3, 3))
            for j in adjacency[i]:
                diff_orig = v[i] - v[j]
                diff_deformed = v_prime[i] - v_prime[j]
                w = weights.get((min(i, j), max(i, j)), 0.0)
                M_i += w * np.outer(diff_orig, diff_deformed)
            
            U, _, Vt = np.linalg.svd(M_i)
            R[i] = Vt.T @ U.T
            if np.linalg.det(R[i]) < 0:
                Vt[-1, :] *= -1
                R[i] = Vt.T @ U.T
                
        # Step 2: Optimize Positions
        A_rows = []
        b_rows = []
        free_indices = [i for i in range(n) if not fixed_mask[i]]
        free_to_idx = {i: idx for idx, i in enumerate(free_indices)}
        
        for i in free_indices:
            row = np.zeros(n)
            rhs = np.zeros(3)
            for j in adjacency[i]:
                w = weights.get((min(i, j), max(i, j)), 0.0)
                row[i] += w
                rhs += w * R[i] @ (v[i] - v[j])
                if j in free_to_idx:
                    row[j] -= w
                elif fixed_mask[j]:
                    rhs += w * v_prime[j]
            A_rows.append(row)
            b_rows.append(rhs)
            
        A_sparse = np.array(A_rows)
        b_sparse = np.array(b_rows).reshape(-1)
        
        v_prime_free = np.linalg.solve(A_sparse, b_sparse)
        for idx, i in enumerate(free_indices):
            v_prime[i] = v_prime_free[idx * 3 : (idx + 1) * 3]
            
        if np.linalg.norm(v_prime - v) < tol:
            break
            
    return v_prime
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Incorrectly fixing boundary vertices can lead to unrealistic deformations [G].
- **Degenerate Triangles:** Cotangent weights become undefined for degenerate triangles (zero area) [G]. Handle by skipping or smoothing weights [G].
- **Reflections:** SVD may produce a reflection ($\det(R) = -1$) [G]. Ensure proper handling to maintain orientation [G].

## Subtopics

### ARAP Tetrahedral Deformation
Applies As-Rigid-As-Possible (ARAP) geometric deformation to tetrahedral meshes under constraints from surface mesh deformation [S1]. It differs from triangular ARAP by extending the energy formulation to volumetric cells and using barycentric coordinates to transfer surface vertex displacements to tetrahedron vertices [S1]. Choose it when deforming volumetric representations or implicit fields that require internal consistency beyond the surface [S1]. See [Embedded Deformation (Stage I)](19906-embedded-deformation-stage-i.md).

## Operating characteristics
- **Time Complexity:** Each iteration requires $O(N)$ work for the rotation step and a sparse linear solve for the position step [S1, G].
- **Memory Complexity:** Proportional to $O(N + E)$ for storing weights, rotations, and the sparse cotangent Laplacian stiffness matrix [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales linearly with vertex count per iteration; sparse solvers handle large meshes efficiently [G].
- **Strengths:** Preserves local rigidity well, handles complex deformations, avoids skeletal dependencies [S1].
- **Limitations:** Iterative optimization can be slow for very large meshes; sensitive to boundary condition placement [S1, G].
- **When to Use:** When rigid deformation is desired without skeletal animation, or for transferring deformations from a surface mesh to a volume representation [S1].
- **When Not to Use:** When real-time performance is critical and iterative convergence is too slow, or when non-rigid stretching/shearing is intentionally desired [G].

## Background
ARAP mesh deformation was introduced in the context of NeRF editing, specifically in the paper "NeRF-Editing: Geometry Editing of Neural Radiance Fields" [S1]. It is used to transfer user edits from a triangular mesh to an implicit volume representation, enabling geometry editing of neural radiance fields [S1]. The method is implemented based on libigl [S2].

## Sources
[S1] NeRF-Editing: Geometry Editing of Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.01781, fetched 2026-10-26
[S2] Proxy-Free Gaussian Splats Deformation with Splat-Based Surface Estimation, https://doi.org/10.1109/3DV69130.2026.00155, fetched 2026-10-30
