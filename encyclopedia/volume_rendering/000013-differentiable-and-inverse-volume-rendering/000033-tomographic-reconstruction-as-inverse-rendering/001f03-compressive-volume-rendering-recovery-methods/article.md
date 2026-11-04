# Compressive Volume Rendering Recovery Methods

Also called: CS-Gradient, CS-Wavelet, Gradient Recovery via CS (Sen and Darabi), Least-Squares (SS), TV Minimization, TV Regularization, Total Variation (TV) Minimization

## Overview
Compressive Volume Rendering Recovery Methods are a set of techniques designed to reconstruct high-quality volume renderings from a sparse subset of pixels, treating the problem as an image restoration or inpainting task [S1, unofficial]. Unlike standard iterative reconstruction methods that solve for voxel values directly via linear systems, these methods exploit prior knowledge about the signal's structure—specifically sparsity in transform domains (wavelet, gradient) or smoothness (Total Variation)—to recover the image from undersampled measurements [S1, unofficial]. They are distinguished from siblings like [SIRT](3101-sirt-simultaneous-iterative-reconstruction-technique.md) by their reliance on non-linear optimization of sparsity-promoting regularizers rather than simple algebraic updates, and from [Deep Image Prior Reconstruction](1221-deep-image-prior-reconstruction.md) by their use of explicit mathematical priors rather than neural network implicit biases [S1, unofficial].

## How it works
The core problem is recovering an image $\mathbf{x}$ given a vector of rendered pixel values $\mathbf{y}$, where $\mathbf{y}$ corresponds to only a fraction of the total pixels. The general formulation is a constrained optimization problem:
$$ \min_{\mathbf{x}} R(\mathbf{x}) \quad \text{subject to} \quad \|\mathbf{S}\mathbf{x} - \mathbf{y}\|_2 \leq \epsilon $$
where $\mathbf{S}$ is the sampling matrix (selecting the observed pixels), $R(\mathbf{x})$ is a regularizer promoting the desired prior (sparsity or smoothness), and $\epsilon$ is a noise tolerance [S1, unofficial].

### Data Structures
*   **Image $\mathbf{x}$**: A 2D vector of pixel values, size $N \times 1$, where $N$ is the total number of pixels in the rendered image (e.g., $1200 \times 1200$) [S1, unofficial].
*   **Sampling Matrix $\mathbf{S}$**: A binary matrix of size $M \times N$, where $M$ is the number of observed pixels. $S_{ij} = 1$ if pixel $j$ is observed, $0$ otherwise [S1, unofficial].
*   **Transform Matrices**:
    *   $\mathbf{W}^{-1}$: Inverse Discrete Wavelet Transform (DWT) matrix [S1, unofficial].
    *   $\Phi^{-1}$: Gaussian blurring matrix (convolution kernel) [S1, unofficial].
*   **Coefficients $\hat{\mathbf{x}}_b$**: The sparse representation of the blurred image in the transform domain [S1, unofficial].

### Steps

1.  **Formulate the Measurement Equation**:
    For methods exploiting wavelet sparsity (CS-Wavelet), the measurement model incorporates a blurring step to improve incoherence between the sampling matrix and the wavelet basis [S1, unofficial]:
    $$ \mathbf{y} = \mathbf{S}\Phi^{-1}\mathbf{W}^{-1}\hat{\mathbf{x}}_b $$
    Here, $\hat{\mathbf{x}}_b$ is the DWT of the blurred image. The goal is to recover $\hat{\mathbf{x}}_b$ [S1, unofficial].

