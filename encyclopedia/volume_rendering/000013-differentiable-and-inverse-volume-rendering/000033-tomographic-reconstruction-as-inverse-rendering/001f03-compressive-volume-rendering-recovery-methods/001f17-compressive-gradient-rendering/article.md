# Compressive Gradient Rendering

Also called: CS-Gradient, Compressed Sensing Gradient, Gradient Recovery via CS, Gradient-based Compressive Rendering

## Overview
Compressive Gradient Rendering is a compressive volume rendering method that recovers high-quality images from sparse pixel samples by exploiting the sparsity of image gradients in the Discrete Fourier Transform (DFT) domain [S1][S2, unofficial]. Unlike sibling methods such as [CS-Wavelet](1915-compressive-volume-rendering-recovery-methods.md) which suffer from coherence issues with random sampling, this approach renders discrete gradient components ($x_1, x_2$) instead of pixel values, solving an $l_1$-minimization problem to recover the gradient's DFT, and finally reconstructing the image using a Poisson solver [S1][S2, unofficial]. It is distinguished by its use of the Fourier basis, which is inherently incoherent with point sampling, allowing for robust recovery when the fraction of missing pixels is moderate, though performance deteriorates significantly when missing pixels are high [S1].

## How it works
**Inputs:**
- A set of $m$ sampled pixel locations $\Omega$ in an image of size $N \times N$ (where $m \ll N^2$).
- The measured horizontal gradient values $y_1$ at these locations.
- The measured vertical gradient values $y_2$ at these locations.
- A reconstruction kernel basis (e.g., bilinear B-spline or Mitchell-Netravali cubic) [S1][S2, unofficial].

**Outputs:**
- The reconstructed image $\mathbf{x}$ of size $N \times N$.

**Data Structures:**
- $\mathbf{S}$: The sensing matrix of size $m \times N^2$. It selects the $m$ sampled locations. In the context of the DFT formulation, it acts as a subsampling operator.
- $\mathbf{F}^{-1}$: The inverse 2D Discrete Fourier Transform (DFT) matrix of size $N^2 \times N^2$.
- $\hat{\mathbf{x}}_1, \hat{\mathbf{x}}_2$: The 2D DFT coefficients of the horizontal and vertical gradient components, respectively. These are vectors of size $N^2 \times 1$.
- $\mathbf{y}_1, \mathbf{y}_2$: Measured gradient vectors of size $m \times 1$.

**Parameters:**
- $\epsilon$: Tolerance for the $l_1$-minimization solver. A typical value is $10^{-2}$, providing a tradeoff between recovery quality and runtime [S1][S2, unofficial].
- Kernel type: Bilinear B-spline or Mitchell-Netravali cubic [S1][S2, unofficial].

**Steps:**

1.  **Gradient Rendering:**
    Render the horizontal gradient component $x_1$ and vertical gradient component $x_2$ from the volume data. This is achieved by differentiating the pixel reconstruction kernel used during ray casting.
    -   *Method A (Kernel Differentiation):* Instead of weighting incoming rays with the standard pixel reconstruction filter $K(u)$, weight them with the derivative of the kernel $K'(u)$ [S1][S2, unofficial].
    -   *Method B (Finite Differencing):* Render the image using a box filter and apply a finite differencing scheme to the resulting pixel values to obtain gradients [S1][S2, unofficial].
    Let the resulting rendered gradient images be $x_1$ and $x_2$.

2.  **Sampling Gradients:**
    Measure the horizontal gradient values at the $m$ sampled locations $\Omega$ to form the vector $\mathbf{y}_1$. Similarly, measure the vertical gradient values to form $\mathbf{y}_2$ [S1][S2, unofficial].

3.  **Formulate Sensing Model:**
    The relationship between the measured gradients and the DFT of the gradients is given by:
    $$ \mathbf{y}_1 = \mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_1 $$
    $$ \mathbf{y}_2 = \mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_2 $$
    where $\mathbf{S}$ is the sampling matrix (subsample operator), $\mathbf{F}^{-1}$ is the inverse 2D DFT matrix, and $\hat{\mathbf{x}}_1, \hat{\mathbf{x}}_2$ are the DFTs of the horizontal and vertical gradients [S1][S2, unofficial]. Note that $\mathbf{A} = \mathbf{S}\mathbf{F}^{-1}$ is a partial Fourier matrix, which possesses the Restricted Isometry Property (RIC) and low coherence with point sampling [S1].

