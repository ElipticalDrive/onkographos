# Mip-NeRF Optimization and Activation Details

Also called: JaxNeRF Training Procedure, Mip-NeRF Misc. Changes

## Overview
This article defines the specific implementation details for training Mip-NeRF and JaxNeRF, focusing on activation functions, input encoding strategies, and learning rate schedules. These modifications are designed to improve training stability and robustness, particularly when using large learning rates or handling edge cases like black/white pixels, without significantly altering the model's accuracy or speed [S1]. Unlike standard NeRF implementations that concatenate identity inputs, Mip-NeRF uses Integrated Positional Encoding (IPE) exclusively, and employs shifted softplus and widened sigmoid activations to prevent optimization failures [S1].

## How it works
### Inputs and Outputs
*   **Inputs:** Training images (RGB), camera poses (intrinsics and extrinsics), and ray origins/directions.
*   **Outputs:** Rendered pixel colors $\hat{C}(\mathbf{r})$ and their gradients for backpropagation.

### Data Structures
*   **Integrated Positional Encoding (IPE):** The input to the MLP is the IPE feature vector only. The original position $\mathbf{x}$ and view direction $\mathbf{d}$ are **not** concatenated with the encoding [S1].
*   **View Direction Injection:** The view direction $\mathbf{d}$ is processed separately and injected into the last layer of the MLP [S1].

### Numbered Steps
1.  **Ray Sampling:** Generate rays from training images. For Mip-NeRF, these rays are represented as cones (or frustums) to account for pixel footprint [S1].
2.  **Point Sampling:** Sample points along the rays. Mip-NeRF uses stratified sampling within the conical frustum [S1].
3.  **Feature Encoding:**
    *   Compute the Integrated Positional Encoding (IPE) for the sampled points. This involves integrating the standard positional encoding over the Gaussian distribution defined by the ray's mean and covariance [S1].
    *   Normalize the ray direction $\mathbf{d}$.
    *   Apply positional encoding to $\mathbf{d}$ with $L=4$ frequencies [S1].
4.  **Network Forward Pass:**
    *   Pass the IPE features through the MLP layers.
    *   Predict the density parameter $\tau$ from the intermediate MLP output.
    *   Apply the **Density Activation** to $\tau$ to get the final density $\sigma$ (or use $\tau$ directly if the rendering equation is adapted, but typically $\tau$ is transformed) [S1]. *Note: The source describes $\tau$ as the output of the activation function, which is then used in the rendering equation. Specifically, the activation produces the value used for density.*
    *   Concatenate the encoded view direction to the features **after** $\tau$ is predicted but **before** the color $\mathbf{c}$ is predicted [S1].
    *   Predict the color $\mathbf{c}$ from the final MLP layer.
    *   Apply the **Color Activation** to $\mathbf{c}$ [S1].
5.  **Volume Rendering:** Compute the accumulated color $\hat{C}(\mathbf{r})$ using the differentiable volume rendering integral [S1]:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i$ is the distance between samples [S1].
6.  **Loss Computation:** Calculate the photometric loss (MSE) between $\hat{C}(\mathbf{r})$ and the ground truth pixel color [S1].
7.  **Optimization:** Update parameters using the Adam optimizer with the specific learning rate schedule described below [S1].

### Formulas
**Density Activation:**
$$ \tau = \log(1 + \exp(x - 1)) $$
where $x$ is the raw output of the MLP layer predicting density. This is a shifted softplus function [S1].

**Color Activation:**
$$ c = \frac{1 + 2\epsilon}{1 + \exp(-x)} - \epsilon $$
where $x$ is the raw output of the MLP layer predicting color, and $\epsilon = 0.001$. This is a "widened" sigmoid [S1].

**Learning Rate Schedule:**
$$ \eta_i = \left(\lambda_w + (1 - \lambda_w) \sin\left(\frac{\pi}{2} \operatorname{clip}\left(\frac{i}{n_w}, 0, 1\right)\right)\right) \times \left(\exp\left((1 - \frac{i}{n}) \log(\eta_0) + \frac{i}{n} \log(\eta_n)\right)\right) $$
where:
*   $i$ is the current iteration.
*   $n$ is the total number of iterations.
*   $n_w$ is the number of warmup iterations.
*   $\eta_0$ is the initial learning rate.
*   $\eta_n$ is the final learning rate.
*   $\lambda_w$ is a scale factor annealed from 0.01 to 1 during warmup [S1].

