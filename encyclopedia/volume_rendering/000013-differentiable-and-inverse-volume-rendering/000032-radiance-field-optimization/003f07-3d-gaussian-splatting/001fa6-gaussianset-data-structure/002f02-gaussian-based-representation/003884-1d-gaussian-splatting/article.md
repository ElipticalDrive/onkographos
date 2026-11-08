# 1D Gaussian Splatting

Also called: 1D Gaussian Representation

## Overview
1D Gaussian Splatting is a signal representation technique that decomposes a one-dimensional signal into a weighted sum of Gaussian probability density functions (kernels) [S1]. Unlike the Fourier transform, which provides a global frequency decomposition, this method offers a local representation where each kernel decays rapidly with respect to the input variable [S1]. It differs fundamentally from Gaussian Mixture Models (GMMs): while GMMs use Gaussians to model the *distribution* of data points, 1D Gaussian Splatting uses them to approximate the *value* of the signal itself [S1].

## How it works
**Inputs:**
- A ground truth signal $S(t)$ sampled at $T$ discrete points $t \in \{1, \dots, T\}$ [S1].
- The number of Gaussian kernels $K$ [S1].

**Outputs:**
- A reconstructed signal $\hat{S}(t)$ [S1].
- Optimized parameters for each kernel: mean $\mu_i$, standard deviation $\sigma_i$, and weight $w_i$ for $i=1 \dots K$ [S1].

**Data Structures:**
The representation consists of $K$ Gaussian kernels. Each kernel $i$ is defined by three scalar parameters:
1.  $\mu_i$: The mean (center) of the Gaussian.
2.  $\sigma_i$: The standard deviation (width) of the Gaussian.
3.  $w_i$: The weight (opacity) of the Gaussian.

To ensure numerical stability during optimization, the variances are initialized and stored as log-variances [S1].

**Formulas:**
The Gaussian kernel function $f_i(t)$ is defined as [S1]:
$$
f_i(t \mid \mu_i, \sigma_i) = \frac{1}{\sqrt{2\pi\sigma_i^2}} \exp\left(-\frac{(t-\mu_i)^2}{2\sigma_i^2}\right)
$$
The reconstructed signal $\hat{S}(t)$ is the weighted sum of these kernels [S1]:
$$
\hat{S}(t) = \sum_{i=1}^K w_i f_i(t)
$$
The optimization minimizes the Mean Squared Error (L2) loss between the reconstructed signal and the ground truth $S(t)$ [S1]:
$$
\mathcal{L} = \frac{1}{T} \sum_{t=1}^T (\hat{S}(t) - S(t))^2
$$

**Algorithm Steps:**
1.  **Initialization:** Initialize $K$ Gaussian kernels. Set means $\mu_i$ and standard deviations $\sigma_i$ randomly within reasonable ranges covering the domain of $t$ [S1]. Initialize variances as log-variances for numerical stability [S1]. Initialize weights $w_i$ (typically to 1 or small random values, though specific initialization for $w_i$ is not detailed beyond "optimize parameters") [S1].
2.  **Optimization:** Use gradient descent to optimize the parameters $\mu_i, \sigma_i, w_i$ to minimize $\mathcal{L}$ [S1].
    -   Optimizer: Adam [S1].
    -   Learning rate: 0.1 [S1].
    -   Iterations: 100,000 [S1].
3.  **Reconstruction:** Compute $\hat{S}(t)$ using the optimized parameters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_kernel(t, mu, sigma):
    """
    Computes the value of a 1D Gaussian kernel.
    t: array of input points
    mu: mean
    sigma: standard deviation
    """
    return (1.0 / np.sqrt(2 * np.pi * sigma**2)) * np.exp(-((t - mu)**2) / (2 * sigma**2))

def reconstruct_signal(t, K, mus, sigmas, weights):
    """
    Reconstructs the signal as a weighted sum of K Gaussians.
    t: array of input points
    K: number of kernels
    mus: array of means, shape (K,)
    sigmas: array of standard deviations, shape (K,)
    weights: array of weights, shape (K,)
    """
    signal = np.zeros_like(t)
    for i in range(K):
        signal += weights[i] * gaussian_kernel(t, mus[i], sigmas[i])
    return signal