4.  **Recover Horizontal Gradient DFT ($\hat{\mathbf{x}}_1$):**
    Solve the $l_1$-minimization problem to find the sparsest approximation of the horizontal gradient's DFT:
    $$ \min \|\hat{\mathbf{x}}_1\|_1 \quad \text{subject to} \quad \|\mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_1 - \mathbf{y}_1\|_2 \leq \epsilon $$
    This problem is typically solved using an iterative solver such as NESTA [S1][S2, unofficial].

5.  **Recover Vertical Gradient DFT ($\hat{\mathbf{x}}_2$):**
    Solve the analogous $l_1$-minimization problem for the vertical gradient:
    $$ \min \|\hat{\mathbf{x}}_2\|_1 \quad \text{subject to} \quad \|\mathbf{S}\mathbf{F}^{-1}\hat{\mathbf{x}}_2 - \mathbf{y}_2\|_2 \leq \epsilon $$

6.  **Reconstruct Image via Poisson Solver:**
    Recover the pixel values $\mathbf{x}$ from the gradient components $\hat{\mathbf{x}}_1$ and $\hat{\mathbf{x}}_2$.
    -   Compute the inverse DFT of $\hat{\mathbf{x}}_1$ and $\hat{\mathbf{x}}_2$ to obtain the spatial gradient fields $g_x$ and $g_y$.
    -   Apply a Poisson solver with Dirichlet boundary conditions to the divergence of the gradient field $\nabla \cdot \mathbf{g} = \frac{\partial g_x}{\partial x} + \frac{\partial g_y}{\partial y}$ to solve for $\mathbf{x}$ [S1][S2, unofficial]. The approach follows Pérez et al. [PGB03].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.fft import fft2, ifft2, ifftshift
from scipy.sparse.linalg import spsolve
from scipy.sparse import diags, kron, eye

def compressive_gradient_rendering(volume_data, sampled_indices, m, epsilon=1e-2):
    """
    Reconstructs an image from sampled gradient measurements using Compressive Gradient Rendering.
    
    Args:
        volume_data: 3D volume data (not directly used in reconstruction step, 
                     assumed gradients y1, y2 are pre-rendered).
        sampled_indices: Array of shape (m, 2) containing (row, col) indices of sampled pixels.
        m: Number of samples.
        epsilon: Tolerance for l1 minimization.
        
    Returns:
        x: Reconstructed image of shape (N, N).
    """
    # Assume N is derived from volume_data or known context. 
    # For the sake of this self-contained example, we assume N is passed or inferred.
    # In practice, N is the image resolution.
    N = 256 # Placeholder for image dimension
    
    # 1. Render Gradients (Pre-processing step described in text)
    # Method A: Differentiate kernel during ray casting.
    # Method B: Render with box filter, then finite difference.
    # Here we assume y1 and y2 are the measured gradient values at sampled_indices.
    # y1: horizontal gradient values at sampled locations
    # y2: vertical gradient values at sampled locations
    y1 = render_gradient(volume_data, sampled_indices, direction='horizontal')
    y2 = render_gradient(volume_data, sampled_indices, direction='vertical')
    
    # 2. Construct Sensing Matrix S
    # S is a subsampling matrix of shape (m, N*N)
    # It selects the rows corresponding to sampled_indices
    flat_indices = sampled_indices[:, 0] * N + sampled_indices[:, 1]
    S = np.zeros((m, N * N))
    for i in range(m):
        S[i, flat_indices[i]] = 1.0
        
    # 3. Define Inverse DFT Matrix F_inv
    # We do not construct the full N^2 x N^2 matrix explicitly for efficiency,
    # but the operation S @ F_inv @ x_hat is equivalent to:
    # 1. Compute FFT of x_hat (which is F_inv @ x_hat in frequency domain? No.)
    # Let's clarify: y = S * F_inv * x_hat.
    # F_inv is the inverse DFT matrix. 
    # If x_hat is the DFT coefficients, then F_inv @ x_hat gives the spatial gradient image.
    # S selects pixels from that spatial image.
    # So, y = S @ (spatial_gradient).
    # The optimization variable is x_hat (DFT coefficients).
    
    # 4. Solve l1 minimization for Horizontal Gradient
    # min ||x_hat_1||_1 subject to ||S @ F_inv @ x_hat_1 - y1||_2 <= epsilon
    
    # We use a simple iterative soft-thresholding algorithm (ISTA) or similar for demonstration.
    # Note: The source specifies NESTA. We implement a basic proximal gradient step.
    
    x_hat_1 = solve_l1_minimization(S, N, y1, epsilon)
    x_hat_2 = solve_l1_minimization(S, N, y2, epsilon)
    
    # 5. Reconstruct Image from Gradients
    # Convert DFT coefficients back to spatial gradients
    grad_x = np.real(ifft2(x_hat_1.reshape(N, N)))
    grad_y = np.real(ifft2(x_hat_2.reshape(N, N)))
    
    # 6. Poisson Solver
    # Solve div(grad) = Laplacian(x)
    # We need to compute the divergence of the gradient field.
    # However, the gradients are derived from the DFT of the *rendered* gradients.
    # The Poisson solver recovers x such that dx = grad_x and dy = grad_y.
    # This is equivalent to solving Laplacian(x) = div(grad_field).
    # But wait, the gradients x1, x2 ARE the derivatives of x.
    # So we have dx = x1, dy = x2.
    # The Poisson solver solves for x given dx and dy.
    
    x = poisson_solver(grad_x, grad_y, N)
    
    return x

