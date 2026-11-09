# Mip-NeRF Optimization and Activation Details

Also called: JaxNeRF Training Procedure, Mip-NeRF Misc. Changes

## Overview
This article defines the specific implementation details for training the Mip-NeRF model, focusing on activation functions, input encoding strategies, and learning rate scheduling. These modifications are designed to improve training stability and robustness, particularly against catastrophic failure modes, without significantly altering the model's accuracy or computational speed compared to the baseline NeRF architecture [S1]. It sets itself apart from standard NeRF training by replacing the identity concatenation with pure Integrated Positional Encoding (IPE), using shifted softplus and widened sigmoid activations, and employing a specific warmup learning rate schedule [S1].

## How it works
The method modifies the standard NeRF training loop [Parent](6267-method-specific-training-configurations.md) through four key implementation changes: input encoding, activation functions, view direction injection, and optimization scheduling.

### Inputs and Outputs
*   **Inputs**: Integrated Positional Encoding (IPE) features derived from the mean and covariance of the conical frustum sampling distribution.
*   **Outputs**: Density $\tau$ and color $c$ for each sample point.

### Data Structures and Layout
The MLP input is strictly the IPE features. The original NeRF practice of concatenating the raw position and view direction with their positional encodings (identity encoding) is removed [S1].

### Step-by-Step Procedure

1.  **Input Encoding**:
    *   Compute the Integrated Positional Encoding (IPE) for the spatial mean and covariance of the sampling Gaussian.
    *   Do **not** concatenate the raw position or view direction vectors to this input. The MLP input is solely the IPE features [S1].

2.  **Density Prediction ($\tau$)**:
    *   Pass the IPE features through the MLP layers up to the density head.
    *   Apply the **shifted softplus** activation to the raw output $x$:
        $$ \tau = \log(1 + \exp(x - 1)) $$
    *   This is equivalent to initializing the density bias to $-1$, ensuring initial $\tau$ values are small [S1].

3.  **View Direction Injection**:
    *   Normalize the ray direction $d$.
    *   Apply positional encoding to $d$ with $L=4$ frequencies [S1].
    *   Inject these encoded view directions into the **last layer** of the MLP.
    *   This injection occurs **after** $\tau$ is predicted but **before** the color $c$ is predicted [S1].

4.  **Color Prediction ($c$)**:
    *   Apply the **widened sigmoid** activation to the raw output $x$ of the color head:
        $$ c = \frac{1 + 2\epsilon}{1 + \exp(-x)} - \epsilon $$
    *   Where $\epsilon = 0.001$. This allows the output to saturate slightly outside the $[0, 1]$ range [S1].

5.  **Optimization Loop**:
    *   Use the Adam optimizer [S1].
    *   Total iterations: $n = 1,000,000$ [S1].
    *   Batch size: $4096$ [S1].
    *   Apply the learning rate schedule defined below.

### Learning Rate Schedule
The learning rate $\eta_i$ at iteration $i$ combines a warmup phase and a logarithmic annealing phase [S1]:

$$ \eta_i = \left( \lambda_w + (1 - \lambda_w) \sin\left(\frac{\pi}{2} \operatorname{clip}\left(\frac{i}{n_w}, 0, 1\right)\right) \right) \times \left( \exp\left( (1 - \frac{i}{n}) \log(\eta_0) + \frac{i}{n} \log(\eta_n) \right) \right) $$

Where:
*   $n_w = 2500$ is the number of warmup iterations [S1].
*   $\eta_0 = 5 \cdot 10^{-4}$ is the initial learning rate [S1].
*   $\eta_n = 5 \cdot 10^{-6}$ is the final learning rate [S1].
*   $\lambda_w$ is a scale factor annealed from $0.01$ to $1$ during the warmup period $i \in [0, n_w]$ [S1].
*   $\operatorname{clip}(x, 0, 1)$ clamps $x$ to the range $[0, 1]$.

### Parameters
*   **Density Activation**: Shifted softplus with shift $-1$ [S1].
*   **Color Activation**: Widened sigmoid with $\epsilon = 0.001$ [S1].
*   **View Direction PE**: $L=4$ frequencies [S1].
*   **Training Iterations**: $1,000,000$ [S1].
*   **Batch Size**: $4096$ [S1].
*   **Warmup Iterations**: $2500$ [S1].
*   **Initial LR**: $5 \cdot 10^{-4}$ [S1].
*   **Final LR**: $5 \cdot 10^{-6}$ [S1].

