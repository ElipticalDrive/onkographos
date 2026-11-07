# Compressive Gradient Rendering

Also called: CS-Gradient

## Overview
Compressive Gradient Rendering is a compressive sensing technique that renders discrete gradient components instead of pixel values to exploit sparsity in the gradient domain, then reconstructs the image using a Poisson solver [S1, unofficial]. It differs from standard volume rendering by optimizing for gradient coefficients in the frequency domain rather than direct pixel intensities, allowing for reconstruction from fewer samples than the Nyquist rate [S1, unofficial].

## How it works
**Inputs:**
- A volume scene to render.
- A sampling matrix $\mathbf{S}$ defining $m$ random pixel locations.
- A pixel reconstruction kernel (e.g., bilinear B-spline or Mitchell-Netravali cubic kernel) [S1, unofficial].
- A noise tolerance parameter $\epsilon$ [S1, unofficial].

**Outputs:**
- The reconstructed image $\mathbf{x}$.

**Data Structures:**
- $\mathbf{y}_1$: An $m \times 1$ column vector containing the measured horizontal gradient components at the $m$ sampled locations [S1, unofficial].
- $\mathbf{y}_2$: An $m \times 1$ column vector containing the measured vertical gradient components at the $m$ sampled locations [S1, unofficial].
- $\hat{\mathbf{x}}_1$: The 2D Discrete Fourier Transform (DFT) of the horizontal gradient component [S1, unofficial].
- $\hat{\mathbf{x}}_2$: The 2D DFT of the vertical gradient component [S1, unofficial].
- $\mathbf{F}^{-1}$: The inverse 2D DFT matrix [S1, unofficial].

**Steps:**

1. **Render Horizontal Gradient Component ($x_1$):**
   Compute the horizontal gradient measurements $\mathbf{y}_1$ by weighting incoming rays with the derivative of the pixel reconstruction kernel, or by using a box filter and applying a finite differencing scheme [S1, unofficial]. The sensing mechanism is formulated as $\mathbf{y}_1 = \mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_1$, where $\mathbf{y}_1$ is the measured gradient vector, $\mathbf{S}$ is the sampling matrix, $\mathbf{F}^{-1}$ is the inverse 2D DFT matrix, and $\hat{\mathbf{x}}_1$ is the 2D DFT of the horizontal gradient [S1, unofficial].

2. **Recover Horizontal Gradient DFT ($\hat{\mathbf{x}}_1$):**
   Solve the $l_1$ minimization problem to approximate $\hat{\mathbf{x}}_1$: $\min \|\hat{\mathbf{x}}_1\|_1$ subject to $\|\mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_1 - \mathbf{y}_1\|_2 \leq \epsilon$ [S1, unofficial]. This step exploits the sparsity of the gradient in the gradient domain [S1, unofficial].

3. **Render Vertical Gradient Component ($x_2$):**
   Compute the vertical gradient measurements $\mathbf{y}_2$ using the same method as step 1 [S1, unofficial].

4. **Recover Vertical Gradient DFT ($\hat{\mathbf{x}}_2$):**
   Solve the analogous $l_1$ minimization problem for the vertical component: $\min \|\hat{\mathbf{x}}_2\|_1$ subject to $\|\mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_2 - \mathbf{y}_2\|_2 \leq \epsilon$ [S1, unofficial].

