# Mip-NeRF Optimization and Activation Details

Also called: JaxNeRF Training Procedure, Mip-NeRF Misc. Changes

## Overview
This article specifies the implementation details for training mip-NeRF and JaxNeRF, focusing on activation functions, input encoding strategies, and learning rate scheduling [S1]. These modifications are designed to improve training stability and robustness, particularly against catastrophic failure modes, without significantly altering the method's accuracy or computational speed [S1]. Unlike the standard NeRF training configuration, mip-NeRF removes the identity concatenation from the MLP input and employs shifted softplus and widened sigmoid activations to handle the multiscale nature of the data and prevent optimization stagnation [S1].

## How it works
The method defines the specific neural network inputs, activation functions, and optimizer parameters used to train the mip-NeRF model. It builds on the general radiance-field optimization framework [Parent](6267-method-specific-training-configurations.md).

### Inputs and Outputs
*   **Input**: Integrated Positional Encoding (IPE) features derived from the mean and covariance of the conical frustum. The original position and view direction are **not** concatenated with the encoding [S1].
*   **Output**: Density $\tau$ and color $c$ (RGB).

### Data Structures
The MLP architecture processes the IPE features. The view direction is handled separately and injected at the final stage.

### Steps
1.  **Input Encoding**: Compute the Integrated Positional Encoding (IPE) for the spatial position and view direction. Do not concatenate the raw position or view direction vectors with the encoded features [S1].
2.  **Density Prediction**: Pass the IPE features through the MLP layers to predict the pre-activation value for density. Apply the shifted softplus activation to obtain $\tau$ [S1].
3.  **View Direction Injection**: Normalize the ray direction $d$. Apply positional encoding with $L=4$ frequencies. Inject this encoded view direction into the last layer of the MLP [S1].
4.  **Color Prediction**: After the view direction injection, predict the pre-activation value for color. Apply the widened sigmoid activation to obtain $c$ [S1].

### Formulas
**Density Activation (Shifted Softplus)** [S1]:
$$ \tau = \log(1 + \exp(x - 1)) $$
where $x$ is the MLP output for density. The shift by $-1$ is equivalent to initializing the bias to $-1$, resulting in small initial density values [S1].

**Color Activation (Widened Sigmoid)** [S1]:
$$ c = \frac{1 + 2\epsilon}{1 + \exp(-x)} - \epsilon $$
where $x$ is the MLP output for color and $\epsilon = 0.001$. This function saturates slightly outside the $[0, 1]$ range [S1].

**Learning Rate Schedule** [S1]:
$$ \eta_i = \left( \lambda_w + (1 - \lambda_w) \sin\left(\frac{\pi}{2} \operatorname{clip}\left(\frac{i}{n_w}, 0, 1\right)\right) \right) \times \left( \exp\left( (1 - \frac{i}{n}) \log(\eta_0) + \frac{i}{n} \log(\eta_n) \right) \right) $$
where:
*   $i$ is the current iteration index.
*   $n$ is the total number of iterations.
*   $n_w$ is the number of warmup iterations.
*   $\eta_0$ is the initial learning rate.
*   $\eta_n$ is the final learning rate.
*   $\lambda_w$ is a scale factor annealed from $0.01$ to $1$ during the warmup period [S1].

### Parameters
*   **Optimizer**: Adam [S1].
*   **Batch Size**: 4096 [S1].
*   **Total Iterations ($n$)**: 1,000,000 [S1].
*   **Warmup Iterations ($n_w$)**: 2,500 [S1].
*   **Initial Learning Rate ($\eta_0$)**: $5 \cdot 10^{-4}$ [S1].
*   **Final Learning Rate ($\eta_n$)**: $5 \cdot 10^{-6}$ [S1]
*   **View Direction PE Frequencies ($L$)**: 4 [S1].