2.  **Solve the Optimization Problem**:
    Depending on the specific variant, solve for the coefficients or the image directly using the appropriate solver and regularizer.

    *   **CS-Wavelet / CS-Gradient**:
        Minimize the $L_1$ norm of the coefficients to promote sparsity.
        $$ \min_{\hat{\mathbf{x}}_b} \|\hat{\mathbf{x}}_b\|_1 \quad \text{subject to} \quad \|\mathbf{S}\Phi^{-1}\hat{\mathbf{x}}_b - \mathbf{y}\|_2 \leq \epsilon $$
        Use the NESTA solver [BBC11] to carry out $L_1$ minimization [S1, unofficial].

    *   **TV Minimization**:
        Minimize the Total Variation seminorm of the image directly.
        $$ \min_{\mathbf{x}} \|\mathbf{x}\|_{TV} \quad \text{subject to} \quad \|\mathbf{S}\mathbf{x} - \mathbf{y}\|_2 \leq \epsilon $$
        The TV norm is defined as:
        $$ \|\mathbf{x}\|_{TV} := \sum_{i,j} |x_{i+1,j} - x_{i,j}| + |x_{i,j+1} - x_{i,j}| $$
        where $x_{i,j}$ is the pixel value at coordinates $(i, j)$ [S1, unofficial]. Use the NESTA solver [S1, unofficial].

    *   **Least-Squares (SS)**:
        Solve a regularized least-squares problem.
        $$ \min_{\mathbf{x}} \|\mathbf{S}\mathbf{x} - \mathbf{y}\|_2^2 + \lambda \|\mathbf{x}\|_2^2 $$
        Use Matlab's conjugate gradient solver [S1, unofficial].

3.  **Reconstruct the Image**:
    *   For **CS-Wavelet**: Once $\hat{\mathbf{x}}_b$ is recovered, reconstruct the blurred image $\mathbf{x}_b = \mathbf{W}^{-1}\hat{\mathbf{x}}_b$. The final image $\mathbf{x}$ is obtained by applying the inverse DWT to the recovered coefficients (effectively $\mathbf{x} = \mathbf{W}^{-1}\hat{\mathbf{x}}_b$ if $\hat{\mathbf{x}}_b$ represents the coefficients of the final image, or deblurring if $\hat{\mathbf{x}}_b$ is strictly for the blurred version; the source states "From the recovered coefficients $\hat{\mathbf{x}}_b$, the final image is obtained according to $\mathbf{x} = \mathbf{W}^{-1}\hat{\mathbf{x}}_b$" [S1, unofficial]).
    *   For **TV Minimization** and **SS**: The solution $\mathbf{x}$ from the optimization step is the final image [S1, unofficial].

