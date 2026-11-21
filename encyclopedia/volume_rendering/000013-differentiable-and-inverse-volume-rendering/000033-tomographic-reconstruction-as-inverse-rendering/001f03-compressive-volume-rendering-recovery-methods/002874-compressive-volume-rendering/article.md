# Compressive Volume Rendering

Also called: Compressive Rendering

## Overview
Compressive Volume Rendering is an image-order acceleration technique that reconstructs a full rendered image from a small subset of rendered pixels, exploiting the inherent smoothness of volume rendered images to reduce computational cost [S2]. It solves the problem of high rendering costs by rendering only a fraction of the pixels and recovering the rest via a smoothness-based variational model, which sources indicate performs better than compressed sensing approaches for this domain [S1, unofficial]. It builds on [Compressive Volume Rendering Recovery Methods](1915-compressive-volume-rendering-recovery-methods.md).

## How it works
**Inputs:**
- Image dimensions $W, H$.
- Total pixel count $N = W \times H$.
- Subset size $M$ (typically $M \ll N$, e.g., 20% of pixels) [S2].
- Rendered pixel values $y$ (column vector of size $M \times 1$) [S1, unofficial].
- Sampling matrix $S$ (size $M \times N$) [S1, unofficial].
- Regularization parameter $\lambda$ [S1, unofficial].

**Outputs:**
- Full rendered image $x$ (column vector of size $N \times 1$) [S1, unofficial].

**Data Structures:**
- **Sampling Matrix $S$**: An $M \times N$ binary matrix. Each row contains exactly one `1` at the column index corresponding to a retained pixel location, and `0` elsewhere [S1, unofficial][S2].
- **Kernel Function $\phi(t)$**: The optimal interpolating cubic B-spline $\beta_3^I$, defined as $\beta_3^I(t) = \beta_3(t) - \frac{1}{6}\frac{\partial^2}{\partial t^2}\beta_3(t)$, where $\beta_3(t)$ is the univariate uniform centered cubic B-spline [S1, unofficial]. The 2D kernel is the tensor product $\phi(t_1, t_2) = \beta_3^I(t_1)\beta_3^I(t_2)$ [S1, unofficial].
- **Matrix $H$**: An $N \times N$ convolution matrix where entries are defined by the second-order Beppo-Levi inner product of the kernel functions: $H_{p,q} = \langle \phi(\cdot - j_p), \phi(\cdot - j_q) \rangle_{BL^2}$ [S1, unofficial]. Here $j_p$ and $j_q$ are the 2D pixel locations in the full image $x$ [S1, unofficial].

**Steps:**
1. **Pixel Sampling**: Select $M$ pixel locations to render. The sources note two approaches:
   - Low-discrepancy distribution such as Poisson-disk [S1, unofficial].
   - Linear pixel shuffling algorithm (proposed by Anderson et al.) [S1, unofficial].
2. **Rendering**: Trace rays for the selected $M$ pixels to obtain the vector $y$ [S1, unofficial].
3. **Formulate Optimization Problem**: The goal is to find $x$ that minimizes:
   $$ \min_x \|Sx - y\|_2^2 + \lambda x^T H x $$
   where the first term is the fidelity to rendered pixels and the second term measures smoothness via the second-order Beppo-Levi seminorm [S1, unofficial].
4. **Solve**: Use a Conjugate Gradient solver to find $x$ [S1, unofficial].

**Formulas:**
- **Sampling Model**: $y = Sx$ [S1, unofficial][S2].
- **Beppo-Levi Inner Product**: For functions $g(t)$ and $h(t)$,
  $$ \langle g, h \rangle_{BL^2} := \langle \partial_{t_1}^2 g, \partial_{t_1}^2 h \rangle + 2\langle \partial_{t_1}\partial_{t_2} g, \partial_{t_1}\partial_{t_2} h \rangle + \langle \partial_{t_2}^2 g, \partial_{t_2}^2 h \rangle $$
  where $\langle \cdot, \cdot \rangle$ is the standard $L^2$ inner product $\int_{\mathbb{R}^2} f(x)g(x)dx$ [S1, unofficial].