### Edge Cases and Pitfalls
*   **Catastrophic Failure**: Using standard ReLU for density can cause the MLP to emit negative values everywhere, resulting in zero gradients and optimization failure. The shifted softplus prevents this [S1].
*   **Saturation Failure**: Standard sigmoid saturation can cause optimization failure when the network attempts to explain pure black or white pixels. The widened sigmoid avoids this by allowing slight saturation outside $[0, 1]$ [S1].
*   **Identity Concatenation**: Concatenating raw position/view direction with IPE is not used as it complicates the formalization of IPE features and does not meaningfully improve performance [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def shifted_softplus(x):
    """
    Density activation function.
    Equivalent to log(1 + exp(x - 1)).
    """
    return np.log(1 + np.exp(x - 1))

def widened_sigmoid(x, epsilon=0.001):
    """
    Color activation function.
    Saturates slightly outside [0, 1] to avoid optimization failure on black/white pixels.
    """
    return (1 + 2 * epsilon) / (1 + np.exp(-x)) - epsilon

def positional_encoding(x, L=4):
    """
    Standard positional encoding with L frequencies.
    Assumes x is a vector of shape (D,) or (N, D).
    Returns concatenated sin/cos terms for frequencies 0 to L-1.
    """
    # Implementation detail: Standard PE function
    # For a vector x of dimension D:
    # pe = [sin(2^0 * pi * x), cos(2^0 * pi * x), ..., sin(2^(L-1) * pi * x), cos(2^(L-1) * pi * x)]
    # This is a placeholder for the specific PE implementation used in JaxNeRF/mip-NeRF.
    # The exact implementation involves scaling frequencies by powers of 2.
    pass 

def compute_learning_rate(i, n, n_w, eta_0, eta_n):
    """
    Computes the learning rate at iteration i with warmup.
    """
    # Anneal lambda_w from 0.01 to 1 during warmup
    # clip(i/n_w, 0, 1) ensures we stay within [0, 1]
    progress = min(i / n_w, 1.0)
    lambda_w = 0.01 + (1 - 0.01) * math.sin(math.pi / 2 * progress)
    
    # Logarithmic annealing for the base learning rate
    log_eta = (1 - i / n) * math.log(eta_0) + (i / n) * math.log(eta_n)
    base_lr = math.exp(log_eta)
    
    return lambda_w * base_lr

def mip_nerf_forward(mlp, position, view_direction):
    """
    Forward pass for mip-NeRF with specific activation and input details.
    
    Args:
        mlp: The neural network model.
        position: 3D position vector.
        view_direction: 3D view direction vector.
        
    Returns:
        tau: Density value.
        c: Color vector (RGB).
    """
    # 1. Input Encoding: Use IPE only, no identity concatenation
    # IPE features are computed from the mean and covariance of the conical frustum
    # This step is assumed to be handled by the data loader or pre-processing
    # IPE_features = compute_ipe_features(position, covariance) 
    # For the purpose of this pseudocode, we assume `ipe_features` is the input to the MLP
    # Note: The actual IPE computation is complex and specific to the Gaussian integration.
    # Here we assume `mlp` takes `ipe_features` as input.
    
    # 2. Density Prediction
    # Pass IPE features through MLP layers (excluding the last layer for density)
    # Let x_tau be the output of the MLP before the final activation for density
    x_tau = mlp.predict_density(ipe_features) 
    tau = shifted_softplus(x_tau)
    
    # 3. View Direction Injection
    # Normalize ray direction
    d_normalized = view_direction / np.linalg.norm(view_direction)
    
    # Apply positional encoding with L=4
    view_pe = positional_encoding(d_normalized, L=4)
    
    # 4. Color Prediction
    # Inject view_pe into the last layer of the MLP
    # Let x_c be the output of the MLP before the final activation for color
    # This injection happens after tau is predicted but before c is predicted
    x_c = mlp.predict_color(ipe_features, view_pe)
    c = widened_sigmoid(x_c)
    
    return tau, c
```

## Subtopics

### [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md)
This article specifies the training configuration, network architecture, and loss functions for the Mip-NeRF 360 method, which extends mip-NeRF to unbounded, "in-the-wild" scenes. It solves the problem of training stability and rendering quality in complex environments with large depth ranges and varying lighting by introducing a two-stage proposal sampling strategy and a specific regularization scheme. It differs from standard NeRF training by using a proposal network to guide sample placement and from basic mip-NeRF by employing a Charbonnier reconstruction loss and a log-linear learning rate schedule to handle the scale and complexity of outdoor/indoor hybrid scenes.

### [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md)
Scale-Dependent Downweighting is a technique to anti-alias trilinear interpolation in neural radiance fields by downweighting features interpolated from grid cells that are small relative to the sampling Gaussian's spread, preventing aliasing artifacts from high-frequency content. Unlike naive trilinear interpolation which causes aliasing at high frequencies, this method uses the Gaussian CDF to weight samples, similar to Mip-NeRF's IPE features but applied to grid-based representations.

### Alternative Gaussian Positional Encoding
An alternative approach explored for featurizing the mean and covariance matrix of the multivariate Gaussians used by mip-NeRF is to apply positional encoding to the mean and the (signed) square root of the elements of the covariance matrix, and use the concatenation of the two as input [S1]. Specifically, positional encoding is computed for the mean $\mu$ with $L=12$, and for the vectorized upper triangular part of the signed covariance matrix with $L=2$ [S1]. This approach performs comparably to IPE features but is sensitive to hyperparameters $L$ and covariance parameterization, whereas IPE is hyperparameter-free and compact [S1]. Additionally, while IPE features are insensitive to the off-diagonal elements of $\Sigma$ (correlation), experiments show that modeling this correlation does not improve accuracy, suggesting it is not a helpful cue [S1].

## Operating characteristics
*   **Performance**: Mip-NeRF outperforms brute-force supersampling (16 rays/pixel) in quality, is 13x faster to train, and 16x faster to evaluate [S1].
*   **Stability**: The shifted softplus and widened sigmoid activations improve training stability, particularly with large learning rates, preventing catastrophic failure modes [S1].
*   **Complexity**: The method uses a single unified multiscale model, reducing aliasing and simplifying the architecture compared to NeRF's separate per-scale models [S1].
*   **Limitations**: The alternative Gaussian positional encoding (concatenation) is sensitive to hyperparameters and does not offer significant advantages over IPE [S1].

## Background
The details described in this article originate from the Mip-NeRF paper, which introduced a multiscale representation for anti-aliasing neural radiance fields [S1]. The "miscellaneous" changes, including activation functions and input encoding modifications, were included to improve robustness and simplify the method's formalization [S1]. These details are also part of the JaxNeRF codebase training procedure [S1].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