### Parameters
*   **NESTA Tolerance/Stopping Criteria**: $10^{-2}$ (Source text says "102", likely a typo for $10^{-2}$ or $1e-2$, but explicitly cited as 102 in the parameter list) [S1, unofficial].
*   **CS-Wavelet Blurring Filter Standard Deviation**: 20. This value balances the trade-off between incoherence (higher sigma) and blurring effects (lower sigma) [S1, unofficial].
*   **SS Regularization Parameter ($\lambda$)**: 102. Suggested by Xu et al. [XAE12] [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.sparse import csr_matrix
from scipy.sparse.linalg import LinearOperator, cg

def compressive_volume_rendering_recovery(y, S, method='TV', params=None):
    """
    Reconstructs an image from sparse pixel measurements y using compressive sensing.
    
    Args:
        y: numpy array, shape (M,), observed pixel values.
        S: numpy array or sparse matrix, shape (M, N), sampling matrix.
        method: str, one of 'TV', 'CS-Wavelet', 'SS'.
        params: dict, optional parameters.
            - 'blurring_sigma': float, standard deviation for Gaussian blur (CS-Wavelet).
            - 'tol': float, tolerance for NESTA/CG.
            - 'lambda': float, regularization parameter for SS.
            
    Returns:
        x: numpy array, shape (N,), reconstructed image.
    """
    if params is None:
        params = {}
        
    tol = params.get('tol', 1e-2)
    
    if method == 'TV':
        # Minimize TV subject to data fidelity
        # Using a simplified gradient descent or proximal method for TV minimization
        # Note: NESTA is referenced in source, but standard Python implementation
        # would use FISTA or ADMM for TV minimization.
        # Here we outline the structure.
        
        # Initialize x
        x = np.zeros(S.shape[1])
        
        # Define TV norm gradient (subgradient)
        def tv_gradient(x_img):
            # x_img is reshaped to 2D
            h_diff = np.diff(x_img, axis=0)
            v_diff = np.diff(x_img, axis=1)
            # Gradient of L1 norm of gradients
            # This is a simplified representation; actual implementation requires
            # handling of the L1 norm subgradient carefully.
            grad_h = np.zeros_like(x_img)
            grad_v = np.zeros_like(x_img)
            # ... (Implementation of TV gradient via divergence of normalized gradients)
            return grad_h + grad_v

        # Optimization loop (e.g., FISTA)
        # x = optimize_tv_minimization(y, S, tol)
        # [G] Placeholder for specific TV solver implementation as NESTA is Matlab-specific.
        # The source specifies using NESTA for TV minimization.
        raise NotImplementedError("TV minimization requires a specific solver like NESTA or FISTA. See source [S1, unofficial].")

    elif method == 'CS-Wavelet':
        sigma = params.get('blurring_sigma', 20)
        
        # 1. Create Gaussian Blurring Matrix Phi_inv
        # In practice, this is a convolution. For matrix form, it's a sparse Toeplitz-like matrix.
        # A = S @ Phi_inv @ W_inv
        # We need to solve for x_hat_b in y = A @ x_hat_b
        
        # Define the combined operator A
        # Note: W_inv is Inverse DWT. Phi_inv is Gaussian Blur.
        # We solve min ||x_hat_b||_1 s.t. ||S @ Phi_inv @ W_inv @ x_hat_b - y||_2 <= eps
        
        # [G] Implementing NESTA in pure Python is complex. 
        # We outline the linear system setup.
        
        # Create a function that applies A to a vector
        def apply_A(v):
            # v is coefficients in wavelet domain
            # 1. Inverse DWT to get blurred image
            img_blurred = inverse_dwt(v)
            # 2. Sample pixels
            # Note: Phi_inv is applied before sampling in the source equation y = S Phi_inv W_inv x_hat_b
            # So we apply Phi_inv to the image? No, Phi_inv is the blur matrix.
            # x_b = Phi_inv x. The source says x_b = Phi x (blur).
            # Equation: y = S Phi_inv W_inv x_hat_b.
            # Let's assume x_hat_b are coefficients of the blurred image.
            # So: img_blurred = W_inv @ x_hat_b.
            # Then y = S @ img_blurred.
            # Wait, the source says: "recover a blurred version x_b ... x_b = Phi x".
            # And "y = S Phi_inv W_inv x_hat_b".
            # This implies x_hat_b are coefficients of x_b.
            # So: x_b = W_inv @ x_hat_b.
            # And y = S @ x_b.
            # But where does Phi_inv come in?
            # Source: "x_b = Phi x" (blur). "y = S Phi_inv W_inv x_hat_b".
            # If x_hat_b are coefficients of x_b, then x_b = W_inv x_hat_b.
            # Then y = S x_b.
            # The term Phi_inv in the source equation y = S Phi_inv W_inv x_hat_b seems to 
            # imply that the measurement is taken on the blurred image, but the coefficients
            # are of the blurred image.
            # Actually, the source says: "recover a blurred version x_b ... x_b = Phi x".
            # And "y = S Phi_inv W_inv x_hat_b".
            # If x_hat_b is the DWT of x_b, then x_b = W_inv x_hat_b.
            # So y = S x_b.
            # The Phi_inv term in the source equation (3.7) might be a typo or refers to 
            # the relationship between the original image and the blurred image in the 
            # optimization constraint.
            # Let's stick to the source's explicit equation: y = S @ Phi_inv @ W_inv @ x_hat_b.
            # This implies the measurement is S applied to (Phi_inv applied to (W_inv applied to x_hat_b)).
            # This is confusing. Let's look at the text: "x_b = Phi x". "recover x_b".
            # "y = S Phi_inv W_inv x_hat_b".
            # If x_hat_b is DWT of x_b, then W_inv x_hat_b = x_b.
            # Then y = S Phi_inv x_b.
            # This implies the measurement is taken on Phi_inv x_b?
            # Or is Phi_inv the inverse of the blur?
            # "Phi corresponding to the Gaussian blurring filter".
            # Usually Phi is the blur. Phi_inv would be deblur.
            # If y = S x_b, then Phi_inv is not needed in the forward model if x_hat_b is coeffs of x_b.
            # However, the source explicitly writes: y = S Phi_inv W_inv x_hat_b.
            # We will implement the matrix-vector product as defined.
            
            # Step 1: Inverse DWT
            img = inverse_dwt(v)
            # Step 2: Apply Phi_inv (Deblurring? Or is Phi_inv the blur matrix itself?)
            # Source: "Phi^-1 is the Gaussian blurring matrix". This is contradictory notation.
            # Usually Phi is the blur. If Phi^-1 is the blur, then Phi is deblur.
            # Let's assume the matrix named Phi_inv in the source is the Blur Matrix.
            img_blurred = apply_gaussian_blur(img, sigma)
            # Step 3: Sample
            y_pred = S @ img_blurred
            return y_pred

        # Solve L1 minimization using NESTA-like algorithm
        # x_hat_b = nest_solve(apply_A, y, tol)
        raise NotImplementedError("NESTA solver is Matlab-specific. See source [S1, unofficial].")

    elif method == 'SS':
        # Least Squares with L2 regularization
        # min ||Sx - y||^2 + lambda ||x||^2
        # Normal equations: (S^T S + lambda I) x = S^T y
        
        lambda_reg = params.get('lambda', 102)
        
        # Construct the linear system
        # A = S^T S + lambda I
        # b = S^T y
        
        # Using conjugate gradient
        def matvec(v):
            return S.T @ (S @ v) + lambda_reg * v
            
        A_op = LinearOperator((S.shape[1], S.shape[1]), matvec=matvec)
        b = S.T @ y
        
        x, status = cg(A_op, b, tol=tol)
        return x

    else:
        raise ValueError(f"Unknown method: {method}")

def inverse_dwt(coeffs):
    # Placeholder for Inverse DWT
    # [G] Use pywt.idwt or similar
    pass

def apply_gaussian_blur(image, sigma):
    # Placeholder for Gaussian Blur
    # [G] Use scipy.ndimage.gaussian_filter
    pass
```

### Edge Cases and Pitfalls
*   **Blurring Variance Sensitivity**: The method is sensitive to the variance of the Gaussian blurring filter. It must be adjusted on a case-by-case basis to balance incoherence and blurring effects [S1, unofficial].
*   **High Missing Pixel Fraction**: The method breaks down when the fraction of missing pixels is high [S1, unofficial].
*   **Pixel Distribution**: The method is sensitive to the distribution of the observed pixels [S1, unofficial].

## Subtopics

### [Global Solver](4281-global-solver.md)
A compressive volume rendering reconstruction approach where the image is reconstructed globally with the solver running the entire image at once.

### [Compressive Volume Rendering](4299-compressive-volume-rendering.md)
A technique to render a full image by using only a subset of pixels, reconstructed using an image representation model.

## Operating characteristics
*   **Time Complexity**: Depends on the solver. NESTA (used for TV and CS-Wavelet) and Conjugate Gradient (used for SS) are iterative. The source does not provide explicit Big-O complexity but notes that runtime is a factor in parameter tuning [S1, unofficial].
*   **Memory Complexity**: Requires storing the sampling matrix $\mathbf{S}$ and potentially the transform matrices. For $1200 \times 1200$ images, this is manageable on the tested hardware (16GB RAM) [S1, unofficial].
*   **Measured Performance**: Tested on a Quad-core 3.4 GHz Intel Core i7-3770 CPU with 16GB RAM. Test images were $1200 \times 1200$. Data resolution was $\sim 256 \times 256 \times 256$ [S1, unofficial].
*   **Quality Metrics**: Evaluated using PSNR (Peak Signal-to-Noise Ratio), SSIM (Structural SIMilarity), and CIELUV error images [S1, unofficial].
*   **Strengths**: Can handle higher percentages of missing pixels compared to simple interpolation; produces high-quality recovery results by exploiting sparsity [S1, unofficial].
*   **Limitations**: Sensitive to parameter tuning (blurring sigma, regularization); breaks down with very high missing pixel fractions; sensitive to pixel distribution [S1, unofficial].
*   **When to Use**: When the volume rendering has sparse sampling and the image exhibits sparsity in wavelet or gradient domains.
*   **When Not to Use**: When the fraction of missing pixels is extremely high, or when real-time performance is critical and iterative solvers are too slow.

## Background
The methods are based on the work of Sen and Darabi [SD11], who introduced CS-Wavelet to handle incoherence between point sampling and the wavelet basis by recovering a blurred version of the image [S1, unofficial]. The application of Total Variation (TV) minimization to compressive rendering was investigated as an image restoration problem, drawing on prior work in denoising and inpainting [S1, unofficial]. The Least-Squares (SS) method references Xu et al. [XAE12] [S1, unofficial]. The thesis [S1, unofficial] provides a comprehensive comparison of these methods, including CS-Gradient (using DFT sparsity) and Dictionary Learning, within the context of compressive volume rendering.

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
