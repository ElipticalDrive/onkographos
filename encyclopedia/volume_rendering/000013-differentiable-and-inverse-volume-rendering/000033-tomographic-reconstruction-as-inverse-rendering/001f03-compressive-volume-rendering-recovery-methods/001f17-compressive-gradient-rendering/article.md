# Compressive Gradient Rendering

Also called: CS-Gradient, Compressed Sensing Gradient, Gradient Recovery via CS, Gradient-based Compressive Rendering

## Overview
Compressive Gradient Rendering is a compressive volume rendering method that recovers high-quality images from sparse pixel samples by exploiting the sparsity of image gradients in the Discrete Fourier Transform (DFT) domain [S1][S2, unofficial]. Unlike sibling methods such as [CS-Wavelet](1915-compressive-volume-rendering-recovery-methods.md) which suffer from coherence issues with random sampling, this approach renders discrete gradient components ($x_1, x_2$) instead of pixel values, solves an $l_1$-minimization problem to recover the gradient's DFT, and reconstructs the image using a Poisson solver [S1][S2, unofficial]. It is distinguished by its use of the Fourier basis, which is inherently incoherent with point sampling, allowing for robust recovery when the fraction of missing pixels is moderate, though performance deteriorates significantly when missing pixels are high [S1].

## How it works
**Inputs:**
- A set of $m$ sampled pixel locations $\Omega$ in an image of size $N \times N$ (where $m \ll N^2$) [S1][S2, unofficial].
- The measured horizontal gradient values $\mathbf{y}_1$ at these locations [S1][S2, unofficial].
- The measured vertical gradient values $\mathbf{y}_2$ at these locations [S1][S2, unofficial].
- A reconstruction kernel basis (e.g., bilinear B-spline or Mitchell-Netravali cubic) [S1][S2, unofficial].

**Outputs:**
- The reconstructed image $\mathbf{x}$ of size $N \times N$ [S1][S2, unofficial].

**Data Structures:**
- $\mathbf{S}$: The sensing matrix of size $m \times N^2$. It selects the $m$ sampled locations [S1][S2, unofficial].
- $\mathbf{F}^{-1}$: The inverse 2D Discrete Fourier Transform (DFT) matrix of size $N^2 \times N^2$ [S1][S2, unofficial].
- $\hat{\mathbf{x}}_1, \hat{\mathbf{x}}_2$: The 2D DFT coefficients of the horizontal and vertical gradient components, respectively [S1][S2, unofficial].
- $\mathbf{y}_1, \mathbf{y}_2$: Measured gradient vectors of size $m \times 1$ [S1][S2, unofficial].

**Parameters:**
- $\epsilon$: Tolerance for the $l_1$-minimization solver. A typical value is $10^{-2}$, providing a tradeoff between recovery quality and runtime [S1][S2, unofficial].
- Kernel type: Bilinear B-spline or Mitchell-Netravali cubic [S1][S2, unofficial].

**Steps:**

