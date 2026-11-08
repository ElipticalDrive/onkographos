# GMRES Solver

Also called: GMRES Solver for CT Reconstruction, GMRES Solver for Regularized Normal Equations, Generalized Minimal Residual Method, Generalized Minimal Residual Method for CT, Iterative CT Reconstruction, Matrix-Free Regularized Least-Squares Solver

## Overview
The GMRES Solver is an iterative Krylov subspace method used to solve the regularized normal equations arising in Compressive Volume Rendering Recovery [S1, unofficial]. It addresses the ill-posed inverse problem of reconstructing a volume from sparse or limited-angle sinogram data by minimizing a least-squares objective with Tikhonov regularization [S1, unofficial]. Unlike direct solvers that require explicit matrix inversion, GMRES operates in a matrix-free manner, making it suitable for large-scale tomographic reconstruction where the system matrix is too large to store [S1, unofficial]. It is distinguished from other local solvers by its global convergence properties over the entire image domain rather than block-wise partitioning [Parent].

## How it works
**Inputs:**
- `vol_geom`: Volume geometry object defining the reconstruction grid dimensions $v \times h$ [S1, unofficial].
- `proj_geom`: Projection geometry object defining the detector and source parameters [S1, unofficial].
- `projector_id`: Identifier for the forward projection operator (e.g., 'line', 'pixel') [S1, unofficial].
- `gNoisy_id`: Identifier for the measured sinogram data $g$ [S1, unofficial].
- `max_angle`: Maximum projection angle in radians [S1, unofficial].
- `n_angles`: Number of projection angles [S1, unofficial].
- `n_samples`: Number of detector samples per projection [S1, unofficial].
- `v`, `h`: Height and width of the reconstruction volume [S1, unofficial].
- `alpha`: Regularization strength parameter [S1, unofficial].
- `penalty`: String specifying the regularization type: "tikhonov" (0-order) or "gradient" (1st-order) [S1, unofficial].
- `maxiter`: Maximum number of iterations (default 700) [S1, unofficial].

**Outputs:**
- `f_alpha`: Reconstructed volume image reshaped to $(v, h)$ [S1, unofficial].

**Data Structures:**
- The system is represented as a `scipy.sparse.linalg.LinearOperator` object `A`. This object does not store the full matrix $A^T A + \alpha R$ but defines a `matvec` function that computes the product of the regularized normal matrix with a vector [S1, unofficial].
- The right-hand side vector is $b = A^T g$, computed using the adjoint projection operator `AT_y` [S1, unofficial].

**Steps:**
1.  **Geometry Setup**: Initialize `vol_geom` and `proj_geom` using the provided dimensions and angles. Compute the total number of pixels $M = v \times h$ and $N = M$ [S1, unofficial].
2.  **Compute Right-Hand Side**: Calculate the vector $b = A^T g$ by applying the adjoint projection operator to the sinogram data $g$ [S1, unofficial].
3.  **Construct Linear Operator**: Define a `LinearOperator` `A` with shape $(M, M)$. Its `matvec` method must compute $(A^T A + \alpha R) x$ for any input vector $x$:
    - Compute $y = A x$ using the forward projection operator.
    - Compute $z = A^T y$ using the adjoint projection operator.
    - Apply regularization $R$:
        - If `penalty` is "tikhonov", $R = I$ (identity). The term is $\alpha x$.
        - If `penalty` is "gradient", $R$ is the Laplacian operator (discrete second derivative). The term is $\alpha R x$. Note that in some implementations, the gradient penalty may use a scaling factor of $2\alpha$ compared to the tikhonov penalty [S1, unofficial].
    - Return $z + \alpha R x$.
4.  **Solve**: Call the GMRES solver with the `LinearOperator` `A`, the flattened right-hand side vector $b.flatten()$, and the callback function for monitoring convergence [S1, unofficial].
5.  **Reshape**: Reshape the resulting solution vector to the 2D volume dimensions $(v, h)$ [S1, unofficial].

**Formulas:**
The solver minimizes the regularized least-squares objective:
$$ \min_f \| A f - g \|^2 + \alpha \| R f \|^2 $$
This leads to the regularized normal equations:
$$ (A^T A + \alpha R) f = A^T g $$
Where:
- $A$ is the forward projection matrix [S1, unofficial].
- $g$ is the sinogram vector [S1, unofficial].
- $R$ is the regularization operator [S1, unofficial].
- $\alpha$ is the regularization parameter [S1, unofficial].

For 0-order Tikhonov regularization:
$$ R = I $$
For 1st-order Tikhonov (gradient) regularization:
$$ R = \nabla^2 $$
where $\nabla^2$ is the discrete Laplacian operator [S1, unofficial].

**Parameters:**
- `penalty`: "tikhonov" or "gradient" [S1, unofficial].
- `alpha`: Regularization strength. The optimal value is often determined using the L-Curve method [S1, unofficial].
- `maxiter`: 700 [S1, unofficial].

