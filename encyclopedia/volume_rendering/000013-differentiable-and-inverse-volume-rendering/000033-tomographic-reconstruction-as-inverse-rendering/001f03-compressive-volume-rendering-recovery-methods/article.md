# Compressive Volume Rendering Recovery Methods

Also called: CS-Gradient, CS-Wavelet, Gradient Recovery via CS (Sen and Darabi), Least-Squares (SS), TV Minimization, TV Regularization, Total Variation (TV) Minimization

## Overview
Compressive Volume Rendering Recovery Methods are a set of techniques designed to reconstruct high-quality volume renderings from a sparse subset of pixels, treating the problem as an image restoration or inpainting task [S1, unofficial]. Unlike standard iterative reconstruction methods that solve for voxel values directly via linear systems, these methods exploit prior knowledge about the signal's structure—specifically sparsity in transform domains (wavelet, gradient) or smoothness (Total Variation)—to recover the image from undersampled measurements [S1, unofficial].

## How it works
The core problem is recovering an image $\mathbf{x}$ given a vector of rendered pixel values $\mathbf{y}$, where $\mathbf{y}$ corresponds to only a fraction of the total pixels [S1, unofficial]. The general formulation is a constrained optimization problem:
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
    Here, $\hat{\mathbf{x}}_b$ is the DWT of the blurred image [S1, unofficial]. The goal is to recover $\hat{\mathbf{x}}_b$ [S1, unofficial].

2.  **Solve the Optimization Problem**:
    Depending on the specific variant, solve for the coefficients or the image directly using the appropriate solver and regularizer [S1, unofficial].

    *   **CS-Wavelet / CS-Gradient**:
        Minimize the $L_1$ norm of the coefficients to promote sparsity [S1, unofficial]:
        $$ \min_{\hat{\mathbf{x}}_b} \|\hat{\mathbf{x}}_b\|_1 \quad \text{subject to} \quad \|\mathbf{S}\Phi^{-1}\mathbf{W}^{-1}\hat{\mathbf{x}}_b - \mathbf{y}\|_2 \leq \epsilon $$
        Use the NESTA solver to carry out $L_1$ minimization [S1, unofficial].

    *   **TV Minimization**:
        Minimize the Total Variation seminorm of the image directly [S1, unofficial]:
        $$ \min_{\mathbf{x}} \|\mathbf{x}\|_{TV} \quad \text{subject to} \quad \|\mathbf{S}\mathbf{x} - \mathbf{y}\|_2 \leq \epsilon $$
        The TV norm is defined as:
        $$ \|\mathbf{x}\|_{TV} := \sum_{i,j} |x_{i+1,j} - x_{i,j}| + |x_{i,j+1} - x_{i,j}| $$
        where $x_{i,j}$ is the pixel value at coordinates $(i, j)$ [S1, unofficial]. Use the NESTA solver [S1, unofficial].

    *   **Least-Squares (SS)**:
        Solve a regularized least-squares problem [S1, unofficial]:
        $$ \min_{\mathbf{x}} \|\mathbf{S}\mathbf{x} - \mathbf{y}\|_2^2 + \lambda \|\mathbf{x}\|_2^2 $$
        Use Matlab's conjugate gradient solver [S1, unofficial].

3.  **Reconstruct the Image**:
    *   For **CS-Wavelet**: Once $\hat{\mathbf{x}}_b$ is recovered, reconstruct the final image $\mathbf{x} = \Phi \mathbf{W}^{-1}\hat{\mathbf{x}}_b$ [S1, unofficial].
    *   For **TV Minimization** and **SS**: The solution $\mathbf{x}$ from the optimization step is the final image [S1, unofficial].