1.  **Gradient Rendering:** Render the horizontal gradient component $x_1$ and vertical gradient component $x_2$ from the volume data [S2, unofficial]. This is achieved by differentiating the pixel reconstruction kernel used during ray casting [S1][S2, unofficial]. Alternatively, render the image using a box filter and apply a finite differencing scheme to obtain the gradients [S1][S2, unofficial].
2.  **Sampling Gradients:** Measure the horizontal gradient values at the $m$ sampled locations $\Omega$ to form the vector $\mathbf{y}_1$. Similarly, measure the vertical gradient values to form $\mathbf{y}_2$ [S1][S2, unofficial].
3.  **Formulate Sensing Model:** The relationship between the measured gradients and the DFT of the gradients is given by $\mathbf{y}_1 = \mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_1$ and $\mathbf{y}_2 = \mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_2$ [S1][S2, unofficial]. Here $\mathbf{S}$ is the sampling matrix, $\mathbf{F}^{-1}$ is the inverse 2D DFT matrix, and $\hat{\mathbf{x}}_1, \hat{\mathbf{x}}_2$ are the DFTs of the horizontal and vertical gradients [S1][S2, unofficial]. The matrix $\mathbf{A} = \mathbf{S}\mathbf{F}^{-1}$ is a partial Fourier matrix with the Restricted Isometry Property (RIC) and low coherence with point sampling [S1].
4.  **Recover Horizontal Gradient DFT ($\hat{\mathbf{x}}_1$):** Solve the $l_1$-minimization problem to find the sparsest approximation of the horizontal gradient's DFT: $\min \|\hat{\mathbf{x}}_1\|_1 \quad \text{subject to} \quad \|\mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_1 - \mathbf{y}_1\|_2 \leq \epsilon$ [S1][S2, unofficial]. This problem is typically solved using an iterative solver such as NESTA [S1][S2, unofficial].
5.  **Recover Vertical Gradient DFT ($\hat{\mathbf{x}}_2$):** Solve the analogous $l_1$-minimization problem for the vertical gradient [S2, unofficial].
6.  **Reconstruct Image via Poisson Solver:** Recover the pixel values $\mathbf{x}$ from the gradient components $\hat{\mathbf{x}}_1$ and $\hat{\mathbf{x}}_2$ [S2, unofficial]. Compute the inverse DFT of $\hat{\mathbf{x}}_1$ and $\hat{\mathbf{x}}_2$ to obtain the spatial gradient fields. Apply a Poisson solver with Dirichlet boundary conditions to the divergence of the gradient field to solve for $\mathbf{x}$ [S1][S2, unofficial]. The approach follows Pérez et al. [PGB03].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.fft import fft2, ifft2
from scipy.sparse import diags, kron, eye
from scipy.sparse.linalg import spsolve

def compressive_gradient_rendering(sampled_indices, N, y1, y2, epsilon=1e-2):
    """
    Reconstructs an image from sampled gradient measurements using Compressive Gradient Rendering.
    
    Args:
        sampled_indices: Array of shape (m, 2) containing (row, col) indices of sampled pixels.
        N: Image resolution (N x N).
        y1: Horizontal gradient values at sampled locations (shape (m,)).
        y2: Vertical gradient values at sampled locations (shape (m,)).
        epsilon: Tolerance for l1 minimization.
        
    Returns:
        x: Reconstructed image of shape (N, N).
    """
    m = len(sampled_indices)
    
    # Gradient rendering is performed by differentiating the reconstruction kernel 
    # or using a box filter with finite differencing [S1][S2, unofficial].
    # y1 and y2 are assumed to be pre-computed from the volume data.

    # 1. Construct sensing matrix S (m x N^2)
    flat_indices = sampled_indices[:, 0] * N + sampled_indices[:, 1]
    S = np.zeros((m, N * N))
    for i in range(m):
        S[i, flat_indices[i]] = 1.0

    # 2. Solve l1 minimization for horizontal gradient DFT
    x_hat_1 = solve_l1(S, N, y1, epsilon)
    # 3. Solve l1 minimization for vertical gradient DFT
    x_hat_2 = solve_l1(S, N, y2, epsilon)

    # 4. Reconstruct image from gradients via Poisson solver
    grad_x = np.real(ifft2(x_hat_1.reshape(N, N)))
    grad_y = np.real(ifft2(x_hat_2.reshape(N, N)))
    x = poisson_solver(grad_x, grad_y, N)

    return x

def solve_l1(S, N, y, epsilon):
    """
    Solves min ||x_hat||_1 s.t. ||S @ F_inv @ x_hat - y||_2 <= epsilon
    using Iterative Soft-Thresholding Algorithm (ISTA) [G].
    """
    x_hat = np.zeros(N * N)
    step_size = 1.0  # Lipschitz constant for S @ F_inv is 1.0
    max_iter = 1000
    threshold = epsilon * 0.1  # Heuristic threshold for sparsity [G]

    for _ in range(max_iter):
        # Compute spatial gradient from DFT coefficients
        spatial_grad = np.real(ifft2(x_hat.reshape(N, N)))
        # Compute residual: S @ spatial_grad - y
        residual = S @ spatial_grad.flatten() - y
        # Compute gradient of data fidelity: F @ S^T @ residual
        error_spatial = S.T @ residual
        grad = np.real(fft2(error_spatial.reshape(N, N)))
        # Proximal gradient step
        x_hat_new = x_hat - step_size * grad
        # Soft thresholding
        x_hat = np.sign(x_hat_new) * np.maximum(np.abs(x_hat_new) - threshold, 0)
    return x_hat