### Edge Cases and Pitfalls
*   **Negative Density**: Using standard ReLU for density can lead to catastrophic failure if the MLP outputs negative values everywhere, resulting in zero gradients. The shifted softplus prevents this [S1].
*   **Black/White Pixel Saturation**: Standard sigmoid saturation can cause optimization failure when trying to explain extreme pixel values. The widened sigmoid avoids this by saturating outside $[0, 1]$ [S1].
*   **Correlation Insensitivity**: IPE features are insensitive to off-diagonal elements of the covariance matrix $\Sigma$ (correlation). Experiments indicate that modeling this correlation does not improve accuracy, so it is omitted [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def shifted_softplus(x):
    """
    Density activation: log(1 + exp(x - 1))
    Equivalent to initializing density bias to -1.
    """
    return math.log(1 + math.exp(x - 1))

def widened_sigmoid(x, epsilon=0.001):
    """
    Color activation: (1 + 2*eps) / (1 + exp(-x)) - eps
    """
    return (1 + 2 * epsilon) / (1 + math.exp(-x)) - epsilon

def positional_encoding(x, L=4):
    """
    Standard positional encoding with L frequencies.
    Returns concatenated [sin(2^0 * pi * x), cos(2^0 * pi * x), ..., sin(2^(L-1) * pi * x), cos(2^(L-1) * pi * x)]
    """
    # Implementation detail: standard PE function
    pass

def compute_learning_rate(i, n=1000000, n_w=2500, eta_0=5e-4, eta_n=5e-6, lambda_w_start=0.01, lambda_w_end=1.0):
    """
    Computes the learning rate at iteration i.
    """
    # Warmup factor
    clip_val = min(max(i / n_w, 0.0), 1.0)
    warmup_factor = lambda_w_start + (lambda_w_end - lambda_w_start) * math.sin(math.pi / 2 * clip_val)
    
    # Logarithmic annealing factor
    anneal_factor = math.exp((1 - i / n) * math.log(eta_0) + (i / n) * math.log(eta_n))
    
    return warmup_factor * anneal_factor

def mip_nerf_forward_pass(ipe_features, ray_direction):
    """
    Forward pass for a single sample point.
    
    Args:
        ipe_features: Tensor of Integrated Positional Encoding features.
        ray_direction: Normalized ray direction vector.
        
    Returns:
        tau: Density value.
        c: Color value.
    """
    # 1. Density Prediction
    # MLP layers processing ipe_features
    x_density = mlp_density_head(ipe_features)
    tau = shifted_softplus(x_density)
    
    # 2. View Direction Injection
    # Encode view direction with L=4
    view_pe = positional_encoding(ray_direction, L=4)
    
    # 3. Color Prediction
    # Concatenate view PE to the features before the final color layer
    # Note: The exact concatenation point is "last layer of the MLP"
    # Assuming the MLP structure allows injecting view PE before the final output layer
    x_color = mlp_color_head(ipe_features, view_pe)
    c = widened_sigmoid(x_color)
    
    return tau, c
```

## Subtopics

### [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md)
This variant defines the specific optimization configuration, network architectures, and loss functions used to train the Mip-NeRF 360 model. It solves the problem of stabilizing the training of unbounded, anti-aliased radiance fields by combining a multi-stage proposal sampling strategy with a robust reconstruction loss. It sets itself apart from standard NeRF training by using a two-stage proposal network to guide sampling and by employing a Charbonnier loss for reconstruction stability. See [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md).

### [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md)
Scale-Dependent Downweighting is a technique to anti-alias trilinear interpolation in neural radiance fields by downweighting features interpolated from grid cells that are small relative to the sampling Gaussian's spread, preventing aliasing artifacts from high-frequency content. Unlike naive trilinear interpolation which causes aliasing at high frequencies, this method uses the Gaussian CDF to weight samples, similar to Mip-NeRF's IPE features but applied to grid-based representations. See [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md).

### Alternative Gaussian Positional Encoding
An alternative approach explored during experimentation involves concatenating the positional encodings of the mean and the (signed) square root of the covariance matrix elements as input to the MLP. Specifically, positional encoding is applied to the mean $\mu$ with $L=12$, and to $\operatorname{vec}(\operatorname{triu}(\operatorname{sign}(\Sigma) \circ \rho |\Sigma|))$ with $L=2$. This approach performs comparably to IPE features but is sensitive to hyperparameters $L$ and covariance parameterization, whereas IPE is hyperparameter-free and compact [S1].

## Operating characteristics
*   **Training Speed**: Mip-NeRF is 13x faster to train than a brute-force supersampling baseline (16 rays/pixel) [S1].
*   **Evaluation Speed**: Mip-NeRF is 16x faster to evaluate than the supersampling baseline [S1].
*   **Quality**: Mip-NeRF outperforms the 16x supersampling baseline in quality metrics (PSNR, SSIM, LPIPS) [S1].
*   **Stability**: The modified activation functions and learning rate schedule improve training stability, particularly with large learning rates, though they do not significantly change the final accuracy or speed compared to the baseline NeRF configuration [S1].
*   **Memory**: Uses chunked processing for forward passes to manage GPU memory [Parent](6267-method-specific-training-configurations.md).

## Background
These optimization details were introduced in the Mip-NeRF paper (Barron et al., 2021) to ensure reproducibility and improve the robustness of the training process [S1]. The changes were identified as "miscellaneous" because they do not fundamentally alter the Mip-NeRF method's accuracy or speed but simplify the formalization of IPE features and prevent common optimization failures [S1]. The training procedure follows the JaxNeRF codebase defaults [S1].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
