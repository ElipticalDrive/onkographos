# Local Least Squares Gradient Estimation

Also called: Gradient Estimation via Local Least Squares, Gram Matrix Gradient Estimation, Tetrahedral Least Squares Gradient

## Overview
Local Least Squares Gradient Estimation computes the gradient of a Signed Distance Field (SDF) at a query point by fitting a local linear plane to the SDF values at the vertices of a surrounding tetrahedron. It improves upon standard finite differences by utilizing all four vertices simultaneously to minimize the residual error, providing a more robust and accurate normal estimate for ray marching [S1].

## How it works
**Inputs:**
*   A tetrahedron $\mathcal{T}_j$ with vertices $\mathbf{s}_A, \mathbf{s}_B, \mathbf{s}_C, \mathbf{s}_D$ enclosing the query point.
*   The SDF values $\phi_A, \phi_B, \phi_C, \phi_D$ at these vertices.

**Outputs:**
*   The estimated gradient vector $\nabla \phi^{(j)}$.

**Data Structures:**
*   $\Delta \mathbf{S}_j$: A $4 \times 3$ matrix where each row $i$ is the vector $(\mathbf{s}_i - \bar{\mathbf{s}}_j)$.
*   $\Delta \Phi_j$: A $4 \times 1$ vector where each element $i$ is $(\phi_i - \bar{\phi}_j)$.
*   $\mathbf{G}_j$: A $3 \times 3$ symmetric matrix (the local Gram matrix).
*   $\mathbf{W}_j$: A $3 \times 4$ matrix, the precomputed weight matrix $\mathbf{G}_j^{-1} \Delta \mathbf{S}_j^\top$.

**Steps:**
1.  Compute the centroid of the tetrahedron vertices:
    $$ \bar{\mathbf{s}}_j = \frac{1}{4} \sum_{i \in \{A,B,C,D\}} \mathbf{s}_i $$
2.  Compute the mean SDF value:
    $$ \bar{\phi}_j = \frac{1}{4} \sum_{i \in \{A,B,C,D\}} \phi_i $$
3.  Construct the centered coordinate matrix $\Delta \mathbf{S}_j$ and centered SDF vector $\Delta \Phi_j$:
    $$ \Delta \mathbf{S}_j = \begin{bmatrix} (\mathbf{s}_A - \bar{\mathbf{s}}_j)^\top \\ (\mathbf{s}_B - \bar{\mathbf{s}}_j)^\top \\ (\mathbf{s}_C - \bar{\mathbf{s}}_j)^\top \\ (\mathbf{s}_D - \bar{\mathbf{s}}_j)^\top \end{bmatrix}, \quad \Delta \Phi_j = \begin{bmatrix} \phi_A - \bar{\phi}_j \\ \phi_B - \bar{\phi}_j \\ \phi_C - \bar{\phi}_j \\ \phi_D - \bar{\phi}_j \end{bmatrix} $$
4.  Compute the local Gram matrix $\mathbf{G}_j$:
    $$ \mathbf{G}_j = \Delta \mathbf{S}_j^\top \Delta \mathbf{S}_j $$
5.  Solve for the gradient by inverting $\mathbf{G}_j$ and multiplying by the centered data:
    $$ \nabla \phi^{(j)} = \mathbf{G}_j^{-1} \Delta \mathbf{S}_j^\top \Delta \Phi_j $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_least_squares_gradient(vertices, sdf_values):
    """
    Compute gradient using local least squares on a tetrahedron.
    
    Args:
        vertices: List of 4 numpy arrays (3,), representing s_A, s_B, s_C, s_D.
        sdf_values: List of 4 floats, representing phi_A, phi_B, phi_C, phi_D.
        
    Returns:
        numpy array (3,), the estimated gradient vector.
    """
    # 1. Compute centroid and mean SDF
    s_centroid = sum(vertices) / 4.0
    phi_mean = sum(sdf_values) / 4.0
    
    # 2. Construct centered vectors/matrices
    # Delta_S is 4x3, Delta_Phi is 4x1
    delta_s = []
    delta_phi = []
    for i in range(4):
        diff_s = vertices[i] - s_centroid
        diff_phi = sdf_values[i] - phi_mean
        delta_s.append(diff_s)
        delta_phi.append(diff_phi)
        
    delta_s = np.array(delta_s) # Shape (4, 3)
    delta_phi = np.array(delta_phi) # Shape (4,)
    
    # 3. Compute Gram Matrix G = Delta_S^T * Delta_S
    # Shape (3, 3)
    G = delta_s.T @ delta_s
    
    # 4. Solve for gradient: grad = G^{-1} * Delta_S^T * Delta_Phi
    # Using solve for numerical stability instead of explicit inverse
    # rhs = Delta_S^T * Delta_Phi -> Shape (3,)
    rhs = delta_s.T @ delta_phi
    
    gradient = np.linalg.solve(G, rhs)
    
    return gradient
```

**Edge Cases and Pitfalls:**
*   **Degenerate Tetrahedra:** If the four vertices are coplanar or coincident, $\mathbf{G}_j$ becomes singular and non-invertible. In such cases, the method fails [G]. A fallback to standard finite differences or a regularization term (e.g., $\lambda I$ added to $\mathbf{G}_j$) is required [G].
*   **Precision:** The inversion of $\mathbf{G}_j$ can be sensitive to ill-conditioning if the tetrahedron is very flat. Using `np.linalg.solve` is preferred over explicit inversion for stability [G].

## Subtopics

### Finite Difference Normal Estimation
The baseline method using only axis-aligned neighbors. Simpler but less accurate for oblique surfaces. [Finite Difference Normal Estimation](1234-finite-difference-normal-estimation.md)

### Tetrahedral Normal Estimation
The parent method that selects the tetrahedron and samples vertices. Local Least Squares is the specific gradient computation strategy used within this framework. [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, dominated by a $3 \times 3$ matrix inversion (or solve).
*   **Memory Complexity:** $O(1)$ auxiliary storage.
*   **Strengths:** More accurate than finite differences for non-axis-aligned surfaces; utilizes all available local information [S1].
*   **Limitations:** Requires a tetrahedral mesh or a method to find enclosing tetrahedra; computationally heavier than simple finite differences; sensitive to degenerate tetrahedra [S1].
*   **When to use:** When high-quality normals are critical for ray marching stability and a tetrahedral structure is available.

## Background
The method is derived from the DCCVT (Differentiable Clipped Centroidal Voronoi Tessellation) framework, which applies robust projection and local least squares fitting to improve SDF optimization and rendering stability [S1]. It addresses instabilities that can occur with standard gradient estimates during optimization [S1].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