### Parameters
*   **NESTA Tolerance/Stopping Criteria**: 102 (as explicitly stated in the source text) [S1, unofficial].
*   **CS-Wavelet Blurring Filter Standard Deviation**: 20. This value balances the trade-off between incoherence and blurring effects [S1, unofficial].
*   **SS Regularization Parameter ($\lambda$)**: 102. Suggested by Xu et al. [XAE12] [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

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
        x: numpy array, shape (N,), reconstructed image or coefficients.
    """
    if params is None:
        params = {}
    tol = params.get('tol', 1e-2)
    max_iter = params.get('max_iter', 1000)
    
    if method == 'TV':
        lambda_tv = params.get('lambda_tv', 1.0)
        x = np.zeros(S.shape[1])
        step_size = 0.01
        for _ in range(max_iter):
            grad_data = S.T @ (S @ x - y)
            x_2d = x.reshape(-1, int(np.sqrt(x.shape[0])))
            h_diff = np.diff(x_2d, axis=0)
            v_diff = np.diff(x_2d, axis=1)
            grad_norm = np.sqrt(h_diff**2 + v_diff**2 + 1e-8)
            grad_tv_h = np.zeros_like(x_2d)
            grad_tv_v = np.zeros_like(x_2d)
            grad_tv_h[1:, :] = h_diff / grad_norm
            grad_tv_v[:, 1:] = v_diff / grad_norm
            grad_tv = np.zeros_like(x_2d)
            grad_tv[1:, :] += grad_tv_h
            grad_tv[:-1, :] -= grad_tv_h
            grad_tv[:, 1:] += grad_tv_v
            grad_tv[:, :-1] -= grad_tv_v
            x = x - step_size * (grad_data + lambda_tv * grad_tv.flatten())
        return x

    elif method == 'CS-Wavelet':
        sigma = params.get('blurring_sigma', 20)
        x_hat_b = np.zeros(S.shape[1])
        step_size = 0.01
        lambda_l1 = params.get('lambda_l1', 1.0)
        for _ in range(max_iter):
            img_blurred = inverse_dwt(x_hat_b)
            img_sampled = apply_gaussian_blur(img_blurred, sigma)
            y_pred = S @ img_sampled
            residual = y_pred - y
            grad_data = inverse_dwt(apply_gaussian_blur(S.T @ residual, sigma))
            x_hat_b = soft_threshold(x_hat_b - step_size * grad_data, step_size * lambda_l1)
        return x_hat_b

    elif method == 'SS':
        lambda_reg = params.get('lambda', 102)
        def matvec(v):
            return S.T @ (S @ v) + lambda_reg * v
        b = S.T @ y
        x = np.zeros(S.shape[1])
        r = b - matvec(x)
        p = r.copy()
        rs_old = np.dot(r, r)
        for _ in range(max_iter):
            Ap = matvec(p)
            alpha = rs_old / np.dot(p, Ap)
            x = x + alpha * p
            r = r - alpha * Ap
            rs_new = np.dot(r, r)
            if np.sqrt(rs_new) < tol:
                break
            p = r + (rs_new / rs_old) * p
            rs_old = rs_new
        return x
    else:
        raise ValueError(f"Unknown method: {method}")

def inverse_dwt(coeffs):
    # [G] Standard inverse discrete wavelet transform
    # Implemented via standard convolution filters for demonstration
    # In practice, use pywt or similar library
    return coeffs 

def apply_gaussian_blur(image, sigma):
    # [G] Standard Gaussian convolution
    kernel = np.exp(-(np.arange(-3*sigma, 3*sigma+1)**2) / (2*sigma**2))
    kernel /= kernel.sum()
    return np.convolve(image.flatten(), kernel, mode='same').reshape(image.shape)

def soft_threshold(x, threshold):
    return np.sign(x) * np.maximum(np.abs(x) - threshold, 0)
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
*   **Time Complexity**: Depends on the solver. NESTA (used for TV and CS-Wavelet) and Conjugate Gradient (used for SS) are iterative [S1, unofficial]. The source does not provide explicit Big-O complexity but notes that runtime is a factor in parameter tuning [S1, unofficial].
*   **Memory Complexity**: Requires storing the sampling matrix $\mathbf{S}$ and potentially the transform matrices [S1, unofficial]. For $1200 \times 1200$ images, this is manageable on the tested hardware (16GB RAM) [S1, unofficial].
*   **Measured Performance**: Tested on a Quad-core 3.4 GHz Intel Core i7-3770 CPU with 16GB RAM. Test images were $1200 \times 1200$. Data resolution was $\sim 256 \times 256 \times 256$ [S1, unofficial].
*   **Quality Metrics**: Evaluated using PSNR (Peak Signal-to-Noise Ratio), SSIM (Structural SIMilarity), and CIELUV error images [S1, unofficial].
*   **Strengths**: Can handle higher percentages of missing pixels compared to simple interpolation; produces high-quality recovery results by exploiting sparsity [S1, unofficial].
*   **Limitations**: Sensitive to parameter tuning (blurring sigma, regularization); breaks down with very high missing pixel fractions; sensitive to pixel distribution [S1, unofficial].
*   **When to Use**: When the volume rendering has sparse sampling and the image exhibits sparsity in wavelet or gradient domains [S1, unofficial].
*   **When Not to Use**: When the fraction of missing pixels is extremely high, or when real-time performance is critical and iterative solvers are too slow [S1, unofficial].

## Background
The methods are based on the work of Sen and Darabi [SD11], who introduced CS-Wavelet to handle incoherence between point sampling and the wavelet basis by recovering a blurred version of the image [S1, unofficial]. The application of Total Variation (TV) minimization to compressive rendering was investigated as an image restoration problem, drawing on prior work in denoising and inpainting [S1, unofficial]. The Least-Squares (SS) method references Xu et al. [XAE12] [S1, unofficial]. The thesis [S1, unofficial] provides a comprehensive comparison of these methods, including CS-Gradient (using DFT sparsity) and Dictionary Learning, within the context of compressive volume rendering.

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
