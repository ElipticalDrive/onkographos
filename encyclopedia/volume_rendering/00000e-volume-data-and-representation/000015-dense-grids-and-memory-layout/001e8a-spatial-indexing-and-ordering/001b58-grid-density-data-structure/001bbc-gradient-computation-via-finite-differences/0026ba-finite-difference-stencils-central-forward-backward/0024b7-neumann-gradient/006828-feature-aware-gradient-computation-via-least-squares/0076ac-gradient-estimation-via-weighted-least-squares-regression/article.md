# Gradient Estimation via Weighted Least Squares Regression

Also called: Meshless Gradient Estimation, Weighted Regression

## Overview
This method estimates the gradient of a scalar field at a query point by fitting a local polynomial function to data points in a spatial neighborhood using weighted least squares, then differentiating that function [S1]. It solves the problem of accurate gradient reconstruction in unstructured or irregular grids where standard finite difference stencils fail due to lack of connectivity or uniform spacing [S1]. Unlike its parent method, Feature-Aware Gradient Computation via Least Squares, which relies on explicit mesh connectivity to define the neighbor set, this method uses a spatial radius to define the neighborhood, making it "meshless" [S1]. It is distinguished from simple averaging methods by its ability to handle low-quality meshes and high-frequency fields through adaptive weighting and polynomial fitting [S1].

## How it works
**Inputs:**
- Scalar field values $\mathbf{b}$ at a set of neighbor positions $\{\mathbf{x}_i\}$.
- Query position $\mathbf{x}$ (typically the vertex or voxel center where the gradient is desired).
- Support radius $R$ defining the neighborhood.
- Weight function parameters (implicit in $\mathbf{W}$).

**Outputs:**
- Gradient vector $\nabla f(\mathbf{x})$.

**Data Structures:**
- **Design Matrix $\mathbf{X}$**: A matrix of size $N \times D$, where $N$ is the number of neighbors and $D$ is the number of basis function terms. For linear basis functions, $D=4$ (terms $1, x, y, z$) or $D=3$ (terms $x, y, z$). The entry $X_{ij}$ is the value of the $j$-th basis function evaluated at the $i$-th neighbor's position relative to $\mathbf{x}$.
- **Weight Matrix $\mathbf{W}$**: A diagonal matrix of size $N \times N$. The diagonal entry $W_{ii}$ is the weight assigned to the $i$-th neighbor based on its distance from $\mathbf{x}$.
- **Value Vector $\mathbf{b}$**: A column vector of size $N \times 1$ containing the scalar field values at the neighbor positions.

**Steps:**

1. **Identify Neighbors**: Find all data points $\mathbf{x}_i$ within distance $R$ from the query point $\mathbf{x}$. The radius $R$ is typically defined as $R = \alpha R_C$, where $R_C$ is the radius of the sphere circumscribing the immediate neighbors of the vertex in the underlying mesh connectivity, and $\alpha$ is a scaling factor [S1].
2. **Construct Basis Functions**: Define the basis vector $\mathbf{g}(\mathbf{x})$. Common choices are:
   - Linear 3D: $\mathbf{g}(\mathbf{x}) = [x, y, z]^\top$
   - Linear 4D (affine): $\mathbf{g}(\mathbf{x}) = [x, y, z, 1]^\top$ [S1]
3. **Compute Weights**: Calculate the weight $w_i$ for each neighbor $i$ based on the distance $d_i = \|\mathbf{x}_i - \mathbf{x}\|$. The specific weight function is not detailed in the sources, but $\mathbf{W}$ is a diagonal matrix of these weights [S1].
4. **Formulate Matrices**:
   - Construct $\mathbf{X}$ such that row $i$ is $\mathbf{g}(\mathbf{x}_i)^\top$.
   - Construct $\mathbf{W} = \text{diag}(w_1, \dots, w_N)$.
   - Set $\mathbf{b}$ as the vector of scalar values at $\mathbf{x}_i$.