- **Kernel Definition**: $\beta_3^I(t) = \beta_3(t) - \frac{1}{6}\frac{\partial^2}{\partial t^2}\beta_3(t)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compressive_volume_rendering(W, H, M, render_function, lambda_param):
    """
    Reconstructs a full image from a subset of rendered pixels.
    
    Args:
        W: Image width
        H: Image height
        M: Number of pixels to render
        render_function: Function that traces rays for a list of (x, y) coordinates
        lambda_param: Regularization parameter for smoothness
        
    Returns:
        x: Reconstructed image as a flat vector of size N
    """
    N = W * H
    
    # Step 1: Select M pixel locations
    # Note: Sources mention Poisson-disk or linear pixel shuffling.
    # Implementation detail: Generate M unique indices in [0, N-1].
    sampled_indices = select_sampling_pattern(M, N) # [G] Placeholder for specific pattern logic
    
    # Step 2: Render the subset
    # Convert flat indices to 2D coordinates
    sampled_coords = indices_to_2d(sampled_indices, W)
    y = render_function(sampled_coords) # y is vector of size M
    
    # Step 3: Construct Sampling Matrix S implicitly
    # S is M x N binary matrix. Row m has 1 at column sampled_indices[m].
    # We do not construct S explicitly if using sparse operations, 
    # but for the CG solver formulation Ax=b, we need A = S^T S + lambda H.
    
    # Step 4: Construct Matrix H
    # H is N x N. H_{p,q} = <phi(. - j_p), phi(. - j_q)>_{BL^2}
    # This is a convolution matrix.
    H = build_convolution_matrix(N, W, H_kernel=build_kernel_matrix()) # [G] Implementation of kernel convolution
    
    # Step 5: Formulate Linear System for Conjugate Gradient
    # The gradient of the objective function f(x) = ||Sx - y||^2 + lambda x^T H x is:
    # grad f(x) = 2 S^T (Sx - y) + 2 lambda H x
    # Setting grad f(x) = 0 gives: (S^T S + lambda H) x = S^T y
    
    # Compute b = S^T y
    # Since S is sparse with one 1 per row, S^T y gathers y values to their original positions.
    b = gather_pixels(y, sampled_indices, N) # [G] Creates N-vector with y at sampled indices, 0 elsewhere
    
    # Compute A = S^T S + lambda H
    # S^T S is a diagonal matrix with 1s at sampled_indices.
    # Let D be the diagonal matrix of sampled indices.
    # A = D + lambda H
    A = build_sparse_operator(D=sampled_indices, H=H, N=N, lambda_param=lambda_param) # [G] Constructs sparse matrix A
    
    # Step 6: Solve Ax = b using Conjugate Gradient
    x = conjugate_gradient_solve(A, b, initial_guess=None)
    
    return x

def build_kernel_matrix():
    """
    Constructs the kernel matrix for the convolution H.
    The kernel is the 2D tensor product of beta_3^I.
    """
    # beta_3(t) is the cubic B-spline.
    # beta_3^I(t) = beta_3(t) - 1/6 * d^2/dt^2 beta_3(t)
    # Implementation of the 1D kernel coefficients would go here.
    # [G] Standard cubic B-spline coefficients are [1, 4, 1]/6.
    # The second derivative of the B-spline is related to the second difference.
    # The optimal interpolating cubic B-spline kernel coefficients are typically derived analytically.
    # For implementation, one would compute the 1D kernel vector k_1d.
    k_1d = compute_optimal_interpolating_cubic_b_spline_kernel() # [G] Returns 1D kernel coefficients
    # 2D kernel is tensor product
    k_2d = outer_product(k_1d, k_1d)
    return k_2d

