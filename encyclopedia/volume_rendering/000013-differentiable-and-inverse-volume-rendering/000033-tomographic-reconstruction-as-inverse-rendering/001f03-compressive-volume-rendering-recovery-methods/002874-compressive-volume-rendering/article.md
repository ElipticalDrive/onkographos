# Compressive Volume Rendering

Also called: Compressive Rendering

## Overview
Compressive Volume Rendering is a technique that renders a full-resolution image by tracing rays for only a sparse subset of pixels and reconstructing the remaining pixels using an image representation model [S1, unofficial]. It extends the two-step compressive rendering proposal by Sen and Darabi [40] by replacing compressed sensing with a smoothness-based recovery approach [S1, unofficial].

## How it works
**Inputs:**
- Image dimensions $W \times H$ [S1, unofficial].
- Total pixel count $N = W \times H$ [S1, unofficial].
- Target number of rendered pixels $M$, where $M \ll N$ [S1, unofficial].
- Volume dataset and camera parameters for ray tracing [G].

**Outputs:**
- Full-resolution image $x$ of size $N \times 1$ (linearized) [S1, unofficial].
- Sparse measurements $y$ of size $M \times 1$ [S1, unofficial].

**Data Structures:**
1. **Sampling Matrix $S$**: An $M \times N$ binary matrix where each row contains exactly one `1` at the index corresponding to a pixel location to be rendered, and `0` elsewhere [S1, unofficial].
2. **Kernel Function $\phi(t)$**: A bi-variate interpolating kernel defined as the tensor product $\phi(t_1, t_2) = \beta_3^I(t_1)\beta_3^I(t_2)$, where $\beta_3^I$ is the optimal interpolating cubic B-spline [S1, unofficial].
3. **Hessian Matrix $H$**: An $N \times N$ symmetric matrix representing the smoothness penalty, where $H_{p,q} = \langle \phi(\cdot - j_p), \phi(\cdot - j_q) \rangle_{BL^2}$ [S1, unofficial].

**Steps:**

1. **Pixel Selection**: Select $M$ pixel locations from the $N$ total pixels using a low-discrepancy distribution such as linear pixel shuffling or Poisson-disk sampling [S1, unofficial]. Let $K = \{k_1, \dots, k_M\}$ be the set of selected 2D pixel coordinates [S1, unofficial].
2. **Sparse Rendering**: Trace rays only for the pixels in $K$ [G]. Collect the resulting color/intensity values into the vector $y \in \mathbb{R}^M$ [S1, unofficial]. This establishes the relationship $y = Sx$, where $x$ is the unknown full image vector [S1, unofficial].
3. **Formulation**: The reconstruction problem is formulated as minimizing the following objective function:
   $$ \min_f \lambda \|f\|_{BL^2}^2 + \sum_{m=1}^M (f(k_m) - y_m)^2 $$
   where $f(t)$ is approximated by coefficients $c$ such that $f(t) = \sum_{n=1}^N c_n \phi(t - j_n)$, $j_n$ are the 2D locations of all pixels in $x$, and $\lambda$ is a smoothing parameter [S1, unofficial].
4. **Matrix Assembly**: Convert the continuous minimization problem into a discrete linear system [S1, unofficial]. The problem becomes:
   $$ \min_x \|Sx - y\|_2^2 + \lambda x^T H x $$
   where $H$ is the convolution matrix defined by the Beppo-Levi inner product of the kernel functions [S1, unofficial]. The Beppo-Levi inner product for functions $g$ and $h$ is:
   $$ \langle g, h \rangle_{BL^2} := \langle \partial_{t_1}^2 g, \partial_{t_1}^2 h \rangle + 2\langle \partial_{t_1}\partial_{t_2} g, \partial_{t_1}\partial_{t_2} h \rangle + \langle \partial_{t_2}^2 g, \partial_{t_2}^2 h \rangle $$
   where $\langle \cdot, \cdot \rangle$ is the standard $L^2$ inner product $\int_{\mathbb{R}^2} f(x)g(x)dx$ [S1, unofficial].