def train_1d_gaussian_splatting(t_data, S_ground_truth, K, num_iterations=100000, lr=0.1):
    """
    Trains 1D Gaussian Splatting parameters.
    t_data: array of input points, shape (T,)
    S_ground_truth: array of target values, shape (T,)
    K: number of Gaussian kernels
    """
    T = len(t_data)
    
    # Initialize parameters
    # Means and sigmas initialized randomly within reasonable ranges of t
    mus = np.random.uniform(t_data.min(), t_data.max(), K)
    # Variances initialized as log-variances for stability
    log_variances = np.random.randn(K)
    sigmas = np.exp(log_variances)
    # Weights initialized (value not specified, assume 1.0 or small random)
    weights = np.ones(K)
    
    # Optimizer state (Adam)
    # Note: Standard Adam implementation required for gradients
    # For simplicity, we outline the gradient update logic
    
    for step in range(num_iterations):
        # Forward pass
        S_hat = reconstruct_signal(t_data, K, mus, sigmas, weights)
        
        # Loss calculation
        loss = np.mean((S_hat - S_ground_truth)**2)
        
        # Backward pass: Compute gradients of loss w.r.t mus, sigmas, weights
        # dL/dS_hat = 2/T * (S_hat - S_ground_truth)
        # dS_hat/dw_i = f_i(t)
        # dS_hat/dmu_i = w_i * df_i/dmu_i
        # dS_hat/dsigma_i = w_i * df_i/dsigma_i
        
        # Gradient of kernel w.r.t mu:
        # df/dmu = f(t) * (t - mu) / sigma^2
        # Gradient of kernel w.r.t sigma:
        # df/dsigma = f(t) * (t - mu)^2 / sigma^3 - f(t) / sigma
        
        # Compute gradients for each kernel
        grad_mus = np.zeros(K)
        grad_sigmas = np.zeros(K)
        grad_weights = np.zeros(K)
        
        for i in range(K):
            f_i = gaussian_kernel(t_data, mus[i], sigmas[i])
            error = S_hat - S_ground_truth
            
            # Gradient w.r.t weight
            grad_weights[i] = np.mean(2.0 / T * error * f_i)
            
            # Gradient w.r.t mean
            # df/dmu = f * (t - mu) / sigma^2
            df_dmu = f_i * (t_data - mus[i]) / (sigmas[i]**2)
            grad_mus[i] = np.mean(2.0 / T * error * weights[i] * df_dmu)
            
            # Gradient w.r.t sigma
            # df/dsigma = f * ((t - mu)^2 / sigma^3 - 1/sigma)
            df_dsigma = f_i * ((t_data - mus[i])**2 / (sigmas[i]**3) - 1.0/sigmas[i])
            grad_sigmas[i] = np.mean(2.0 / T * error * weights[i] * df_dsigma)
            
        # Update parameters using Adam optimizer
        # [G] Standard Adam update step applied to mus, sigmas, weights using their respective gradients
        # mus, sigmas, weights = adam_update(mus, sigmas, weights, grad_mus, grad_sigmas, grad_weights, lr)
        
    return mus, sigmas, weights
```

**Edge Cases and Pitfalls:**
-   **Smoothing Side-effect:** A limited number of Gaussians causes the reconstructed signal to be smoother than the ground truth, losing high local variance [S1]. This acts as a form of compression [S1].
-   **Oscillation:** If too many Gaussians are used or hyper-parameters are not tuned, reconstruction quality may oscillate (e.g., between 40 and 45 dB PSNR) [S1].
-   **Diminishing Returns:** Reconstruction quality only improves until a certain number of Gaussians is reached; adding more beyond this point may not help or may cause instability [S1].

## Operating characteristics
-   **Compression:** The method compresses data. For a signal with $T$ points, $K$ Gaussians require $2K$ parameters (mean and sigma) plus $K$ weights. In the Delhi temperature example (113 days), 45 Gaussians (90 parameters) achieved 47 dB PSNR, effectively compressing the data [S1].
-   **Performance:** In the cited experiment, 45 Gaussians achieved 47 dB PSNR. Quality oscillates between 40-45 dB after $K=47$ [S1].
-   **Strengths:** Local representation; effective compression for smooth or moderately varying signals; differentiable, allowing gradient-based optimization [S1].
-   **Limitations:** Limited capacity to represent high local variance without many kernels; potential for oscillation with poor hyper-parameter tuning; smoothing effect may be undesirable for signals with sharp features [S1].
-   **When to use:** Signal approximation, forecasting (where overall behavior is more interesting than small variations), and compression [S1].

## Background
1D Gaussian Splatting was introduced in the paper "Gaussian Splatting: An Introduction" [S1]. It is presented as an alternative to Fourier decomposition for signal representation, leveraging the local nature of Gaussian kernels [S1]. The method is distinct from Gaussian Mixture Models, which model data distributions rather than signal values [S1].

## Sources
[S1] Gaussian Splatting: An Introduction, https://doi.org/10.5201/ipol.2025.566, fetched 2026-10-26