def compute_optimal_interpolating_cubic_b_spline_kernel():
    """
    Computes the 1D kernel beta_3^I.
    beta_3(t) is supported on [-2, 2].
    beta_3^I(t) = beta_3(t) - 1/6 * beta_3''(t).
    """
    # [G] The cubic B-spline beta_3(t) is defined piecewise.
    # Its second derivative beta_3''(t) is also piecewise linear.
    # The resulting kernel beta_3^I is also compactly supported.
    # Standard coefficients for the optimal interpolating cubic B-spline 
    # in the context of this specific formulation (Blu et al.) are required.
    # Without the exact coefficients from [48], this is a placeholder.
    return [0.0] # [G] Placeholder for exact kernel coefficients from Blu et al. [48]

def conjugate_gradient_solve(A, b, initial_guess=None, max_iter=1000, tol=1e-6):
    """
    Solves Ax = b using the Conjugate Gradient method.
    Based on Algorithm 1 from [S1, unofficial].
    """
    if initial_guess is None:
        x = [0.0] * len(b)
    else:
        x = list(initial_guess)
        
    # r0 = b - A*x0
    r = vector_sub(b, mat_vec_mul(A, x))
    p = list(r)
    
    for k in range(max_iter):
        # alpha_k = (r_k^T r_k) / (p_k^T A p_k)
        r_tr = dot_product(r, r)
        Ap = mat_vec_mul(A, p)
        p_Ap = dot_product(p, Ap)
        
        if p_Ap == 0:
            break # Singular matrix
            
        alpha = r_tr / p_Ap
        
        # x_{k+1} = x_k + alpha_k * p_k
        x = vector_add(x, vector_scale(p, alpha))
        
        # r_{k+1} = r_k - alpha_k * A * p_k
        r = vector_sub(r, vector_scale(Ap, alpha))
        
        # Check convergence
        if norm(r) < tol:
            break
            
        # beta_k = (r_{k+1}^T r_{k+1}) / (r_k^T r_k)
        r_new_tr = dot_product(r, r)
        beta = r_new_tr / r_tr
        
        # p_{k+1} = r_{k+1} + beta_k * p_k
        p = vector_add(r, vector_scale(p, beta))
        
    return x
```

**Edge Cases and Pitfalls:**
- **Ill-posedness**: The problem is inherently ill-posed because $M \ll N$. The smoothness prior ($\lambda$) is critical for stability [S2].
- **Kernel Size**: Larger kernels increase accuracy but also computational cost. The optimal interpolating cubic B-spline is chosen to balance this [S1, unofficial].
- **Boundary Handling**: Automatic boundary handling is required for the convolution operations near image edges [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by the Conjugate Gradient solver iterations. Each iteration involves matrix-vector multiplications with $S^T S$ (sparse, $O(M)$) and $H$ (convolution, $O(N \log N)$ with FFT or $O(N)$ with compact kernel if optimized) [S1, unofficial]. The solver is described as "computationally expensive" [S1, unofficial].
- **Memory Complexity**: $O(N)$ for storing the image and $O(N)$ for the sparse matrix $H$ (if stored explicitly) or $O(1)$ if $H$ is applied via convolution. The sampling matrix $S$ is implicit.
- **Performance**: Can achieve high-quality recovery with as few as 20% of the pixels rendered [S2].
- **Strengths**: Significantly reduces rendering cost by tracing fewer rays. Exploits image smoothness effectively [S2].
- **Limitations**: The recovery step is computationally intensive. The method assumes the image is smooth; high-frequency details may be lost or blurred [S1, unofficial].
- **When to use**: When rendering cost is prohibitive and the volume rendered image exhibits smooth intensity variations.
- **When not to use**: When high-frequency details are critical or the image is not smooth.

## Background
Compressive Volume Rendering was initially proposed by Sen and Darabi [40] (cited as [SD11] in [S2]), who presented a two-step algorithm using compressed sensing. Later, Liu and Alim [4] (cited as [S2]) demonstrated that smoothness-based approaches perform better than compressed sensing for this task. This work extends the initial proposal by using linear pixel shuffling instead of Poisson-disk patterns and a smoothness-based variational reconstruction [S1, unofficial]. The method is part of the broader field of compressive rendering, which aims to reconstruct full images from sparse samples [S2].

## Sources
[S1] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S2] Compressive Volume Rendering, https://doi.org/10.1111/cgf.12622, fetched 2026-10-26
