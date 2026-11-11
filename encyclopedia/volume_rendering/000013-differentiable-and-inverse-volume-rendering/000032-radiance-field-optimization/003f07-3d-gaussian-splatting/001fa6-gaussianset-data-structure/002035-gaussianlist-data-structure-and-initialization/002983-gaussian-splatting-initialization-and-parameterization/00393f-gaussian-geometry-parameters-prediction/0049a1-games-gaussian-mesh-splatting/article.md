# GaMeS (Gaussian Mesh Splatting)

Also called: Gaussian Mesh Splatting, Mesh-based adapting and modification of Gaussian Splatting

## Overview
GaMeS is a hybrid 3D representation that integrates mesh and Gaussian Splatting (GS) primitives, allowing the modification of Gaussian components in a manner analogous to mesh editing [S1]. Unlike standard GS where Gaussians are independent entities, GaMeS parameterizes each Gaussian component by the vertices of a mesh face, ensuring that means and covariances automatically adapt when mesh positions are modified [S1]. This distinguishes it from [Textured-GS](9436-textured-gs.md) and [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) by directly binding the Gaussian geometry to the mesh topology without requiring expensive pre-processing stages or SDF-based regularization [S1].

## How it works
GaMeS operates in two primary input scenarios: **Mesh-Initialized Input** (baseline) and **Mesh-Free Input** (Triangle Soup). In both cases, the core mechanism involves parameterizing Gaussian distributions using the geometric properties of triangle faces.

### Inputs and Outputs
*   **Inputs:**
    *   A 3D mesh defined by a set of faces $F_M$, where each face $V_i$ is a triangle with vertices $v_1, v_2, v_3 \in \mathbb{R}^3$.
    *   Alternatively, a set of standard GS primitives to be converted into a Triangle Soup.
*   **Outputs:**
    *   A set of parameterized Gaussians $G_M$ associated with the mesh faces, including mean $\hat{m}_V$, covariance $\hat{\Sigma}_V$, opacity $\sigma$, and spherical harmonic colors $c$.

### Data Structures
For a mesh with $n$ faces, GaMeS defines a fixed number $k$ of Gaussian components per face. The total number of Gaussians is $k \cdot n$.

**Per Face Data ($V_i$):**
*   Vertices: $v_1, v_2, v_3$ (3D coordinates).
*   Centroid: $m = \text{mean}(v_1, v_2, v_3)$.

**Per Gaussian Component Data ($G_{V,i,j}$ for face $i$, component $j \in [1, k]$):**
*   Trainable parameters: $\alpha_{1j}, \alpha_{2j}, \alpha_{3j} \in \mathbb{R}$ (barycentric coordinates) and $\rho_j \in \mathbb{R}^+$ (scale factor).
*   Constraint: $\alpha_{1j} + \alpha_{2j} + \alpha_{3j} = 1$.
*   Derived Geometry: Mean $m_V$, Covariance $\Sigma_V$.
*   Appearance: Opacity $\sigma$, SH coefficients $c$.

### Algorithm Steps

#### 1. Baseline: Mesh-Initialized Input
Given a mesh face $V = \{v_1, v_2, v_3\}$, the Gaussian components are placed on the surface.

1.  **Calculate Face Normal ($r_1$):**
    Compute the normal vector using the cross product of edge vectors:
    $$ r_1 = \frac{(v_2 - v_1) \times (v_3 - v_1)}{\|(v_2 - v_1) \times (v_3 - v_1)\|} $$
    [S1]