**Pitfalls:**
- The regularization parameter $\alpha$ significantly affects the trade-off between data fidelity and smoothness. An inappropriate $\alpha$ can lead to over-smoothing or noisy reconstructions [S1, unofficial].
- The implementation relies on the ASTRA toolbox for projection operations. Incorrect geometry setup or projector IDs will cause errors [S1, unofficial].
- For large volumes, memory usage can be high if the `LinearOperator` is not implemented efficiently, although GMRES itself is matrix-free [S1, unofficial].
- The callback function is optional but useful for monitoring convergence [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import scipy.sparse.linalg as sp

def gmres_solver(vol_geom, proj_geom, projector_id, gNoisy_id, max_angle, n_angles, n_samples, v, h, alpha, penalty, maxiter=700):
    """
    Solves the regularized normal equations for CT reconstruction using GMRES.
    """
    # Step 1: Compute A^T g
    # Assuming astra toolbox is available and initialized
    # AT_y computes the adjoint projection of the sinogram
    ATg = astra.opfop('AT_y', proj_geom, vol_geom, projector_id, gNoisy_id)
    b = ATg.flatten()
    
    # Step 2: Define the LinearOperator for (A^T A + alpha R)
    def matvec(x):
        # x is a flattened vector of shape (v*h,)
        # Compute A * x
        Ax = astra.opfop('A_y', proj_geom, vol_geom, projector_id, x.reshape(v, h))
        # Compute A^T * (A * x)
        AT_Ax = astra.opfop('AT_y', proj_geom, vol_geom, projector_id, Ax)
        
        # Apply regularization
        if penalty == "tikhonov":
            R_x = x
        elif penalty == "gradient":
            # Compute Laplacian of x
            # Reshape to 2D for Laplacian computation
            x_2d = x.reshape(v, h)
            # Discrete Laplacian: sum of second derivatives in x and y directions
            # Using numpy for simplicity, though astra might have a built-in operator
            laplacian = np.zeros_like(x_2d)
            laplacian[1:-1, 1:-1] = (x_2d[2:, 1:-1] + x_2d[:-2, 1:-1] + 
                                     x_2d[1:-1, 2:] + x_2d[1:-1, :-2] - 
                                     4 * x_2d[1:-1, 1:-1])
            # Handle boundaries: zero-padding or replication (not specified in source)
            # For simplicity, we assume zero-padding for boundaries
            laplacian[0, :] = 0
            laplacian[-1, :] = 0
            laplacian[:, 0] = 0
            laplacian[:, -1] = 0
            R_x = laplacian.flatten()
        else:
            raise ValueError("Unknown penalty type")
            
        return AT_Ax + alpha * R_x

    A = sp.LinearOperator(shape=(v*h, v*h), matvec=matvec)
    
    # Step 3: Solve using GMRES
    # Callback function for monitoring
    def callback(rk):
        print(f"Iteration {rk}")
        
    solution, info = sp.gmres(A, b, callback=callback, maxiter=maxiter)
    
    if info != 0:
        print(f"GMRES did not converge: {info}")
        
    # Step 4: Reshape output
    f_alpha = solution.reshape(v, h)
    
    return f_alpha
```

## Subtopics

### [L-Curve Regularization Parameter Selection](8262-l-curve-regularization-parameter-selection.md)
A heuristic method for determining the optimal regularization parameter $\alpha$ by identifying the corner of the L-shaped curve plotting the norm of the solution against the norm of the residual on a log-log scale. It is used to balance data fidelity and smoothness in ill-posed inverse problems like tomographic reconstruction.

## Operating characteristics
- **Time Complexity**: GMRES is an iterative method. The cost per iteration is dominated by the matrix-vector product, which involves forward and adjoint projections. For $k$ iterations, the complexity is $O(k \cdot (N_{proj} + N_{adj}))$, where $N_{proj}$ and $N_{adj}$ are the costs of forward and adjoint projections. The total complexity is $O(k \cdot M^2)$ if the projection is dense, but typically $O(k \cdot M \log M)$ or better with sparse representations [S1, unofficial].
- **Memory Complexity**: GMRES requires storing the Krylov basis vectors, which grows with the number of iterations. However, since it is matrix-free, it does not store the system matrix $A^T A + \alpha R$. The memory usage is $O(k \cdot M)$, where $M$ is the number of pixels [S1, unofficial].
- **Performance**: GMRES is effective for large-scale problems where direct solvers are infeasible. It converges faster than simple iterative methods like Landweber or SIRT for ill-conditioned systems [S1, unofficial].
- **Strengths**: Matrix-free operation, handles large sparse systems, supports various regularization types [S1, unofficial].
- **Limitations**: Convergence can be slow for ill-conditioned problems without preconditioning. Memory usage grows with iterations [S1, unofficial].
- **When to use**: Large-scale tomographic reconstruction with sparse or limited-angle data [S1, unofficial].
- **When not to use**: Small-scale problems where direct solvers are faster, or when real-time performance is critical and convergence is slow [S1, unofficial].

## Background
The GMRES (Generalized Minimal Residual) method was developed by Yousef Saad and Martin Schultz in 1986 [S1, unofficial]. It is a popular iterative solver for non-symmetric linear systems. In the context of CT reconstruction, it is applied to the regularized normal equations to solve the inverse problem efficiently [S1, unofficial]. The method is particularly useful in compressive volume rendering recovery where the system matrix is large and sparse [S1, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
