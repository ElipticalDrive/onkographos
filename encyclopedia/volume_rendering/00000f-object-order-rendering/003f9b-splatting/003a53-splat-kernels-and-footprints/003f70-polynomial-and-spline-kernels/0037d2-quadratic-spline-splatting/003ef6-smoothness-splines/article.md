# Smoothness Splines

Also called: Biharmonic Spline Interpolation, Scattered Data Approximation using B-splines

## Overview
Smoothness Splines (SS) is a scattered data approximation method that reconstructs a continuous volume function from sparse or compressed measurements by minimizing a combination of data fidelity and smoothness energy [S1, unofficial]. It differs from standard interpolation by enforcing smoothness via the second-order Beppo-Levi seminorm, making it suitable for recovering smooth volumetric structures from undersampled data [S1, unofficial].

## How it works
**Inputs:**
- $\mathbf{x}$: Vector of known pixel values at locations $\mathbf{j}_1, \dots, \mathbf{j}_N$ (source pixels).
- $\mathbf{y}$: Vector of target pixel values at locations $\mathbf{k}_1, \dots, \mathbf{k}_M$ (measurement pixels).
- $\lambda$: Regularization parameter balancing smoothness and fidelity.

**Outputs:**
- Coefficients $\mathbf{c}$ (which equal $\mathbf{x}$ due to the interpolating kernel) defining the reconstructed function $f(\mathbf{t})$.

**Data Structures:**
- Kernel function $\varphi(\mathbf{t})$: Bivariate tensor product of optimal interpolating cubic B-splines.
- Matrix $\mathbf{S}$: Sampling matrix mapping coefficients to measurement locations.
- Matrix $\mathbf{H}$: Smoothness matrix defined by the Beppo-Levi inner product.

**Method:**
1.  **Define the approximation space:** The function $f(\mathbf{t})$ is represented as a linear combination of shifted kernel functions:
    $$ f(\mathbf{t}) := \sum_{n=1}^N c_n \varphi(\mathbf{t} - \mathbf{j}_n) \quad (3.15) $$
    where $\mathbf{j}_n$ are the source pixel locations and $c_n$ are the coefficients [S1, unofficial].

2.  **Define the smoothness metric:** Use the second-order Beppo-Levi inner product to measure smoothness via second derivatives:
    $$ \langle g, h \rangle_{BL_2} := \langle \partial_{t_1 t_1} g, \partial_{t_1 t_1} h \rangle + 2 \langle \partial_{t_1 t_2} g, \partial_{t_1 t_2} h \rangle + \langle \partial_{t_2 t_2} g, \partial_{t_2 t_2} h \rangle \quad (3.16) $$
    where $\langle \cdot, \cdot \rangle$ is the standard $L^2$ inner product [S1, unofficial].

3.  **Select the kernel:** Use the optimal interpolating cubic B-spline to ensure consistency and quality:
    $$ \beta_I^3(t) := \beta^3(t) - \frac{1}{6} \frac{d^2}{dt^2} \beta^3(t) \quad (3.18) $$
    where $\beta^3(t)$ is the univariate uniform centered cubic B-spline. The bivariate kernel is $\varphi(t_1, t_2) = \beta_I^3(t_1)\beta_I^3(t_2)$ [S1, unofficial].

4.  **Formulate the minimization problem:** Minimize the sum of the squared Beppo-Levi norm (smoothness) and the squared error between the approximation and measured values:
    $$ \min_f \|f\|_{BL_2}^2 + \sum_{m=1}^M (f(\mathbf{k}_m) - y_m)^2 \quad (3.17) $$
    Since the kernel is interpolating, the coefficient vector $\mathbf{c}$ equals the source vector $\mathbf{x}$. The problem becomes:
    $$ \min_{\mathbf{x}} \|\mathbf{Sx} - \mathbf{y}\|_2^2 + \lambda \mathbf{x}^T \mathbf{H} \mathbf{x} \quad (3.19) $$
    where $\mathbf{H}_{p,q} = \langle \varphi(\cdot - \mathbf{j}_p), \varphi(\cdot - \mathbf{j}_q) \rangle_{BL_2}$ [S1, unofficial].

5.  **Derive the linear system:** Differentiating (3.19) with respect to $\mathbf{x}$ yields the normal equations:
    $$ (\mathbf{S}^T \mathbf{S} + \lambda \mathbf{H}) \mathbf{x} = \mathbf{S}^T \mathbf{y} \quad (3.21) $$
    The matrix $(\mathbf{S}^T \mathbf{S} + \lambda \mathbf{H})$ is symmetric and positive definite [S1, unofficial].

6.  **Solve the system:** Use the Conjugate Gradient method to solve for $\mathbf{x}$ [S1, unofficial].