5. **Reconstruct Image ($\mathbf{x}$):**
   Recover the pixel values from the gradient components by applying a Poisson solver with Dirichlet boundary conditions to the divergence of the gradient [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.fft import fft2, ifft2
from scipy.sparse import csr_matrix
from scipy.sparse.linalg import spsolve

def soft_threshold(x, tau):
    return np.sign(x) * np.maximum(np.abs(x) - tau, 0.0)

def solve_l1_minimization(y, sampling_indices, epsilon, H, W):
    N = H * W
    m = len(sampling_indices)
    mask = np.zeros((H, W))
    for r, c in sampling_indices:
        mask[r, c] = 1.0
    
    def forward_op(v):
        v_img = v.reshape(H, W)
        v_ifft = ifft2(v_img)
        return (v_ifft * mask).flatten()
        
    def adjoint_op(y_vec):
        y_img = y_vec.reshape(H, W)
        y_expanded = np.zeros((H, W))
        y_expanded[sampling_indices] = y_vec
        return fft2(y_expanded).flatten()
        
    # Lipschitz constant approximation for ISTA
    L = 1.0
    x_hat = np.zeros(N)
    tau = epsilon / L
    
    for _ in range(100):
        residual = forward_op(x_hat) - y
        grad = adjoint_op(residual)
        x_hat = soft_threshold(x_hat - (1.0 / L) * grad, tau)
        
    return x_hat

def compressive_gradient_rendering(volume, sampling_indices, epsilon, kernel_derivative_func, image_shape):
    H, W = image_shape
    N = H * W
    
    # 1 & 3. Render gradient measurements
    y1 = np.zeros(len(sampling_indices))
    y2 = np.zeros(len(sampling_indices))
    for i, (r, c) in enumerate(sampling_indices):
        y1[i] = kernel_derivative_func(volume, r, c, direction='horizontal')
        y2[i] = kernel_derivative_func(volume, r, c, direction='vertical')

    # 2 & 4. Recover DFT of gradients
    x_hat_1 = solve_l1_minimization(y1, sampling_indices, epsilon, H, W)
    x_hat_2 = solve_l1_minimization(y2, sampling_indices, epsilon, H, W)

    # 5. Reconstruct image via Poisson solver
    grad_x = np.real(ifft2(x_hat_1))
    grad_y = np.real(ifft2(x_hat_2))
    
    # Compute divergence of gradient (Laplacian RHS)
    rhs = np.zeros(N)
    for r in range(H):
        for c in range(W):
            idx = r * W + c
            if c > 0 and c < W - 1:
                rhs[idx] += (grad_x[r, c+1] - grad_x[r, c-1]) / 2.0
            elif c == 0:
                rhs[idx] += (grad_x[r, 1] - grad_x[r, 0])
            elif c == W - 1:
                rhs[idx] += (grad_x[r, c] - grad_x[r, c-1])
            if r > 0 and r < H - 1:
                rhs[idx] += (grad_y[r+1, c] - grad_y[r-1, c]) / 2.0
            elif r == 0:
                rhs[idx] += (grad_y[1, c] - grad_y[0, c])
            elif r == H - 1:
                rhs[idx] += (grad_y[r, c] - grad_y[r-1, c])

    # Build sparse Laplacian for interior points (Dirichlet BCs = 0)
    interior_mask = np.zeros((H, W), dtype=bool)
    interior_mask[1:-1, 1:-1] = True
    flat_interior = np.where(interior_mask)[0] * W + np.where(interior_mask)[1]
    n_int = len(flat_interior)
    
    rows, cols, vals = [], [], []
    for r in range(1, H-1):
        for c in range(1, W-1):
            idx = r * W + c
            rows.append(idx); cols.append(idx); vals.append(-4.0)
            if c > 1: rows.append(idx); cols.append(idx-1); vals.append(1.0)
            if c < W-2: rows.append(idx); cols.append(idx+1); vals.append(1.0)
            if r > 1: rows.append(idx); cols.append(idx-W); vals.append(1.0)
            if r < H-2: rows.append(idx); cols.append(idx+W); vals.append(1.0)
            
    A_lap = csr_matrix((vals, (rows, cols)), shape=(N, N))
    A_sub = A_lap[np.ix_(flat_interior, flat_interior)]
    rhs_sub = rhs[flat_interior]
    
    x_interior = spsolve(A_sub, rhs_sub)
    x = np.zeros((H, W))
    x[interior_mask] = x_interior
    return x
```

**Parameters:**
- $\epsilon$: Noise tolerance. Typical values depend on the noise level of the rendering process [S1, unofficial].
- Sampling matrix $\mathbf{S}$: Should contain random pixel locations to ensure $\mathbf{A} = \mathbf{S}\mathbf{F}^{-1}$ is a partial Fourier matrix [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The Poisson solver requires Dirichlet boundary conditions, and the choice of boundary values affects the reconstruction [S1, unofficial].
- **DC Component:** The Fourier transform of the gradient at the DC component (0,0) is zero, which must be handled when reconstructing the image from gradients [S1, unofficial].
- **Precision:** The $l_1$ minimization is sensitive to the choice of solver and parameters, and inaccurate recovery of $\hat{\mathbf{x}}_1$ and $\hat{\mathbf{x}}_2$ will lead to artifacts in the final image [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method involves rendering $m$ samples, solving two $l_1$ minimization problems, and solving a Poisson equation [S1, unofficial]. Not given in the sources.
- **Memory Complexity:** Requires storing the volume, the gradient measurements, and the intermediate Fourier coefficients [S1, unofficial]. Not given in the sources.
- **Strengths:** Exploits sparsity in the gradient domain, allowing for reconstruction from fewer samples than the Nyquist rate [S1, unofficial].
- **Limitations:** The $l_1$ minimization is computationally intensive and iterative [S1, unofficial].
- **When to Use:** Suitable for applications where rendering is expensive and the image is known to be sparse in the gradient domain [S1, unofficial].
- **When Not to Use:** Not given in the sources.

## Background
Compressive Gradient Rendering was introduced in the thesis "Compressive Volume Rendering" [S1, unofficial]. It builds on the principles of Compressive Sensing (CS), which suggests that signals sparse in some domain can be reconstructed from a small number of measurements [G]. The method adapts CS to volume rendering by rendering gradient components instead of pixel values, leveraging the sparsity of gradients in the gradient domain [S1, unofficial]. The image reconstruction from gradients uses a Poisson solver as described by Pérez et al. [S1, unofficial].

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