def poisson_solver(grad_x, grad_y, N):
    """
    Solves for x given horizontal gradient grad_x and vertical gradient grad_y.
    Uses Dirichlet boundary conditions [S2, unofficial].
    """
    # Compute divergence of the gradient field: div(g) = d(gx)/dx + d(gy)/dy
    div_g = np.zeros((N, N))
    # d(gx)/dx using central differences for interior, forward/backward for boundaries
    div_g[:, 1:-1] += (grad_x[:, 2:] - grad_x[:, :-2]) / 2.0
    div_g[:, 0] = grad_x[:, 1] - grad_x[:, 0]
    div_g[:, -1] = grad_x[:, -1] - grad_x[:, -2]
    # d(gy)/dy
    div_g[1:-1, :] += (grad_y[2:, :] - grad_y[:-2, :]) / 2.0
    div_g[0, :] = grad_y[1, :] - grad_y[0, :]
    div_g[-1, :] = grad_y[-1, :] - grad_y[-2, :]

    # Construct 2D Laplacian matrix (5-point stencil)
    e = np.ones(N)
    T = diags([-1, 2, -1], [-1, 0, 1], shape=(N, N), format='csr')
    A = kron(eye(N), T) + kron(T, eye(N))

    # Apply Dirichlet BCs: remove boundary rows/columns
    interior_indices = []
    for i in range(1, N - 1):
        for j in range(1, N - 1):
            interior_indices.append(i * N + j)

    A_interior = A[np.ix_(interior_indices, interior_indices)]
    b_interior = div_g.flatten()[interior_indices]

    # Solve linear system
    x_interior = spsolve(A_interior, b_interior)

    # Reconstruct full image
    x = np.zeros((N, N))
    for idx, flat_idx in enumerate(interior_indices):
        x[flat_idx // N, flat_idx % N] = x_interior[idx]

    return x
```

**Edge Cases and Pitfalls:**
-   **High Missing Pixel Fraction:** Performance deteriorates quickly when the fraction of missing pixels is high [S1].
-   **Boundary Conditions:** The Poisson solver requires boundary conditions [S2, unofficial].
-   **Solver Tolerance:** The tolerance $\epsilon$ must be chosen carefully; $10^{-2}$ is recommended as a tradeoff between quality and runtime [S1][S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Dominated by the $l_1$-minimization solver (e.g., NESTA) and the Poisson solver [S1]. The $l_1$ minimization is iterative and scales with the number of samples $m$ and image size $N$ [G].
-   **Memory Complexity:** Requires storage for the DFT coefficients ($O(N^2)$) and the sensing matrix operations [G].
-   **Measured Performance:** The method shows improvement over CS-Wavelet and Sen and Darabi's approach [SD11] in terms of recovery quality for moderate missing pixel fractions [S1].
-   **Strengths:** Exploits sparsity in the Fourier domain, which is inherently incoherent with point sampling, avoiding the coherence issues of wavelet-based methods [S1]. Gradient components are often sparser in the Fourier domain than the image itself [S1].
-   **Limitations:** Deteriorates significantly with high percentages of missing pixels [S1]. Requires solving two $l_1$-minimization problems and a Poisson solver, which can be computationally expensive [G].
-   **When to Use:** Suitable for compressive volume rendering when the fraction of missing pixels is moderate and high coherence with the sampling basis is a concern [S1].
-   **When Not to Use:** When the fraction of missing pixels is very high, or when computational resources are limited and iterative solvers are prohibitive [S1][G].

## Background
Compressive Gradient Rendering was introduced as part of the broader field of Compressive Volume Rendering [S1][S2, unofficial]. It extends the idea of Sen and Darabi [SD11] by rendering gradient components instead of the image itself [S1]. The method leverages the Restricted Isometry Property (RIC) of random partial Fourier matrices, which are known to have low coherence with point sampling [S1]. The approach was developed to address coherence issues found in CS-Wavelet methods and to exploit the natural sparsity of gradients in the Fourier domain [S1]. The Poisson solver step follows the approach of Pérez et al. [PGB03]. The method is discussed in the context of tomographic reconstruction as inverse rendering and is a key component of Compressive Volume Rendering Recovery Methods [S1][S2, unofficial].

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.1111/cgf.12622, fetched 2026-10-26
[S2] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