5. **Solve for Coefficients**: Compute the coefficient vector $\mathbf{c}_x$ using the weighted least squares solution:
   $$ \mathbf{c}_x = (\mathbf{X}^\top \mathbf{W}^2 \mathbf{X})^{-1} \mathbf{X}^\top \mathbf{W}^2 \mathbf{b} \quad (16) $$
   Let $\mathbf{A}(\mathbf{x}) = \mathbf{X}^\top \mathbf{W}^2(\mathbf{x}) \mathbf{X}$ [S1].
6. **Compute Gradient**: Calculate the gradient $\frac{\partial f(\mathbf{x})}{\partial x_k}$ for each coordinate $k \in \{x, y, z\}$.

   **Exact Gradient (Full Derivative):**
   The sources provide the full derivative of the weighted least squares solution with respect to spatial coordinates $x_k$:
   $$ \frac{\partial f(\mathbf{x})}{\partial x_k} = \frac{\partial \mathbf{g}(\mathbf{x})^\top}{\partial x_k} \mathbf{c}_x - \mathbf{g}(\mathbf{x})^\top \mathbf{A}(\mathbf{x})^{-1} \left( \frac{\partial \mathbf{A}(\mathbf{x})}{\partial x_k} \mathbf{c}_x - \mathbf{X}^\top \frac{\partial \mathbf{W}^2(\mathbf{x})}{\partial x_k} \mathbf{b} \right) \quad (17) $$
   This accounts for the variation of the weight matrix $\mathbf{W}$ and the inverse matrix $\mathbf{A}^{-1}$ with respect to position [S1].

   **Approximate Gradient (First Term):**
   An approximation uses only the first term of the derivative:
   $$ \frac{\partial f(\mathbf{x})}{\partial x_k} \approx \frac{\partial \mathbf{g}(\mathbf{x})^\top}{\partial x_k} \mathbf{c}_x $$
   For linear basis functions, this is equivalent to the coefficients corresponding to the basis functions $x, y, z$ [S1]. This approximation is noted to be a good estimate and is equivalent to regression methods using linear basis functions [S1].

**Parameters:**
- **Support Radius Factor ($\alpha$)**: Defines $R = \alpha R_C$. A factor near 1.0 yields accuracy similar to mesh-based regression. Larger factors improve reconstruction momentarily until the scalar function deviates from the linear approximation [S1].
- **Basis Function Order**: Linear (3D or 4D) is standard. 4D regression includes the constant term [S1].

**Edge Cases and Pitfalls:**
- **Low-Quality Meshes**: For meshes with a stretch factor below 0.05, unweighted 3D regression performs poorly. Weighted 3D regression outperforms averaging methods in these cases [S1].
- **High-Frequency Fields**: For high-frequency functions (e.g., Marschner-Lobb), increasing the number of neighbors beyond an optimal point decreases accuracy. The reconstruction "flattens" the surface by approximating the region with a plane, deviating from the ground truth gradient [S1].
- **4D Regression Weighting**: For 4D regression, the effect of weighting has no apparent effect on accuracy [S1].
- **Smoothness Dependency**: Accuracy is highly dependent on scalar field smoothness. It performs much better for smooth functions (e.g., spherical) compared to high-frequency functions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gradient_estimation_wls(x_query, neighbors_positions, neighbors_values, radius_factor, R_C):
    """
    Estimate gradient at x_query using weighted least squares regression.
    
    Args:
        x_query: (3,) array, query position.
        neighbors_positions: (N, 3) array, positions of neighbor points.
        neighbors_values: (N,) array, scalar values at neighbor positions.
        radius_factor: float, scaling factor for support radius R = radius_factor * R_C.
        R_C: float, circumsphere radius of immediate mesh neighbors.
        
    Returns:
        grad: (3,) array, estimated gradient.
    """
    # 1. Identify Neighbors within radius R
    R = radius_factor * R_C
    distances = np.linalg.norm(neighbors_positions - x_query, axis=1)
    mask = distances <= R
    x_neighbors = neighbors_positions[mask]
    b = neighbors_values[mask]
    N = len(x_neighbors)
    
    if N == 0:
        return np.zeros(3) # [G] Fallback for empty neighborhood

    # 2. Construct Basis Functions (Linear 4D: [x, y, z, 1])
    # Relative positions to x_query
    rel_pos = x_neighbors - x_query
    # Basis matrix X: N x 4
    X = np.column_stack((rel_pos[:, 0], rel_pos[:, 1], rel_pos[:, 2], np.ones(N)))
    
    # 3. Compute Weights
    # [G] Standard inverse distance weighting is assumed as specific function not given
    # Add small epsilon to avoid division by zero
    eps = 1e-8
    weights = 1.0 / (distances[mask] + eps)
    W = np.diag(weights)
    
    # 4. Solve for Coefficients c_x
    # A = X^T W^2 X
    W2 = W @ W
    A = X.T @ W2 @ X
    
    # [G] Check for singularity of A
    if np.linalg.matrix_rank(A) < A.shape[1]:
        # Fallback to pseudo-inverse if singular
        c_x = np.linalg.pinv(A) @ (X.T @ W2 @ b)
    else:
        c_x = np.linalg.solve(A, X.T @ W2 @ b)
        
    # 5. Compute Gradient
    # Using the Approximate Gradient (First Term)
    # grad_k = d(g(x)^T)/dx_k * c_x
    # For linear basis [x, y, z, 1], d(g)/dx = [1, 0, 0, 0], d(g)/dy = [0, 1, 0, 0], etc.
    # The gradient components are simply the first 3 coefficients of c_x
    grad = c_x[:3]
    
    return grad