def solve_l1_minimization(S, N, y, epsilon):
    """
    Solves min ||x_hat||_1 s.t. ||S @ F_inv @ x_hat - y||_2 <= epsilon
    Using a simplified ISTA approach for demonstration.
    """
    # Initialize x_hat
    x_hat = np.zeros(N * N)
    
    # Parameters for ISTA
    L = 1.0 # Lipschitz constant approximation, needs tuning
    max_iter = 1000
    
    for i in range(max_iter):
        # Compute spatial gradient from DFT coefficients
        # F_inv @ x_hat
        spatial_grad = np.real(ifft2(x_hat.reshape(N, N)))
        
        # Compute residual
        residual = S @ spatial_grad.flatten() - y
        
        # Check constraint (approximate)
        if np.linalg.norm(residual) <= epsilon:
            break
            
        # Gradient of data fidelity term: (S @ F_inv)^T @ (S @ F_inv @ x_hat - y)
        # This is complex to compute directly without large matrices.
        # We use the property that adjoint of S @ F_inv is F @ S^T.
        # So gradient is F @ S^T @ residual.
        
        # 1. S^T @ residual: maps m-dim residual to N^2-dim spatial gradient error
        error_spatial = S.T @ residual # Shape (N*N,)
        
        # 2. Reshape to 2D and apply FFT (which is F, the forward DFT)
        error_spatial_2d = error_spatial.reshape(N, N)
        error_freq = fft2(error_spatial_2d)
        
        # 3. Update x_hat
        x_hat_new = x_hat - (1.0/L) * error_freq
        
        # 4. Soft thresholding (proximal operator for l1 norm)
        # Threshold value lambda. In NESTA, this is adaptive.
        # For simplicity, we use a fixed small lambda or derive from epsilon.
        # The source uses NESTA which handles this automatically.
        # Here we apply a simple soft threshold.
        lambda_val = epsilon * 0.1 # Heuristic
        x_hat = np.sign(x_hat_new) * np.maximum(np.abs(x_hat_new) - lambda_val, 0)
        
    return x_hat

