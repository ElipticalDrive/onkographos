# Compressive Volume Rendering

Also called: Compressive Rendering

## Overview
Compressive Volume Rendering is an image-order acceleration technique that reconstructs a full rendered image from a small subset of rendered pixels, exploiting the inherent smoothness of volume rendered images to reduce computational cost [S2]. It solves the problem of high rendering costs by rendering only a fraction of the pixels and recovering the rest via a smoothness-based variational model, which sources indicate performs better than compressed sensing approaches for this domain [S1, unofficial]. It builds on [Compressive Volume Rendering Recovery Methods](1915-compressive-volume-rendering-recovery-methods.md) [S1, unofficial].

## How it works
**Inputs:**
- Image dimensions $W, H$.
- Total pixel count $N = W \times H$ [S1, unofficial][S2].
- Subset size $M$ (typically $M \ll N$, e.g., 20% of pixels) [S2].
- Rendered pixel values $y$ (column vector of size $M \times 1$) [S1, unofficial][S2].
- Sampling matrix $S$ (size $M \times N$) [S1, unofficial][S2].
- Regularization parameter $\lambda$ [S1, unofficial].

**Outputs:**
- Full rendered image $x$ (column vector of size $N \times 1$) [S1, unofficial][S2].

**Data Structures:**
- **Sampling Matrix $S$**: An $M \times N$ binary matrix. Each row contains exactly one `1` at the column index corresponding to a retained pixel location, and `0` elsewhere [S2].
- **Kernel Function $\phi(t)$**: The optimal interpolating cubic B-spline $\beta_3^I$, defined as $\beta_3^I(t) = \beta_3(t) - \frac{1}{6}\frac{\partial^2}{\partial t^2}\beta_3(t)$, where $\beta_3(t)$ is the univariate uniform centered cubic B-spline [S1, unofficial]. The 2D kernel is the tensor product $\phi(t_1, t_2) = \beta_3^I(t_1)\beta_3^I(t_2)$ [S1, unofficial].
- **Matrix $H$**: An $N \times N$ convolution matrix where entries are defined by the second-order Beppo-Levi inner product of the kernel functions: $H_{p,q} = \langle \phi(\cdot - j_p), \phi(\cdot - j_q) \rangle_{BL^2}$ [S1, unofficial]. Here $j_p$ and $j_q$ are the 2D pixel locations in the full image $x$ [S1, unofficial].