2.  **Calculate First Tangent ($r_1'$):**
    Define the vector from the face centroid $m$ to vertex $v_1$ and normalize it:
    $$ r_1' = \frac{v_1 - m}{\|v_1 - m\|} $$
    [S1]

3.  **Calculate Second Tangent ($r_2$):**
    Orthogonalize the vector from the centroid to vertex $v_2$ ($v_2 - m$) with respect to $r_1'$ using the Gram-Schmidt process:
    $$ \text{orth}(x; r_1, r_2) = x - \text{proj}(x, r_1) - \text{proj}(x, r_2) $$
    $$ \text{proj}(v, u) = \frac{\langle v, u \rangle}{\langle u, u \rangle} u $$
    $$ r_2 = \frac{\text{orth}(v_2 - m; r_1', \mathbf{0})}{\|\text{orth}(v_2 - m; r_1', \mathbf{0})\|} $$
    *Note: The source defines $r_2$ via Gram-Schmidt of $(v_2-m)$ against $r_1'$. Since $r_1'$ is the first vector, the projection is only onto $r_1'$.*
    [S1]

4.  **Calculate Third Vector ($r_3$):**
    Complete the orthonormal basis:
    $$ r_3 = \frac{\text{orth}(v_2 - m; r_1', r_2)}{\|\text{orth}(v_2 - m; r_1', r_2)\|} $$
    [S1]

5.  **Construct Rotation Matrix ($R_V$):**
    $$ R_V = [r_1, r_2, r_3] $$
    [S1]

6.  **Define Scaling Matrix ($S_V$):**
    $$ S_V = \text{diag}(s_1, s_2, s_3) $$
    Where:
    *   $s_1 = \epsilon$ (a small constant to avoid numerical problems, corresponding to the normal direction).
    *   $s_2 = \|m - v_1\|$ (distance from centroid to $v_1$).
    *   $s_3 = \langle v_2, r_3 \rangle$ (projection of $v_2$ onto $r_3$).
    [S1]

7.  **Compute Covariance ($\Sigma_V$):**
    $$ \Sigma_V = R_V S_V S_V^T R_V^T $$
    [S1]

8.  **Parameterize Gaussians:**
    For each of the $k$ Gaussians on the face, define trainable parameters $\alpha_{1j}, \alpha_{2j}, \alpha_{3j}$ and $\rho_j$.
    *   **Mean ($m_V$):** Convex combination of vertices:
        $$ m_V(\alpha_{1j}, \alpha_{2j}, \alpha_{3j}) = \alpha_{1j}v_1 + \alpha_{2j}v_2 + \alpha_{3j}v_3 $$
        [S1]
    *   **Covariance:** Scaled by $\rho_j$:
        $$ \Sigma_{G,j} = \rho_j \Sigma_V $$
        [S1]
    *   **Gaussian Component:**
        $$ G_{V,j} = \mathcal{N}(m_V, \rho_j \Sigma_V) $$
        [S1]

#### 2. Mesh-Free Input: Triangle Soup Initialization
When no mesh is available, GaMeS creates a "Triangle Soup" by converting standard GS primitives into mesh-like triangles.

1.  **Train Standard GS:**
    Initialize standard 3D Gaussian Splatting with flat splats, setting the first scaling parameter $s_1 = \epsilon$.
    [S1]

2.  **Estimate Triangle Vertices:**
    For each Gaussian component defined by mean $\hat{m}$, rotation $R = [r_1, r_2, r_3]$, and scaling $S = \text{diag}(\epsilon, s_2, s_3)$, define three vertices $V = [v_1, v_2, v_3]$:
    *   $v_1 = \hat{m}$
    *   $v_2 = \hat{m} + s_2 r_2$
    *   $v_3 = \hat{m} + s_3 r_3$
    [S1]

3.  **Freeze Vertices and Re-parameterize:**
    Freeze $V$ and re-parameterize the Gaussian $\hat{G}_V$ using the mesh-based method from the baseline.
    *   **New Mean:** $\hat{m}_V = v_1$
    *   **New Rotation Basis:**
        *   $\hat{r}_1 = \frac{(v_2 - v_1) \times (v_3 - v_1)}{\|(v_2 - v_1) \times (v_3 - v_1)\|}$
        *   $\hat{r}_3 = \text{orth}(v_3 - v_1; r_1, r_2)$ (Note: Source uses original $r_1, r_2$ for orthogonalization step, then normalizes).
        *   $\hat{r}_2$ is derived to complete the orthonormal basis (implied by context of $R_V$ construction).
    *   **New Scaling:**
        *   $\hat{s}_1 = \epsilon$
        *   $\hat{s}_2 = \|v_2 - v_1\|$
        *   $\hat{s}_3 = \langle v_3 - v_1, \hat{r}_3 \rangle$
    *   **New Covariance:**
        $$ \hat{\Sigma}_V = \hat{R}_V \hat{S}_V \hat{S}_V^T \hat{R}_V^T $$
    *   **Final Component:**
        $$ \hat{G}_V = \mathcal{N}(\hat{m}_V, \hat{\Sigma}_V) $$
    [S1]

4.  **Form Triangle Soup:**
    The final set $G_M$ consists of these parameterized Gaussians on the estimated mesh faces $\hat{M}$:
    $$ G_M = \bigcup_{V_i \in \hat{M}} \{ \hat{G}_{V_i}, \sigma_i, c_i \} $$
    [S1]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gram_schmidt_orthogonalize(x, r1, r2):
    """
    Performs one step of Gram-Schmidt orthogonalization.
    orth(x; r1, r2) = x - proj(x, r1) - proj(x, r2)
    """
    def proj(v, u):
        return (np.dot(v, u) / np.dot(u, u)) * u
    
    return x - proj(x, r1) - proj(x, r2)

def compute_face_basis(v1, v2, v3):
    """
    Computes the orthonormal basis (r1, r2, r3) and scaling factors (s1, s2, s3)
    for a triangle face defined by vertices v1, v2, v3.
    Returns: R (3x3 matrix), S (3x1 vector), centroid m
    """
    # Centroid
    m = (v1 + v2 + v3) / 3.0
    
    # r1: Normal vector
    edge1 = v2 - v1
    edge2 = v3 - v1
    normal = np.cross(edge1, edge2)
    r1 = normal / np.linalg.norm(normal)
    
    # r1_prime: Vector from centroid to v1
    vec_to_v1 = v1 - m
    r1_prime = vec_to_v1 / np.linalg.norm(vec_to_v1)
    
    # r2: Orthogonalize (v2 - m) against r1_prime
    vec_to_v2 = v2 - m
    orth_vec = gram_schmidt_orthogonalize(vec_to_v2, r1_prime, np.zeros(3))
    r2 = orth_vec / np.linalg.norm(orth_vec)
    
    # r3: Orthogonalize (v2 - m) against r1_prime and r2
    # Note: Source implies r3 is derived from the same vector (v2-m) orthogonalized against previous vectors
    # However, standard basis completion usually uses cross product or orthogonalizing v3.
    # Source text: "r3 = orth(v2 - m; r1, r2)" where r1 is normal.
    # Let's follow source strictly: orth(v2 - m; r1, r2)
    # But r1 in source formula for r3 is the normal (r1 from step 1), not r1_prime.
    # Let's re-read carefully: 
    # "r3 = orth(v2 - m; r1, r2)" where r1 is normal, r2 is the tangent calculated above.
    # This seems redundant if r2 is already orthogonal to r1. 
    # Actually, source says: "To obtain r3, we use the vector from the center to the second vertex... 
    # r3 = orth(v2 - m; r1, r2) / norm".
    # If r2 is orthogonal to r1, projecting onto r2 removes the component along r2.
    # Since v2-m is in the plane of the triangle, and r1 is normal, v2-m has no component along r1.
    # So proj(v2-m, r1) is 0.
    # So orth(v2-m; r1, r2) = (v2-m) - proj(v2-m, r2).
    # This results in a vector orthogonal to r2 in the plane.
    
    orth_vec_r3 = gram_schmidt_orthogonalize(vec_to_v2, r1, r2)
    r3 = orth_vec_r3 / np.linalg.norm(orth_vec_r3)
    
    # Scaling factors
    epsilon = 1e-6 # Small constant
    s1 = epsilon
    s2 = np.linalg.norm(m - v1)
    s3 = np.dot(v2, r3) # Source says <v2, r3>
    
    R = np.column_stack((r1, r2, r3))
    S = np.array([s1, s2, s3])
    
    return R, S, m

def create_gaussian_mesh_splat(mesh_faces, k_per_face, epsilon=1e-6):
    """
    Creates GaMeS representation from a mesh.
    
    Args:
        mesh_faces: List of triangles, each triangle is [[v1, v2, v3], ...]
        k_per_face: Integer k, number of Gaussians per face.
        epsilon: Small constant for s1.
        
    Returns:
        gaussians: List of Gaussian objects/parameters.
    """
    gaussians = []
    
    for face_vertices in mesh_faces:
        v1, v2, v3 = face_vertices
        
        # 1. Compute Basis and Covariance Structure
        R, S, m = compute_face_basis(v1, v2, v3)
        
        # Covariance Sigma_V = R * diag(s1, s2, s3)^2 * R^T
        S_diag = np.diag(S)
        Sigma_V = R @ (S_diag @ S_diag.T) @ R.T
        
        # 2. Create k Gaussians for this face
        for j in range(k_per_face):
            # Trainable parameters (initialized randomly or uniformly)
            # alpha must sum to 1
            alpha = np.random.dirichlet([1, 1, 1])
            rho = np.random.uniform(0.1, 1.0) # rho in R+
            
            # Mean: Convex combination
            mean = alpha[0] * v1 + alpha[1] * v2 + alpha[2] * v3
            
            # Covariance: Scaled by rho
            # Note: Source says N(mV, rho * Sigma_V). 
            # In standard GS, covariance is RSS^T R^T. Here Sigma_V is already RSS^T R^T.
            # So we scale the matrix.
            cov = rho * Sigma_V
            
            # Appearance parameters (initialized)
            opacity = np.random.uniform(0.0, 1.0)
            color_sh = np.random.randn(3, 16) # Example SH dim
            
            gaussians.append({
                'mean': mean,
                'covariance': cov,
                'opacity': opacity,
                'color_sh': color_sh,
                'alpha': alpha,
                'rho': rho,
                'face_vertices': face_vertices
            })
            
    return gaussians

def triangle_soup_from_gs(gs_primitives, epsilon=1e-6):
    """
    Converts standard GS primitives into Triangle Soup (GaMeS mesh-free input).
    
    Args:
        gs_primitives: List of GS objects with mean, rotation R, scaling S.
        
    Returns:
        mesh_faces: List of triangles [v1, v2, v3]
    """
    mesh_faces = []
    
    for prim in gs_primitives:
        m = prim['mean']
        R = prim['rotation'] # 3x3 matrix [r1, r2, r3]
        S = prim['scaling']  # [s1, s2, s3]
        
        # Define vertices
        v1 = m
        v2 = m + S[1] * R[:, 1] # s2 * r2
        v3 = m + S[2] * R[:, 2] # s3 * r3
        
        mesh_faces.append([v1, v2, v3])
        
    return mesh_faces
```

### Parameters
*   $\epsilon$: Small constant (e.g., $10^{-6}$) used for $s_1$ to prevent numerical instability in the normal direction scaling [S1].
*   $k$: Integer number of Gaussian components per mesh face. Determines density. Total Gaussians = $k \times n$ (where $n$ is number of faces) [S1].

### Edge Cases and Pitfalls
*   **Variable Face Sizes:** If mesh faces vary significantly in size, using a fixed $k$ per face leads to poor representation (too sparse on large faces, too dense on small ones). The recommended solution is to subdivide large faces into smaller parts before assigning Gaussians [S1].
*   **Degenerate Triangles:** If vertices are collinear, the cross product for the normal $r_1$ will be zero, causing division by zero. Standard mesh preprocessing should handle this.
*   **Numerical Precision:** The calculation of $s_3 = \langle v_2, r_3 \rangle$ relies on precise vector arithmetic. Using $\epsilon$ for $s_1$ is critical to avoid singular covariance matrices [S1].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(n \cdot k)$ where $n$ is the number of mesh faces. Rendering complexity is similar to standard GS, $O(G)$ where $G$ is the number of Gaussians, but with the overhead of computing means/covariances from mesh vertices if not precomputed.
*   **Memory Complexity:** $O(n \cdot k)$. The number of Gaussians is fixed by the mesh topology and $k$, unlike standard GS which may require hundreds of thousands of independent Gaussians.
*   **Strengths:**
    *   **Editability:** Allows direct manipulation of 3D objects via mesh editing tools. Modifying mesh vertices automatically updates Gaussian means and covariances [S1].
    *   **Efficiency:** Avoids expensive pre-processing stages (like SDF optimization in SuGaR) [S1].
    *   **Compactness:** Can represent complex geometry with fewer parameters if the mesh is a good approximation.
*   **Limitations:**
    *   **Dependency on Mesh Quality:** Performance degrades if the input mesh is a poor fit for the target object. Requires refinement during training [S1].
    *   **Fixed Topology:** Cannot easily add or remove topology (vertices/faces) during training without re-initialization or subdivision [S1].
    *   **Triangle Soup Artifacts:** In mesh-free mode, the "Triangle Soup" is an approximation and may not perfectly capture surface continuity or complex curvature compared to a true mesh [S1].
*   **When to Use:** Ideal for applications requiring explicit 3D editing, such as character animation (e.g., using FLAME meshes for faces) or scene modification [S1]. Not suitable for highly unstructured or non-manifold geometry without preprocessing.

## Background
GaMeS was introduced to address the lack of a well-defined approach for conditioning Gaussian Splatting, which typically involves hundreds of thousands of independent components [S1]. The method was developed to allow real-time modification and adaptation of GS, similar to how meshes are edited [S1]. It builds on the concept of placing Gaussians on mesh surfaces, as seen in SuGaR, but removes the need for expensive two-stage training and SDF regularization [S1]. The method has been applied to human mesh initialization using the FLAME framework for facial expression modification [S1].

## Sources
[S1] GaMeS: Mesh-based adapting and modification of Gaussian Splatting, https://doi.org/10.1016/j.patcog.2026.114561, fetched 2026-10-29
