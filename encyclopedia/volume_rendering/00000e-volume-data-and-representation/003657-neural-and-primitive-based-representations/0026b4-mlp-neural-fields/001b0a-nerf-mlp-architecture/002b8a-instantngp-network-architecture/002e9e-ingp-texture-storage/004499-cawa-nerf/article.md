# CAwa-NeRF

Also called: Compression-Aware NeRF, Instant Learning of Compression-Aware NeRF Features

## Overview
CAwa-NeRF is a method for the instant learning of compression-aware NeRF features that applies an entropy-aware training scheme with a rate-distortion trade-off to learn low entropy features, allowing export of zip-compressed feature grids with negligible overhead without changing the storage architecture or parameters of the base INGP method [S1]. It builds on [iNGP Texture Storage](5826-ingp-texture-storage.md) by modifying the training objective to minimize the information content of the learned feature grids, enabling efficient post-training compression [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A scene dataset (images and poses), the INGP hyperparameters (number of levels $L$, hash table size $T$, features per level $F$, resolutions $N_{min}, N_{max}$), and training configuration (iterations, learning rate).
*   **Outputs:** A trained neural field model with low-entropy feature grids, which can be exported as a zip-compressed archive [S1].

**Data Structures:**
The method operates on the multi-resolution hash encoding feature grids $\Theta$ defined in the parent architecture. The grids consist of $L$ levels. At each level $l \in \{0, \dots, L-1\}$, there is a hash table of size $T$. Each entry in the hash table stores $F$ features [S1]. The standard INGP parameters are:
*   $L = 16$ (Number of levels)
*   $T = 2^{19}$ (Hash table size)
*   $F = 2$ (Number of features per level)
*   $N_{min} = 2$ (Coarsest resolution)
*   $N_{max} = 16$ (Finest resolution)
*   Training iterations: 30,000 [S1]

**Parameters:**
*   $\Delta$: Quantization step. Set to the maximum value that does not affect PSNR performance when $\lambda=0$ (e.g., $\Delta = 2048$ in experiments) [S1].
*   $\lambda$: Fixed rate-distortion loss trade-off parameter. Typical range: $\{10^{-4}, \dots, 5 \times 10^{-3}\}$ [S1].
*   $\bar{\lambda}$: Adaptive rate loss trade-off coefficient. Typical range: $\{0.1, \dots, 10\}$ [S1].
*   $\Lambda$: Hybrid rate loss trade-off threshold. Typical values: $0.0009$ (Lego scene), $0.005$ (Breakdance scene) [S1].
*   $\mu_{init}$: Initial value for distribution parameter $\mu$. Set to $0$ [S1].
*   $b_{init}$: Initial value for distribution parameter $b$. Set to $0.01$ [S1].

**Algorithm Steps:**
The method prepares the system for two concurrent tasks: quantization-aware learning and entropy-aware learning [S1].

1.  **Initialization:**
    *   Initialize feature grids $\Theta$ as in INGP.
    *   Initialize distribution parameters $\mu = \mu_{init} = 0$ and $b = b_{init} = 0.01$ [S1].

2.  **Training Loop (per batch of rays $\mathcal{R}$):**
    a.  **Forward Pass with Quantization Noise:**
        For each ray, interpolate features $y$ from the hash grids. Add uniform quantization noise $\Delta \sim U[-\Delta/2, \Delta/2]$ to obtain noisy features $\hat{y}$:
        $$ \hat{y} = y + \Delta $$
        [S1]
        
        Compute volume density $\sigma$ using the density MLP ($MLP_{\Phi_1}$):
        $$ \sigma = MLP_{\Phi_1}(\hat{y}) $$
        [S1]
        
        Render the pixel color $\hat{C}(\mathbf{r})$ and compute the RGB rendering loss $L_{rgb}$ against the ground truth $C(\mathbf{r})$:
        $$ L_{rgb} = \sum_{\mathbf{r} \in \mathcal{R}} ||\hat{C}(\mathbf{r}) - C(\mathbf{r})||^2 $$
        [S1]

    b.  **Entropy-Aware Rate Loss Calculation:**
        Approximate the distribution of the feature grids $\Theta$ using a cumulative distribution function (CDF) $P(\theta)$ with parameters $\mu$ and $b$. The choice of distribution determines the quantizer and the loss formulation [S1].
        
        *   **Option 1: Laplace Distribution**
            *   Parameters: $\mu$ is the mean, $b$ is the standard deviation.
            *   Quantizer: Non-centered (mid-rise) quantizer.
                $$ Q_{mid-rise}(\theta_{l,\tau,f}) = \Delta \left[ \lceil \frac{\theta_{l,\tau,f}}{\Delta} \rceil - \frac{1}{2} \right] $$
                [S1]
            *   Rate Loss $R(\Theta)$:
                $$ R(\Theta) = -\frac{1}{LF} \sum_{\hat{\theta}_{l,\tau,f} \in \Theta} \log_2 \left[ P\left(\hat{\theta}_{l,\tau,f} + \frac{\Delta}{2}\right) - P\left(\hat{\theta}_{l,\tau,f} - \frac{\Delta}{2}\right) \right] $$
                [S1]
            *   Global Loss: Use adaptive coefficient $\bar{\lambda}$.
                $$ L_{global} = L_{rgb} (1 + \bar{\lambda} R(\Theta)) $$
                [S1]
            *   Adaptation: $\bar{\lambda}$ changes throughout training based on current PSNR to linearize the relation between compressed size and PSNR [S1].

        *   **Option 2: Cauchy Distribution**
            *   Parameters: $\mu$ is the median, $b$ is the median absolute deviation (MAD).
            *   Quantizer: Centered (mid-tread) quantizer.
                $$ Q_{mid-tread}(\theta_{l,\tau,f}) = \Delta \left\lfloor \frac{\theta_{l,\tau,f}}{\Delta} + \frac{1}{2} \right\rfloor $$
                [S1]
            *   Rate Loss $R(\Theta)$: Same formula as above, using Cauchy CDF [S1].
            *   Global Loss: Use hybrid coefficient $\lambda$.
                $$ L_{global} = L_{rgb} + \lambda R(\Theta) $$
                [S1]
            *   Adaptation: $\lambda$ is fixed for high compression, but uses a hybrid scheme for lower compression to prioritize PSNR:
                $$ \lambda = \begin{cases} \lfloor \lambda \rceil L_{rgb} & \text{if } \lambda < \Lambda < L_{rgb} \\ L_{rgb} & \text{if } \Lambda \le \lambda < L_{rgb} \\ \lambda & \text{otherwise} \end{cases} $$
                [S1]

    c.  **Backpropagation:**
        Update model parameters (feature grids and MLP weights) and distribution parameters ($\mu, b$) using gradients of $L_{global}$ [S1].

3.  **Export:**
    After training, quantize the learned feature grids $\Theta$ using the selected quantizer (mid-rise for Laplace, mid-tread for Cauchy) [S1]. Apply a lossless compression tool (e.g., 7zip) on the quantized feature grids [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.stats import laplace, cauchy

def compute_laplace_cdf(theta, mu, b):
    """Compute CDF of Laplace distribution."""
    return laplace.cdf(theta, loc=mu, scale=b)

def compute_cauchy_cdf(theta, mu, b):
    """Compute CDF of Cauchy distribution."""
    return cauchy.cdf(theta, loc=mu, scale=b)

def quantize_mid_rise(theta, delta):
    """Non-centered (mid-rise) quantizer for Laplace."""
    return delta * (np.ceil(theta / delta) - 0.5)

def quantize_mid_tread(theta, delta):
    """Centered (mid-tread) quantizer for Cauchy."""
    return delta * np.floor(theta / delta + 0.5)

def compute_rate_loss(features, quantized_features, cdf_func, delta):
    """
    Compute rate loss R(Theta) as the negative log probability 
    of quantized features over one quantization interval.
    """
    # P(theta_hat + delta/2) - P(theta_hat - delta/2)
    prob_upper = cdf_func(quantized_features + delta / 2)
    prob_lower = cdf_func(quantized_features - delta / 2)
    
    # Avoid log(0)
    prob = np.clip(prob_upper - prob_lower, 1e-10, 1.0)
    
    rate = -np.mean(np.log2(prob))
    return rate

def train_cawa_nerf(scene_data, distribution='laplace', delta=2048, 
                    mu_init=0, b_init=0.01, lambda_fixed=1e-3, 
                    lambda_adaptive_init=1.0, lambda_threshold=0.001):
    """
    CAwa-NeRF Training Loop.
    
    Args:
        scene_data: Object providing rays and ground truth colors.
        distribution: 'laplace' or 'cauchy'.
        delta: Quantization step.
        mu_init, b_init: Initial distribution parameters.
        lambda_fixed: Fixed lambda for Cauchy or initial adaptive lambda.
        lambda_adaptive_init: Initial value for adaptive lambda (Laplace).
        lambda_threshold: Hybrid threshold Lambda (Cauchy).
        
    Returns:
        Trained feature grids.
    """
    # Initialize feature grids Theta as in INGP
    Theta = initialize_ingo_features()
    
    # Initialize distribution parameters
    mu = mu_init
    b = b_init
    
    # Select CDF and Quantizer functions
    if distribution == 'laplace':
        cdf_func = compute_laplace_cdf
        quantize_func = quantize_mid_rise
        use_adaptive = True
    else: # cauchy
        cdf_func = compute_cauchy_cdf
        quantize_func = quantize_mid_tread
        use_adaptive = False
        
    lambda_current = lambda_fixed if not use_adaptive else lambda_adaptive_init

    for iteration in range(30000):
        # 1. Sample batch of rays
        rays, gt_colors = scene_data.sample_batch()
        
        # 2. Forward Pass with Quantization Noise
        # Interpolate features y from Theta based on ray positions
        y = interpolate_features(Theta, rays)
        
        # Add uniform noise
        noise = np.random.uniform(-delta/2, delta/2, size=y.shape)
        y_noisy = y + noise
        
        # Compute density and color
        sigma = density_mlp(y_noisy)
        color_pred = color_mlp(y_noisy, sigma, rays)
        
        # Compute RGB Loss
        l_rgb = np.mean((color_pred - gt_colors) ** 2)
        
        # 3. Compute Rate Loss
        # Quantize features for rate calculation (using current Theta, not noisy)
        theta_quantized = quantize_func(Theta, delta)
        
        # Compute rate loss
        r_theta = compute_rate_loss(Theta, theta_quantized, cdf_func, delta)
        
        # 4. Compute Global Loss
        if use_adaptive:
            # Laplace: L_global = L_rgb * (1 + lambda_bar * R(Theta))
            # lambda_bar is updated based on PSNR (simplified here as fixed for structure)
            l_global = l_rgb * (1 + lambda_current * r_theta)
        else:
            # Cauchy: Hybrid lambda
            if lambda_current < lambda_threshold < l_rgb:
                lambda_hybrid = np.round(lambda_current) * l_rgb
            elif lambda_threshold <= lambda_current < l_rgb:
                lambda_hybrid = l_rgb
            else:
                lambda_hybrid = lambda_current
            
            l_global = l_rgb + lambda_hybrid * r_theta
            
        # 5. Backpropagation
        # Update Theta, MLP weights, mu, b using gradients of l_global
        gradients = compute_gradients(l_global, Theta, mu, b)
        update_parameters(Theta, mu, b, gradients)
        
        # Update lambda_current for Laplace based on current PSNR (implementation detail)
        if use_adaptive:
            current_psnr = compute_psnr(gt_colors, color_pred)
            lambda_current = update_adaptive_lambda(lambda_current, current_psnr)

    return Theta

def export_compressed(Theta, distribution='laplace', delta=2048):
    """
    Export feature grids as zip-compressed.
    """
    if distribution == 'laplace':
        quantized = quantize_mid_rise(Theta, delta)
    else:
        quantized = quantize_mid_tread(Theta, delta)
        
    # Apply lossless compression (e.g., 7zip)
    # Note: Actual file I/O is outside the scope of the algorithm definition
    save_as_zip(quantized)
```

**Edge Cases and Pitfalls:**
*   **Laplace Distribution:** Causes aggressive PSNR degradation when decreasing size beyond the minimal rate loss trade-off, breaking the semi-linear PSNR vs. size relation [S1]. Requires adaptive $\bar{\lambda}$ to maintain quality [S1].
*   **Cauchy Distribution:** With fixed $\lambda$, fails to achieve high PSNR for minimal compression (small $\lambda$ values) [S1]. Requires hybrid $\lambda$ at the start to prioritize PSNR [S1].
*   **Initialization:** Features are initialized in $[-10^{-4}, 10^{-4}]$. $\mu_{init}=0$ and $b_{init}=0.01$ are chosen to match this narrow range and model quantization noise [S1].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to INGP, with negligible overhead for the entropy-aware terms [S1]. Exporting compressed grids takes only a few seconds [S1].
*   **Memory Complexity:** The uncompressed feature grids have the same size as INGP. The compressed size varies based on the rate-distortion trade-off [S1].
*   **Measured Performance:**
    *   **Lego Scene (Synthetic):** Compresses from 21.9 MB to 1.2 MB without PSNR loss (32.96 dB to 33.14 dB) or to 0.6 MB with 0.59 dB loss (32.37 dB) [S1]. Another experiment notes compression from 6.4 MB to 1.1 MB with minimal rate loss trade-off [S1].
    *   **Single Object Masked Background:** Compresses down to 6% (1.2 MB) of original size without PSNR loss (33 dB) or to 2.4% (0.53 MB) with slight virtual loss (32.31 dB) [S1].
    *   **Breakdance Scene (Real-life):** Compresses from 21.9 MB (INGP) to 6.4 MB without quality loss, or to 1.3 MB (Laplace) / 1.1 MB (Cauchy) with 31.4 dB PSNR [S1].
*   **Strengths:**
    *   Achieves significant compression (up to ~90% reduction) with negligible training overhead [S1].
    *   Does not change the storage architecture or parameters of the base INGP method [S1].
    *   Cauchy distribution favors PSNR over entropy minimization due to higher weights on tails [S1].
*   **Limitations:**
    *   Laplace distribution can aggressively degrade PSNR at high compression levels [S1].
    *   Requires careful tuning of $\lambda$ or $\bar{\lambda}$ depending on the distribution and scene complexity [S1].

## Background
CAwa-NeRF was introduced in 2024 by authors publishing in the *2024 IEEE 15th International Conference on Sensing Technology (ICST)* [S1]. It addresses the high storage requirements of NeRFs, which limit their use in streaming, network, and cloud applications [S1]. The method is inspired by COOLCHIC [S1] and adapts entropy-aware training to the INGP architecture [S1]. It is designed to be applicable to any NeRF explicit abstract scene representation method, regardless of the architecture of the stored features [S1].

## Sources
[S1] CAwa-NeRF: Instant Learning of Compression-Aware NeRF Features, https://doi.org/10.1109/sds60720.2024.00015, fetched 2026-10-26