5. **Solution**: Solve the linear system for $x$ [S1, unofficial]. The gradient of the objective function with respect to $x$ is:
   $$ \nabla E(x) = 2S^T(Sx - y) + 2\lambda H x $$
   Setting the gradient to zero yields the normal equation:
   $$ (S^T S + \lambda H) x = S^T y $$
   Let $A = S^T S + \lambda H$ and $b = S^T y$ [S1, unofficial]. Solve $Ax = b$ using an iterative solver such as Conjugate Gradient [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compressive_volume_rendering(W, H, M, volume_data, lambda_smooth):
    """
    Renders a full image by sampling M pixels and reconstructing the rest.
    """
    N = W * H
    
    # Step 1: Select M pixel indices using a low-discrepancy distribution [S1, unofficial]
    # Implementation uses a deterministic low-discrepancy sampler (e.g., linear pixel shuffling)
    sampled_indices = sorted([i * N // M for i in range(M)])
    sampled_indices_set = set(sampled_indices)
    
    # Step 2: Render only the sampled pixels [G]
    y = []
    for idx in sampled_indices:
        u = idx % W
        v = idx // W
        # Standard volume rendering integration along the ray at (u, v). Not given in the sources.
        color = trace_ray(u, v, volume_data)
        y.append(float(color))
        
    # Step 3 & 4: Construct A and b implicitly [S1, unofficial]
    # A = S^T S + lambda * H
    # b = S^T y
    # H is a convolution matrix based on the Beppo-Levi inner product of the kernel.
    # We define a matvec function for H to avoid explicit O(N^2) storage [S1, unofficial].
    
    diag_S = [1.0 if i in sampled_indices_set else 0.0 for i in range(N)]
    
    def matvec_H(x):
        # H corresponds to convolution with the BL2 kernel of the cubic B-spline [S1, unofficial].
        # Hx is computed via separable convolution with zero-padding boundary handling [G].
        h_result = convolve_2d(x, kernel_1d_bl2, W, H)
        return [lambda_smooth * v for v in h_result]

    def matvec_A(x):
        # S^T S is diagonal: 1 at sampled indices, 0 elsewhere.
        h_part = matvec_H(x)
        return [diag_S[i] * x[i] + h_part[i] for i in range(N)]

    # Construct b
    b = [0.0] * N
    for m, idx in enumerate(sampled_indices):
        b[idx] = y[m]
        
    # Step 5: Solve Ax = b using Conjugate Gradient [S1, unofficial]
    x = conjugate_gradient_solver(matvec_A, b, N)
    return x

# Kernel derived from the BL2 inner product of beta_3^I [S1, unofficial]
kernel_1d_bl2 = [1.0/6.0, -1.0/6.0, 0.0, 1.0, 0.0, -1.0/6.0, 1.0/6.0]

def convolve_2d(x, kernel_1d, W, H):
    # Separable convolution with zero-padding [G]
    temp = [0.0] * (W * H)
    for i in range(H):
        for j in range(W):
            val = 0.0
            for k in range(len(kernel_1d)):
                u = j - (len(kernel_1d)//2) + k
                if 0 <= u < W:
                    val += x[i * W + u] * kernel_1d[k]
            temp[i * W + j] = val
    result = [0.0] * (W * H)
    for j in range(W):
        for i in range(H):
            val = 0.0
            for k in range(len(kernel_1d)):
                v = i - (len(kernel_1d)//2) + k
                if 0 <= v < H:
                    val += temp[v * W + j] * kernel_1d[k]
            result[i * W + j] = val
    return result

def trace_ray(u, v, volume_data):
    # Standard volume rendering integration along the ray at (u, v). Not given in the sources.
    return 0.0

def conjugate_gradient_solver(matvec_A, b, N, max_iter=1000, tol=1e-6):
    # Matches Algorithm 1 from the source [S1, unofficial]
    x = [0.0] * N
    r = [b[i] - matvec_A(x)[i] for i in range(N)]
    p = r[:]
    r_dot_r = sum(r[i] * r[i] for i in range(N))
    
    for k in range(max_iter):
        Ap = matvec_A(p)
        pAp = sum(p[i] * Ap[i] for i in range(N))
        if pAp == 0:
            break
        alpha = r_dot_r / pAp
        for i in range(N):
            x[i] += alpha * p[i]
        r_new = [r[i] - alpha * Ap[i] for i in range(N)]
        r_new_dot_r_new = sum(r_new[i] * r_new[i] for i in range(N))
        if r_new_dot_r_new < tol:
            break
        beta = r_new_dot_r_new / r_dot_r
        for i in range(N):
            p[i] = r_new[i] + beta * p[i]
        r = r_new
        r_dot_r = r_new_dot_r_new
    return x
```

**Parameters:**
- $\lambda$: Smoothing parameter that controls the trade-off between fidelity to the rendered pixels and the smoothness of the reconstructed image [S1, unofficial].
- Kernel: Optimal interpolating cubic B-spline $\beta_3^I(t) = \beta_3(t) - \frac{1}{6}\partial^2_t \beta_3(t)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: The matrix $H$ can be ill-conditioned if $\lambda$ is too small or too large [G]. A typical range for $\lambda$ is determined empirically based on the noise level of the volume rendering [G].
- **Boundaries**: The convolution matrix $H$ requires boundary handling [S1, unofficial]. Zero-padding is a common default for such convolutions [G].
- **Degenerate Input**: If $M=0$, the problem is unsolvable [G]. If $M=N$, the problem reduces to standard rendering with no reconstruction needed [G].

## Operating characteristics
- **Time Complexity**: The rendering phase is $O(M \cdot R)$, where $R$ is the average ray length [G]. The reconstruction phase involves solving a linear system of size $N \times N$ [G]. Using Conjugate Gradient, each iteration is $O(N)$ if $H$ is applied via convolution since $H$ is sparse/banded [G]. The number of iterations depends on the condition number of $A$ [G].
- **Memory Complexity**: Storing $H$ explicitly requires $O(N \cdot \text{support}^2)$ space [G]. If $H$ is applied via convolution, only the kernel needs to be stored, reducing memory to $O(1)$ relative to $N$ excluding the image buffer [G]. The sampling matrix $S$ is stored implicitly [G].
- **Performance**: The method reduces rendering time by a factor of $N/M$ [G]. However, the reconstruction cost must be considered [S1, unofficial]. For large $N$, the conjugate gradient solver can be expensive [S1, unofficial]. The solver is implemented on CUDA for acceleration [S1, unofficial].
- **Strengths**: Significant reduction in ray-casting overhead [G]. High-quality recovery if the image is smooth [S1, unofficial].
- **Limitations**: Reconstruction artifacts if the image is not smooth or if $M$ is too small [G]. The method assumes the image can be well-approximated by the chosen kernel basis [S1, unofficial].
- **When to use**: When ray casting is the bottleneck and the volume rendering result is expected to be spatially smooth [G].
- **When not to use**: When high-frequency details are critical and cannot be interpolated, or when $M$ cannot be reduced significantly [G].

## Background
Compressive Volume Rendering was initially proposed by Sen and Darabi [40] as cited in [S1, unofficial]. The method extends the concept of compressive rendering to volume data [S1, unofficial]. The thesis [S1, unofficial] modifies the original proposal by using linear pixel shuffling instead of Poisson-disk distribution and a smoothness-based approach by Liu and Alim [4] instead of general compressed sensing, showing better performance [S1, unofficial]. The kernel choice is based on work by Blu et al. [48] [S1, unofficial].

## Sources
[S1] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