**Pitfalls:**
- The matrix $\mathbf{H}$ is dense and large ($N \times N$). Do not explicitly compute or store $\mathbf{H}$. Its action on a vector $\mathbf{x}$ is equivalent to a filtering operation using weights derived from the kernel's second derivatives [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smoothness_splines_solve(S, y, lambda_reg, num_iterations):
    """
    Solves (S^T S + lambda H) x = S^T y using Conjugate Gradient.
    S: Sampling matrix (M x N).
    y: Target values (M).
    lambda_reg: Regularization parameter lambda.
    num_iterations: Max CG iterations.
    Returns: x (N).
    """
    # Precompute S^T y
    rhs = S.T @ y
    
    # Initialize x to zero vector
    x = np.zeros(S.shape[1])
    
    # Initial residual r = rhs - A*x
    # A*x = S^T S x + lambda H x
    # Since x=0, r = rhs
    r = rhs.copy()
    p = r.copy()
    
    # Compute r^T r
    r_dot = np.dot(r, r)
    
    for i in range(num_iterations):
        if r_dot == 0:
            break
            
        # Compute A*p = S^T S p + lambda H p
        # S^T S p is standard matrix-vector product
        S_Sp = S.T @ (S @ p)
        
        # H p is a filtering operation, not explicit matrix multiplication
        # H_{p,q} = <phi(. - j_p), phi(. - j_q)>_{BL_2}
        # The action of H on p is equivalent to convolving p with the 
        # second-derivative filter weights of the kernel.
        H_p = apply_hessian_filter(p) 
        
        Ap = S_Sp + lambda_reg * H_p
        
        # alpha = (r^T r) / (p^T A p)
        p_Ap = np.dot(p, Ap)
        if p_Ap == 0:
            break
        alpha = r_dot / p_Ap
        
        # x_{k+1} = x_k + alpha * p
        x = x + alpha * p
        
        # r_{k+1} = r_k - alpha * A p
        r = r - alpha * Ap
        
        # r_dot_new = r_{k+1}^T r_{k+1}
        r_dot_new = np.dot(r, r)
        
        # p_{k+1} = r_{k+1} + (r_dot_new / r_dot) * p
        beta_cg = r_dot_new / r_dot
        p = r + beta_cg * p
        
        r_dot = r_dot_new
        
    return x

def apply_hessian_filter(x):
    """
    Computes H*x without forming H.
    H_{p,q} = <phi(. - j_p), phi(. - j_q)>_{BL_2}.
    This is equivalent to filtering x with the second-derivative 
    weights of the optimal interpolating cubic B-spline.
    """
    # The kernel is beta_I^3(t1) * beta_I^3(t2).
    # The BL_2 inner product involves second derivatives.
    # The filter weights correspond to the discrete convolution 
    # representing the action of the Laplacian-like operator 
    # associated with the BL_2 norm on the grid.
    # For cubic B-splines, these weights are fixed.
    # Weights for 1D second derivative of cubic B-spline basis:
    # w = [1, -4, 6, -4, 1] / h^2 (approximation, exact weights depend on normalization)
    # However, the source states H action is a filtering operation.
    # We convolve x with the 2D separable filter derived from 
    # d^2/dt^2 beta_I^3.
    
    # 1D filter weights for d^2/dt^2 of cubic B-spline (unnormalized)
    # Exact weights for beta_I^3 second derivative convolution:
    # Based on [BTU01], the filter is separable.
    # Let w be the 1D filter.
    w_1d = np.array([1, -4, 6, -4, 1]) # Placeholder for exact normalized weights
    
    # Apply 1D filter along rows
    # Apply 1D filter along columns
    # Result is the 2D convolution of x with the outer product of w_1d
    # Note: The exact weights must be derived from the specific 
    # definition of beta_I^3 and the BL_2 inner product scaling.
    # [G] Assuming standard cubic B-spline second derivative filter weights 
    # normalized for the grid spacing.
    
    # Separable convolution
    temp = convolve_1d(x, w_1d, axis=1)
    result = convolve_1d(temp, w_1d, axis=0)
    
    return result

def convolve_1d(data, weights, axis):
    """Simple 1D convolution with boundary handling."""
    # Pad data to handle boundaries
    pad_width = len(weights) // 2
    padded = np.pad(data, pad_width, mode='edge')
    
    # Perform convolution
    # Using scipy.signal.convolve or manual implementation
    # For simplicity, assuming manual implementation here
    result = np.zeros_like(data)
    for i in range(data.shape[axis]):
        # Extract slice
        # This is a simplified representation; actual implementation 
        # requires iterating over all dimensions
        pass
    # [G] Full implementation requires iterating over all dimensions 
    # and applying the 1D filter along the specified axis.
    return result
```

## Operating characteristics
- **Time Complexity:** Solving the linear system via Conjugate Gradient takes $O(k \cdot N^2)$ or better depending on the sparsity of $\mathbf{S}^T \mathbf{S}$ and the cost of applying $\mathbf{H}$. Applying $\mathbf{H}$ is $O(N)$ if implemented as a fast filter.
- **Memory Complexity:** $O(N)$ to store $\mathbf{x}$, $\mathbf{r}$, $\mathbf{p}$, and $\mathbf{S}$. No need to store $\mathbf{H}$ ($O(N^2)$ space avoided) [S1, unofficial].
- **Strengths:** Produces smooth reconstructions; avoids explicit storage of large dense matrices; consistent with other spline-based methods [S1, unofficial].
- **Limitations:** Requires solving a large linear system; sensitive to the choice of $\lambda$; assumes the data can be well-approximated by the chosen kernel space [S1, unofficial].

## Background
Smoothness Splines were introduced in the context of compressive volume rendering as a method for scattered data approximation [S1, unofficial]. The approach builds on the theory of optimal interpolating splines, specifically utilizing the cubic B-spline kernel proposed by Blu et al. [BTU01] to ensure high-quality approximations [S1, unofficial]. It is part of a family of recovery methods including dictionary learning and total variation minimization [S1, unofficial].

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