def poisson_solver(grad_x, grad_y, N):
    """
    Solves for x given horizontal gradient grad_x and vertical gradient grad_y.
    Uses Dirichlet boundary conditions (x=0 on boundary).
    """
    # Compute divergence of the gradient field
    # div(g) = d(gx)/dx + d(gy)/dy
    # Using finite differences for the Laplacian system
    # Laplacian(x) = div(g)
    
    # Create divergence source term
    # Using central differences for interior, forward/backward for boundaries
    div_g = np.zeros((N, N))
    
    # d(gx)/dx
    div_g[:, 1:-1] += (grad_x[:, 2:] - grad_x[:, :-2]) / 2.0
    div_g[:, 0] = grad_x[:, 1] - grad_x[:, 0]
    div_g[:, -1] = grad_x[:, -1] - grad_x[:, -2]
    
    # d(gy)/dy
    div_g[1:-1, :] += (grad_y[2:, :] - grad_y[:-2, :]) / 2.0
    div_g[0, :] = grad_y[1, :] - grad_y[0, :]
    div_g[-1, :] = grad_y[-1, :] - grad_y[-2, :]
    
    # Solve Laplacian(x) = div_g with Dirichlet BCs (x=0 on boundary)
    # Discrete Laplacian matrix A
    # A is sparse, size N^2 x N^2
    # We use a simple iterative solver or sparse direct solver
    
    # Construct Laplacian matrix for 2D grid
    # Standard 5-point stencil
    e = np.ones(N)
    T = diags([-1, 2, -1], [-1, 0, 1], shape=(N, N), format='csr')
    A = kron(eye(N), T) + kron(T, eye(N))
    
    # Flatten div_g
    b = div_g.flatten()
    
    # Apply Dirichlet BCs: x=0 on boundary.
    # The equations for boundary nodes are removed or modified.
    # Here we simply solve for interior nodes and set boundary to 0.
    # Indices of interior nodes
    interior_indices = []
    for i in range(1, N-1):
        for j in range(1, N-1):
            interior_indices.append(i * N + j)
            
    A_interior = A[np.ix_(interior_indices, interior_indices)]
    b_interior = b[interior_indices]
    
    # Solve
    x_interior = spsolve(A_interior, b_interior)
    
    # Reconstruct full image
    x = np.zeros((N, N))
    for idx, flat_idx in enumerate(interior_indices):
        x[flat_idx // N, flat_idx % N] = x_interior[idx]
        
    return x

def render_gradient(volume_data, sampled_indices, direction):
    """
    Placeholder for the actual volume rendering step.
    Returns gradient values at sampled_indices.
    """
    # Implementation depends on volume renderer.
    # If using kernel differentiation:
    # Weight rays with derivative of kernel.
    # If using finite difference:
    # Render image with box filter, then compute differences.
    return np.zeros(len(sampled_indices))
```

**Edge Cases and Pitfalls:**
-   **High Missing Pixel Fraction:** Performance deteriorates quickly when the fraction of missing pixels is high. The method is sensitive to the distribution of pixels and can break down under extreme sparsity [S1].
-   **Boundary Conditions:** The Poisson solver requires boundary conditions. Dirichlet conditions (typically zero) are used, which may introduce artifacts if the image boundaries are not zero [S1][S2, unofficial].
-   **Solver Tolerance:** The tolerance $\epsilon$ must be chosen carefully. A value of $10^{-2}$ is recommended as a tradeoff between quality and runtime [S1][S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Dominated by the $l_1$-minimization solver (e.g., NESTA) and the Poisson solver. The $l_1$ minimization is iterative and scales with the number of samples $m$ and image size $N$. The Poisson solver scales with $N^2$.
-   **Memory Complexity:** Requires storage for the DFT coefficients ($O(N^2)$) and the sensing matrix operations. The full sensing matrix $\mathbf{S}$ is sparse, but operations involving $\mathbf{F}^{-1}$ are dense in the frequency domain.
-   **Measured Performance:** The method shows improvement over CS-Wavelet and Sen and Darabi's approach [SD11] in terms of recovery quality for moderate missing pixel fractions [S1].
-   **Strengths:** Exploits sparsity in the Fourier domain, which is inherently incoherent with point sampling, avoiding the coherence issues of wavelet-based methods [S1]. Gradient components are often sparser in the Fourier domain than the image itself [S1].
-   **Limitations:** Deteriorates significantly with high percentages of missing pixels [S1]. Requires solving two $l_1$-minimization problems and a Poisson solver, which can be computationally expensive.
-   **When to Use:** Suitable for compressive volume rendering when the fraction of missing pixels is moderate and high coherence with the sampling basis is a concern.
-   **When Not to Use:** When the fraction of missing pixels is very high, or when computational resources are limited and iterative solvers are prohibitive.

## Background
Compressive Gradient Rendering was introduced as part of the broader field of Compressive Volume Rendering. It extends the idea of Sen and Darabi [SD11] by rendering gradient components instead of the image itself [S1]. The method leverages the Restricted Isometry Property (RIC) of random partial Fourier matrices, which are known to have low coherence with point sampling [S1]. The approach was developed to address coherence issues found in CS-Wavelet methods and to exploit the natural sparsity of gradients in the Fourier domain [S1]. The Poisson solver step follows the approach of Pérez et al. [PGB03]. The method is discussed in the context of tomographic reconstruction as inverse rendering and is a key component of Compressive Volume Rendering Recovery Methods [S1][S2, unofficial].

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.1111/cgf.12622, fetched 2026-10-26
[S2] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
