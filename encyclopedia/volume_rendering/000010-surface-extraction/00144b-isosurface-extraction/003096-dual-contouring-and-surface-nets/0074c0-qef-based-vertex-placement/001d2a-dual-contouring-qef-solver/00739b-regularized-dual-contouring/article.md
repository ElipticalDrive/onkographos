# Regularized Dual Contouring

Also called: DC with centroid bias, QEF regularization

## Overview
Regularized Dual Contouring modifies the standard Dual Contouring QEF solver to prevent vertex solutions from exploding to distant locations when surface gradients are nearly coplanar [S1]. It sets itself apart from standard Dual Contouring by adding a centroid bias term to the optimization objective, distinguishing it from [Mass Point Centering for Dual Contouring](3939-mass-point-centering-for-dual-contouring.md) which uses the centroid directly as the vertex position, and from [Quadric Error Function (QEF) for Sharp Features](1604-quadric-error-function-qef-for-sharp-features.md) which focuses on feature preservation rather than numerical stability [S1].

## How it works
**Inputs:**
*   Signed Distance Field (SDF) $s(x)$, computed via trilinear interpolation [S1].
*   Grid cube $e$ containing zero-crossings.
*   Set of zero-crossing points $\mathcal{Z}_e = \{u_e\}$ on the edges of cube $e$ [S1].
*   Regularization weight $\lambda$ [S1].

**Outputs:**
*   Vertex position $v_d$ for the dual vertex associated with cube $e$ [S1].

**Data Structures:**
*   $\mathcal{Z}_e$: List of 3D coordinates $u_e$ where the SDF crosses zero on the cube edges.
*   $\nabla s(u_e)$: Gradient vector at each zero-crossing point.

**Steps:**
1.  **Compute Gradients:** For each zero-crossing point $u_e \in \mathcal{Z}_e$, compute the gradient $\nabla s(u_e)$ via local differentiation over the SDF computed via trilinear interpolation [S1].
2.  **Compute Centroid:** Calculate the centroid $\overline{u_e}$ of the zero-crossing points:
    $$ \overline{u_e} = \frac{1}{|\mathcal{Z}_e|} \sum_{u_e \in \mathcal{Z}_e} u_e $$
3.  **Solve Regularized QEF:** Find $v_d$ that minimizes the regularized quadratic error function:
    $$ v_d = \underset{v_d}{\operatorname{argmin}} \left( \sum_{u_e \in \mathcal{Z}_e} \left( \nabla s(u_e) \cdot (v_d - u_e) \right)^2 + \lambda \| v_d - \overline{u_e} \|^2 \right) $$
    *Note: The source formula [S1] writes the linear term $\nabla s(u_e) \cdot (v_d - u_e)$ without squaring, but standard QEF minimization involves squared residuals. The source text explicitly cites Equation 10 as the regularized form. Given the context of "QEF" (Quadratic Error Function), the standard formulation is the sum of squared distances to tangent planes. The source notation $\sum \nabla s(u_e) \cdot (v_d - u_e)$ likely implies the squared norm or the linear system derivation thereof. We implement the standard least-squares interpretation of the QEF with the added centroid regularization term.*
    
    *Correction based on strict source adherence:* The source [S1] presents Equation 10 as:
    $$ v_d = \underset{v_d}{\operatorname{argmin}} \sum_{u_e \in \mathcal{Z}_e} \nabla s(u_e) \cdot (v_d - u_e) + \lambda |v_d - \overline{u_e}| $$
    This notation is ambiguous regarding squaring. However, QEF solvers typically solve $A^T A v = A^T b$. The term $\nabla s(u_e) \cdot (v_d - u_e)$ represents the distance from $v_d$ to the tangent plane at $u_e$. Minimizing the sum of these distances (without square) is non-standard for QEF. Standard QEF minimizes the sum of *squared* distances. The source likely omitted the square in the display or assumes the linear system formulation. We assume the standard QEF formulation (squared residuals) plus the L2 regularization term, as this is the standard "regularized QEF" approach in graphics literature referenced by the source [S1]. If the source strictly meant linear terms, it would not be a QEF. We proceed with the squared residual interpretation for the QEF part, as "QEF" stands for Quadratic Error Function.

    The linear system to solve is:
    $$ (A^T A + \lambda I) v_d = A^T b + \lambda \overline{u_e} $$
    Where $A$ is the matrix of normals $\nabla s(u_e)$ and $b$ is the vector of dot products $\nabla s(u_e) \cdot u_e$.

4.  **Solve Linear System:** Use a linear solver (e.g., least-squares solver) to find $v_d$ [S1].

**Parameters:**
*   $\lambda$: Regularization weight. Typical values ablated: 1 and 0.01 [S1].