```

## Subtopics

### Meshless Gradient Estimation via Weighted Least Squares
This variant reconstructs gradients at vertices using connected neighbors without relying on the underlying mesh topology, approximating the scalar field locally [S1]. It uses a support radius factor to control neighborhood size; near 1, error is similar to mesh-based regression, while larger support improves reconstruction until the scalar function deviates from linear approximation [S1]. Accuracy is highly dependent on scalar field smoothness, performing much better for smooth functions (e.g., spherical) compared to high-frequency functions (e.g., Marschner-Lobb) [S1]. Increasing neighbors beyond an optimal point decreases accuracy by flattening the surface [S1].

## Operating characteristics
- **Time Complexity**: Dominated by the matrix inversion or solving of the linear system $(\mathbf{X}^\top \mathbf{W}^2 \mathbf{X})^{-1}$. For a fixed neighborhood size $N$, this is $O(N^3)$ for inversion, or $O(N^2)$ if using iterative solvers or precomputed inverses for fixed basis.
- **Memory Complexity**: $O(N \cdot D)$ to store the design matrix and weights, where $D$ is the number of basis terms (typically 3 or 4).
- **Measured Performance**: On low-quality meshes (stretch factor < 0.05), weighted 3D regression outperforms averaging methods and unweighted regression [S1]. For 4D regression, weighting has no apparent effect [S1].
- **Scaling**: Accuracy improves with neighborhood size up to a point, after which it degrades due to the "flattening" effect on high-frequency data [S1].
- **Strengths**: Handles irregular/unstructured grids; robust to low-quality mesh elements when weighted; meshless nature allows flexibility in neighborhood definition.
- **Limitations**: Computationally more expensive than simple finite differences; sensitive to neighborhood size selection; accuracy drops for high-frequency fields if too many neighbors are included.
- **When to Use**: Unstructured meshes, low-quality elements, or when mesh connectivity is unavailable or insufficient.
- **When Not to Use**: High-frequency scalar fields with large neighborhoods; structured grids where finite differences are sufficient and faster.

## Background
This method originates from research into gradient estimation for volume rendering on unstructured meshes. It builds upon the work of Ledergerber et al. [13] and Mavriplis [17], who discussed the effects of weighting in regression-based gradient reconstruction [S1]. The specific evaluation of meshless vs. mesh-based approaches and the impact of mesh quality (stretch factor) was detailed in a 2011 study comparing gradient estimation methods [S1].

## Sources
[S1] A Comparison of Gradient Estimation Methods for Volume Rendering on Unstructured Meshes, https://www.computer.org/csdl/journal/tg/2011/03/ttg2011030305/13rRUx0xPi5, fetched 2026-11-01
