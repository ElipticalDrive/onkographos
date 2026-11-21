# Implicit Voxel Simplification with Error Metric

Also called: Novel error metric simplification, Zero-level set preservation simplification

## Overview
This method simplifies an implicit voxel model by coarsening the grid hierarchy where the signed distance function is smooth, minimizing the approximation error of the surface geometry [S1, unofficial]. It differs from standard geometric simplification by operating directly on the scalar field values to preserve the zero-level set (the surface) rather than mesh vertices, and it differs from lossy compression by providing a mathematically optimal least-squares fit for the coarse representation [S1, unofficial].

## How it works
**Inputs:** A fine-grid signed distance field $F_0$ defined on voxel vertices $V_0$, and a target coarse grid $V_1$ where $V_0 \subset V_1$ (each coarse voxel contains 8 fine voxels).
**Outputs:** A coarse-grid function $F_1$ that minimizes the integral squared error between the prolonged coarse function and the fine function.

**Data Structures:**
*   **Voxel Vertices:** Indexed using Morton codes (Z-order curve). Corner 0 is the minimum coordinate; indices increase along ascending x, y, z axes [S1, unofficial].
*   **Integral Matrix ($M$):** An $8 \times 8$ symmetric matrix precomputed as $\int_V B^T B dV$ over a unit cube, where $B$ are the barycentric weights [S1, unofficial].
*   **Prolongation Matrix ($P$):** Maps coarse vertex values to fine vertex values. For a single coarse voxel, this is a $8 \times 8$ identity-like mapping where coarse vertices are placed at the corresponding fine corners, and intermediate fine vertices are interpolated or set to zero depending on the specific prolongation scheme (typically $P$ acts as an embedding operator) [S1, unofficial]. *Note: The source defines $P_{1->0}$ generally but implies a least-squares solution for the coefficients.*

**Algorithm Steps:**

1.  **Precompute Integral Matrix $M$:**
    Calculate the $8 \times 8$ matrix $M$ where $M_{ij} = \int_V B_i(p) B_j(p) dV$. The values for a unit cube are:
    $$
    M = \begin{pmatrix}
    \frac{1}{27} & \frac{1}{54} & \frac{1}{54} & \frac{1}{108} & \frac{1}{54} & \frac{1}{108} & \frac{1}{108} & \frac{1}{216} \\
    \frac{1}{54} & \frac{1}{27} & \frac{1}{108} & \frac{1}{54} & \frac{1}{108} & \frac{1}{54} & \frac{1}{216} & \frac{1}{108} \\
    \frac{1}{54} & \frac{1}{108} & \frac{1}{27} & \frac{1}{54} & \frac{1}{108} & \frac{1}{216} & \frac{1}{54} & \frac{1}{108} \\
    \frac{1}{108} & \frac{1}{54} & \frac{1}{54} & \frac{1}{27} & \frac{1}{108} & \frac{1}{54} & \frac{1}{108} & \frac{1}{216} \\
    \frac{1}{54} & \frac{1}{108} & \frac{1}{108} & \frac{1}{54} & \frac{1}{27} & \frac{1}{54} & \frac{1}{108} & \frac{1}{216} \\
    \frac{1}{108} & \frac{1}{54} & \frac{1}{216} & \frac{1}{54} & \frac{1}{54} & \frac{1}{27} & \frac{1}{108} & \frac{1}{216} \\
    \frac{1}{108} & \frac{1}{216} & \frac{1}{54} & \frac{1}{108} & \frac{1}{108} & \frac{1}{108} & \frac{1}{27} & \frac{1}{54} \\
    \frac{1}{216} & \frac{1}{108} & \frac{1}{108} & \frac{1}{216} & \frac{1}{216} & \frac{1}{216} & \frac{1}{54} & \frac{1}{27}
    \end{pmatrix}
    $$
    *Correction from source text:* The source text lists specific fractions. The last element in the first row is cited as `1/218` in the raw text but `1/216` in the matrix structure context and standard integration of Bernstein polynomials over a cube. The source explicitly lists `1/216` in the matrix display for the (0,7) and (7,7) positions in the provided LaTeX block, though the text snippet had a typo `1/218`. We use the matrix values provided in the LaTeX block [S1, unofficial].

2.  **Define Barycentric Weights:**
    For a point $p$ within a voxel, the weight $B_v(p)$ for vertex $v$ is:
    $$B(p) = \sum_{z=0}^1 \sum_{y=0}^1 \sum_{x=0}^1 b_{x,1}(p_x)b_{y,1}(p_y)b_{z,1}(p_z)$$
    where $b_{x,1}$ are first-order Bernstein polynomials [S1, unofficial].

3.  **Formulate Energy Function:**
    The total error energy $E$ is the sum of errors over all voxels $X$:
    $$E = \sum_X E_X = (P F_1 - F_0)^T M (P F_1 - F_0)$$
    where $F_1$ is the vector of coarse vertex values, $F_0$ is the vector of fine vertex values, and $P$ is the prolongation operator [S1, unofficial].

4.  **Solve for Coarse Values:**
    Minimize $E$ with respect to $F_1$. The gradient is:
    $$\nabla_{F_1} E = 2 P^T M P F_1 - 2 P^T M F_0$$
    Setting the gradient to zero yields the linear system:
    $$P^T M P F_1 = P^T M F_0$$
    Solve for $F_1$ using a linear solver (e.g., Cholesky decomposition or conjugate gradient) [S1, unofficial].

5.  **Simplification Decision:**
    Compare the residual energy $E$ against a threshold. If the error is acceptable, replace the 8 fine voxels with the single coarse voxel defined by $F_1$. Otherwise, keep the fine resolution or refine further [S1, unofficial].

**Parameters:**
*   **Error Threshold:** Not given in the sources.
*   **Grid Hierarchy:** Fixed 2:1 refinement ratio (8 fine voxels per coarse voxel) [S1, unofficial].

**Edge Cases:**
*   **Discontinuous Surfaces:** The method assumes a single continuous mesh. Discontinuous surfaces or multiple disjoint meshes violate the assumption that all stored surface data is continuous [S1, unofficial].
*   **Empty Regions:** Only voxels containing surface data (sign changes in the distance field) are stored. Empty regions are implicitly handled by the sparse structure [S1, unofficial].

## Operating characteristics
*   **Complexity:** Solving the linear system $P^T M P F_1 = P^T M F_0$ is $O(1)$ per coarse voxel if solved independently, or $O(N)$ for the whole grid if solved as a global system. The precomputation of $M$ is constant time.
*   **Memory:** Reduces memory usage by storing only coarse vertices where the error metric allows, leveraging the sparse octree structure [S1, unofficial].
*   **Strengths:** Preserves the zero-level set geometry optimally in the least-squares sense; integrates naturally with sparse octree storage [S1, unofficial].
*   **Limitations:** Assumes continuous single-mesh surfaces; requires precomputation of the integral matrix; sensitive to the sign-change detection for sparsity [S1, unofficial].

## Background
The method originates from a 2026 student thesis by an author at Utrecht University, titled "Level-of-Detail Independent Voxel-Based Surface..." [S1, unofficial]. It addresses the memory constraints of voxel-based rendering by introducing a simplification algorithm that coarsens voxels based on a novel error metric derived from barycentric interpolation [S1, unofficial].

## Sources
[S1] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