**Pitfalls:**
*   If gradients are nearly coplanar, the standard QEF matrix $A^T A$ becomes ill-conditioned, causing the solution to explode to distant locations [S1].
*   The method is sensitive to the scale of the SDF inputs [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def regularized_dual_contouring(cube_edges, sdf_function, lambda_val):
    """
    Computes the dual vertex position for a cube using regularized QEF.
    
    Args:
        cube_edges: List of edge objects, each having a 'zero_crossing' (u_e) 
                    and 'gradient' (n_e) computed via local differentiation.
        sdf_function: Function to compute SDF via trilinear interpolation.
        lambda_val: Scalar weight for regularization.
        
    Returns:
        v_d: The computed vertex position.
    """
    # Step 1: Collect zero-crossings and gradients
    # Note: Gradients are approximated by local differentiation over SDF.
    # This step is assumed to be done externally or via automatic differentiation 
    # if using a framework like PyTorch as per [S1].
    points = []
    normals = []
    
    for edge in cube_edges:
        u_e = edge.zero_crossing
        # Compute gradient via local differentiation over SDF (trilinear)
        n_e = compute_gradient_sdf(u_e, sdf_function)
        
        points.append(u_e)
        normals.append(n_e)
        
    if not points:
        return None # No zero crossings in this cube

    # Step 2: Compute Centroid
    points_array = np.array(points)
    centroid = np.mean(points_array, axis=0)
    
    # Step 3: Build QEF System
    # Standard QEF: minimize sum( (n_i . (v - u_i))^2 )
    # Regularized: minimize sum( (n_i . (v - u_i))^2 ) + lambda * ||v - centroid||^2
    
    # A matrix where each row is the normal n_i
    A = np.array(normals)
    
    # b vector where each element is n_i . u_i
    b = np.array([np.dot(n, u) for n, u in zip(normals, points)])
    
    # Identity matrix for regularization
    I = np.eye(3)
    
    # Solve (A^T A + lambda I) v = A^T b + lambda * centroid
    ATA = A.T @ A
    ATb = A.T @ b
    
    # Add regularization term to the system
    # (A^T A + lambda I) v = A^T b + lambda * centroid
    lhs = ATA + lambda_val * I
    rhs = ATb + lambda_val * centroid
    
    # Step 4: Solve linear system
    # Using lstsq as per [S1]
    v_d, residuals, rank, s = np.linalg.lstsq(lhs, rhs, rcond=None)
    
    return v_d

def compute_gradient_sdf(point, sdf_func):
    """
    Computes gradient via local differentiation over SDF.
    [G] Implementation detail: Central differences or automatic differentiation.
    """
    # Example using central differences for demonstration
    h = 1e-5
    x, y, z = point
    sx = (sdf_func([x+h, y, z]) - sdf_func([x-h, y, z])) / (2*h)
    sy = (sdf_func([x, y+h, z]) - sdf_func([x, y-h, z])) / (2*h)
    sz = (sdf_func([x, y, z+h]) - sdf_func([x, y, z-h])) / (2*h)
    return np.array([sx, sy, sz])
```

## Subtopics

### Adaptive Regularizer Weight for Dual Contouring
An adaptive strategy to determine the regularization weight $\lambda$ per cube by iteratively doubling it until the QEF solution falls inside the cube or a limit is reached [S1]. Initial $\lambda$ is 0.01 and the limit is $10^6$ [S1]. This approach is very time-consuming to evaluate and hard to leverage in a general gradient-based mesh optimization framework [S1].

## Operating characteristics
*   **Time Complexity:** Solving the linear system is $O(N^3)$ for the matrix inversion or decomposition, where $N$ is the dimension (3 for 3D space), but the cost is dominated by the number of zero-crossings $|\mathcal{Z}_e|$ in the construction of $A$ and $b$. Adaptive $\lambda$ adds an iterative loop, significantly increasing time [S1].
*   **Memory Complexity:** Low, requiring storage for the local cube's zero-crossings and gradients [S1].
*   **Strengths:** Prevents vertex explosion in degenerate cases (coplanar gradients) [S1].
*   **Limitations:** Sensitive to SDF scale [S1]. Adaptive weight is computationally expensive [S1].
*   **When to use:** When standard Dual Contouring produces unstable vertices due to noisy or coplanar gradients [S1].

## Background
Originated from the need to stabilize Dual Contouring in gradient-based mesh optimization pipelines, specifically following the implementation in NDC source code [S1]. It was adapted in the context of flexible isosurface extraction for gradient-based mesh optimization [S1].

## Sources
[S1] Flexible Isosurface Extraction for Gradient-Based Mesh Optimization, https://arxiv.org/abs/2308.05371v1, fetched 2026-10-26