### Parameters
*   **Optimizer:** Adam [S1].
*   **Total Iterations ($n$):** 1,000,000 [S1].
*   **Batch Size:** 4096 [S1].
*   **Initial Learning Rate ($\eta_0$):** $5 \cdot 10^{-4}$ [S1].
*   **Final Learning Rate ($\eta_n$):** $5 \cdot 10^{-6}$ [S1].
*   **Warmup Iterations ($n_w$):** 2500 [S1].
*   **Warmup Start Factor ($\lambda_{w, start}$):** 0.01 [S1].
*   **Warmup End Factor ($\lambda_{w, end}$):** 1 [S1].
*   **View Direction PE Frequencies ($L$):** 4 [S1].
*   **Color Activation Epsilon ($\epsilon$):** 0.001 [S1].

### Edge Cases and Pitfalls
*   **Negative Density:** Using standard ReLU for density can lead to catastrophic failure if the MLP outputs negative values everywhere, resulting in zero gradients. The shifted softplus prevents this [S1].
*   **Black/White Pixel Saturation:** Standard sigmoid saturation can cause optimization failure when trying to explain extreme pixel values. The widened sigmoid avoids this by saturating slightly outside [0, 1] [S1].
*   **Identity Concatenation:** Concatenating original position/view direction with IPE is unnecessary and makes formalization harder. Removing it simplifies the method without performance loss [S1].
*   **Correlation Insensitivity:** IPE features are insensitive to off-diagonal elements of the covariance matrix $\Sigma$ (correlation). Experiments show that modeling this correlation does not improve accuracy, so it is ignored [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def shifted_softplus(x):
    """
    Density activation function.
    log(1 + exp(x - 1))
    """
    return np.log(1 + np.exp(x - 1))

def widened_sigmoid(x, epsilon=0.001):
    """
    Color activation function.
    (1 + 2*epsilon) / (1 + exp(-x)) - epsilon
    """
    return (1 + 2 * epsilon) / (1 + np.exp(-x)) - epsilon

def learning_rate_schedule(i, n, n_w, eta_0, eta_n, lambda_w_start=0.01, lambda_w_end=1.0):
    """
    Calculates the learning rate at iteration i.
    """
    # Clip i/n_w to [0, 1]
    clip_val = np.clip(i / n_w, 0, 1)
    
    # Calculate lambda_w for current iteration
    # Annealed from lambda_w_start to lambda_w_end
    lambda_w = lambda_w_start + (lambda_w_end - lambda_w_start) * np.sin(np.pi / 2 * clip_val)
    
    # Calculate logarithmic annealing factor
    log_anneal = np.exp((1 - i / n) * np.log(eta_0) + (i / n) * np.log(eta_n))
    
    return lambda_w * log_anneal

def mip_nerf_forward_pass(ipe_features, view_direction, mlp):
    """
    Simulates the forward pass of Mip-NeRF MLP.
    
    Args:
        ipe_features: Integrated Positional Encoding features.
        view_direction: Normalized ray direction vector.
        mlp: The neural network model.
        
    Returns:
        density: Predicted density.
        color: Predicted color.
    """
    # Step 1: Predict density from IPE features
    # The MLP processes IPE features through hidden layers
    # Let's assume mlp.predict_density(x) returns the raw output for density
    raw_density = mlp.predict_density(ipe_features)
    
    # Apply density activation
    density = shifted_softplus(raw_density)
    
    # Step 2: Encode view direction
    # Positional encoding with L=4
    encoded_view_dir = positional_encoding(view_direction, L=4)
    
    # Step 3: Predict color
    # Inject encoded view direction into the last layer
    # This is typically done by concatenating or adding to the features before the final layer
    # For simplicity, we assume the MLP has a method that takes features and view_dir
    raw_color = mlp.predict_color(ipe_features, encoded_view_dir)
    
    # Apply color activation
    color = widened_sigmoid(raw_color)
    
    return density, color

def positional_encoding(v, L=4):
    """
    Standard positional encoding for a vector v with L frequencies.
    Returns a concatenated vector of sin(2^i * pi * v) and cos(2^i * pi * v) for i in 0..L-1.
    """
    freq_bands = 2.0 ** np.arange(L)
    scaled_v = v * freq_bands * np.pi
    
    sin_enc = np.sin(scaled_v)
    cos_enc = np.cos(scaled_v)
    
    return np.concatenate([sin_enc, cos_enc])
```

## Subtopics

### [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md)
This article defines the specific optimization configuration, network architectures, and loss functions used to train the Mip-NeRF 360 model. It solves the problem of stabilizing the training of unbounded, anti-aliased radiance fields by combining a multi-stage proposal sampling strategy with a robust reconstruction loss. It sets itself apart from standard NeRF training by using a two-stage proposal network to guide sampling and by employing a Charbonnier loss for reconstruction stability.

### [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md)
Scale-Dependent Downweighting is a technique to anti-alias trilinear interpolation in neural radiance fields by downweighting features interpolated from grid cells that are small relative to the sampling Gaussian's spread, preventing aliasing artifacts from high-frequency content. Unlike naive trilinear interpolation which causes aliasing at high frequencies, this method uses the Gaussian CDF to weight samples, similar to Mip-NeRF's IPE features but applied to grid-based representations.

### Alternative Gaussian Positional Encoding
An alternative approach explored during Mip-NeRF development involves concatenating the positional encodings of the mean and the (signed) square root of the covariance matrix elements as input to the MLP. Specifically, positional encoding with $L=12$ is applied to the mean $\mu$, and positional encoding with $L=2$ is applied to the vectorized upper triangular part of the signed covariance matrix. This approach performs comparably to IPE features but is sensitive to hyperparameters $L$ and covariance parameterization, whereas IPE is hyperparameter-free and compact [S1].

### Reflected Cone Derivation
A geometric derivation used in Mip-NeRF-Casting to shift the origin of a reflected ray so that its cone radius matches the incident ray's cone radius at the intersection point. This ensures consistent view-dependent appearance for reflections. The condition for equating radii is $\dot{r}\|\bar{\mathbf{x}} - \mathbf{o}\| = \dot{r}'\|\bar{\mathbf{x}} - \mathbf{o}'\|$, where $\dot{r}$ is the incident ray radius parameter, $\bar{\mathbf{x}}$ is the intersection point, $\mathbf{o}$ is the incident ray origin, $\dot{r}'$ is the reflected ray radius parameter, and $\mathbf{o}'$ is the reflected ray origin [S2].

### RawNeRF
A method for High Dynamic Range (HDR) image view synthesis and denoising that adapts Mip-NeRF to use raw linear images as training data. It renders in a linear color space and supervises with variable exposure images, scaling the NeRF models’ “exposure” by the training image’s shutter speed, as well as a per-channel learned scaling factor. Unlike HDR-NeRF which uses LDR images, RawNeRF allows varying exposure and tone-mapping curves to be applied after NeRF rendering [S3].

## Operating characteristics
*   **Training Time:** Mip-NeRF is 13x faster to train than a brute-force supersampling baseline (16 rays/pixel) [S1].
*   **Evaluation Time:** Mip-NeRF is 16x faster to evaluate than the supersampling baseline [S1].
*   **Quality:** Mip-NeRF outperforms the 16x supersampling baseline in quality metrics (PSNR, SSIM, LPIPS) [S1].
*   **Stability:** The activation functions and learning rate schedule improve training stability, especially with large learning rates [S1].
*   **Memory:** Not given in the sources.
*   **Scalability:** Not given in the sources.
*   **Strengths:** Anti-aliasing, fast training/evaluation compared to supersampling, stable optimization.
*   **Limitations:** IPE features are insensitive to correlation in the covariance matrix (though this was found not to hurt performance) [S1].

## Background
Mip-NeRF was introduced by Barron et al. in 2021 [S1]. The optimization details described here are from the supplementary material of the Mip-NeRF paper, specifically section E. The method builds on the original NeRF framework [S1]. RawNeRF, a variant using raw images, was created by Mildenhall et al. in November 2021 [S3]. NeRF-Casting, which includes the reflected cone derivation, was published in 2024 [S2].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S2] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