**Steps:**
1. **Pixel Sampling**: Select $M$ pixel locations to render. Sources note two approaches: low-discrepancy distributions such as Poisson-disk [S1, unofficial], or a linear pixel shuffling algorithm [S1, unofficial].
2. **Rendering**: Trace rays for the selected $M$ pixels to obtain the vector $y$ [S1, unofficial].
3. **Formulate Optimization Problem**: The goal is to find $x$ that minimizes $\min_x \|Sx - y\|_2^2 + \lambda x^T H x$, where the first term is the fidelity to rendered pixels and the second term measures smoothness via the second-order Beppo-Levi seminorm [S1, unofficial].
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
    """
    N = W * H
    
    # Step 1: Select M pixel locations. Sources mention Poisson-disk or linear pixel shuffling [S1, unofficial].
    sampled_indices = [i * N // M for i in range(M)]  # [G] Uniform sampling; replace with Poisson-disk or linear shuffling per source
    sampled_coords = [(idx % W, idx // W) for idx in sampled_indices]
    
    # Step 2: Render the subset
    y = render_function(sampled_coords)  # y is list of size M
    
    # Step 3: Construct b = S^T y
    b = [0.0] * N
    for m, idx in enumerate(sampled_indices):
        b[idx] = y[m]
        
    # Step 4: Define linear operator A = S^T S + lambda * H
    # S^T S is a diagonal matrix with 1s at sampled_indices.
    # H is a convolution matrix derived from the Beppo-Levi inner product.
    # Exact discrete coefficients for beta_3^I are not given in the sources [S1, unofficial].
    kernel_2d = [[0.0] * 5 for _ in range(5)]  # [G] Exact 5x5 kernel coefficients from Blu et al. [48] not provided in sources
    
    def apply_A(v):
        # (S^T S) v: zero out non-sampled indices
        res = list(v)
        sampled_set = set(sampled_indices)
        for idx in range(N):
            if idx not in sampled_set:
                res[idx] = 0.0
        # lambda * H v: 2D convolution with boundary handling [S1, unofficial]
        conv_res = convolve_2d(res, kernel_2d, W, H)
        for i in range(N):
            res[i] += lambda_param * conv_res[i]
        return res

    # Step 5: Solve Ax = b using Conjugate Gradient (Algorithm 1 from [S1, unofficial])
    x = [0.0] * N
    r = [b[i] - apply_A(x)[i] for i in range(N)]
    p = list(r)
    r_dot_r = sum(r[i] * r[i] for i in range(N))
    
    for k in range(1000):
        Ap = apply_A(p)
        p_Ap = sum(p[i] * Ap[i] for i in range(N))
        if p_Ap == 0.0:
            break
        alpha = r_dot_r / p_Ap
        
        # x_{k+1} = x_k + alpha_k * p_k
        for i in range(N):
            x[i] += alpha * p[i]
            
        # r_{k+1} = r_k - alpha_k * A * p_k
        for i in range(N):
            r[i] -= alpha * Ap[i]
            
        r_dot_r_new = sum(r[i] * r[i] for i in range(N))
        if r_dot_r_new < 1e-12:
            break
            
        beta = r_dot_r_new / r_dot_r
        r_dot_r = r_dot_r_new
        
        # p_{k+1} = r_{k+1} + beta_k * p_k
        for i in range(N):
            p[i] = r[i] + beta * p[i]
            
    return x

def convolve_2d(image_flat, kernel, W, H):
    """Applies 2D convolution with zero-padding boundary handling."""
    out = [0.0] * (W * H)
    kh, kw = len(kernel), len(kernel[0])
    for y in range(H):
        for x in range(W):
            acc = 0.0
            for ky in range(kh):
                for kx in range(kw):
                    iy, ix = y + ky - kh // 2, x + kx - kw // 2
                    if 0 <= iy < H and 0 <= ix < W:
                        acc += image_flat[iy * W + ix] * kernel[ky][kx]
            out[y * W + x] = acc
    return out
```

**Edge Cases and Pitfalls:**
- **Ill-posedness**: The problem is inherently ill-posed because $M \ll N$ [S2].
- **Kernel Size**: Larger kernels increase accuracy but also computational cost [S1, unofficial].
- **Boundary Handling**: Automatic boundary handling is required for the convolution operations near image edges [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources. The solver is described as "computationally expensive" [S1, unofficial].
- **Memory Complexity**: Not given in the sources. Storing the image requires $O(N)$ [S2].
- **Performance**: Can achieve high-quality recovery with as few as 20% of the pixels rendered [S2].
- **Strengths**: Significantly reduces rendering cost by tracing fewer rays. Exploits image smoothness effectively [S2].
- **Limitations**: The recovery step is computationally intensive. The method assumes the image is smooth; high-frequency details may be lost or blurred [S1, unofficial].
- **When to use**: When rendering cost is prohibitive and the volume rendered image exhibits smooth intensity variations [G].
- **When not to use**: When high-frequency details are critical or the image is not smooth [G].

## Background
Compressive Volume Rendering was initially proposed by Sen and Darabi [40] (cited as [SD11] in [S2]), who presented a two-step algorithm using compressed sensing [S1, unofficial]. Later, Liu and Alim [4] (cited as [S2]) demonstrated that smoothness-based approaches perform better than compressed sensing for this task [S1, unofficial]. This work extends the initial proposal by using linear pixel shuffling instead of Poisson-disk patterns and a smoothness-based variational reconstruction [S1, unofficial]. The method is part of the broader field of compressive rendering, which aims to reconstruct full images from sparse samples [S2].

## Sources
[S1] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S2] Compressive Volume Rendering, https://doi.org/10.1111/cgf.12622, fetched 2026-10-26
