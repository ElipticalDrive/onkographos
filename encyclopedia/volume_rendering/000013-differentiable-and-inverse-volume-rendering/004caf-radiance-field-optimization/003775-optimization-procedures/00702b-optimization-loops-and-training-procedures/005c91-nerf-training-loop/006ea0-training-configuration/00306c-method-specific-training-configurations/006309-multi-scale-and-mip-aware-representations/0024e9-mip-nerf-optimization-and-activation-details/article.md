# Mip-NeRF Optimization and Activation Details

Also called: JaxNeRF Training Procedure, Mip-NeRF Misc. Changes

## Overview
This article defines the specific hyperparameters, activation functions, and input preprocessing steps used to train Mip-NeRF and JaxNeRF models [S1]. These details address implementation-specific challenges such as training stability, catastrophic failure modes in density prediction, and optimization robustness, rather than introducing new geometric or rendering primitives [S1]. The method sets itself apart from standard NeRF configurations by replacing the identity concatenation in MLP inputs with pure Integrated Positional Encoding (IPE), using shifted softplus and widened sigmoid activations for density and color respectively, and employing a specific warmup learning rate schedule [S1]. It builds on [Method-Specific Training Configurations](6267-method-specific-training-configurations.md) [G].

## How it works
The training procedure minimizes photometric error using the Adam optimizer [S1]. The following details define the network input, activation functions, and optimization schedule [S1].

### Inputs and Outputs
*   **Input:** Integrated Positional Encoding (IPE) of the ray mean and covariance. The original position and view direction are **not** concatenated with the encoding [S1].
*   **View Direction:** The ray direction $d$ is normalized, positionally encoded with $L=4$, and injected into the last layer of the MLP after density $\tau$ is predicted but before color $c$ is predicted [S1].
*   **Output:** Density $\tau$ and color $c$ for each sample point [S1].

### Data Structures
*   **MLP Input:** A vector containing only the IPE features [S1].
*   **MLP Architecture:** Standard MLP as defined in the parent Mip-NeRF method [G]. The view direction encoding is added to the final layer weights/biases or concatenated before the final output layer [S1].

### Activation Functions
Standard ReLU and sigmoid activations are replaced to improve optimization stability [S1].

1.  **Density Activation ($\tau$):**
    A shifted softplus function is used instead of ReLU. This prevents catastrophic failure where the MLP outputs negative values everywhere, resulting in zero gradients for density [S1].
    $$ \tau = \log(1 + \exp(x - 1)) $$
    The shift by $-1$ is equivalent to initializing the biases producing $\tau$ to $-1$, resulting in small initial density values [S1].

2.  **Color Activation ($c$):**
    A "widened" sigmoid is used instead of the standard sigmoid. This function saturates slightly outside the $[0, 1]$ range to avoid optimization failure when explaining black or white pixels, which might otherwise cause the network to saturate activations excessively [S1].
    $$ c = \frac{1 + 2\epsilon}{1 + \exp(-x)} - \epsilon $$
    where $\epsilon = 0.001$ [S1].

### Optimization Schedule
Training uses the Adam optimizer with a logarithmic learning rate annealing and a warmup phase [S1].

*   **Total Iterations ($n$):** 1,000,000 [S1].
*   **Batch Size:** 4096 [S1].
*   **Learning Rate Annealing:** Logarithmic interpolation from $\eta_0 = 5 \cdot 10^{-4}$ to $\eta_n = 5 \cdot 10^{-6}$ [S1].
*   **Warmup:** The first $n_w = 2500$ iterations use a warmup scale factor annealed from 0.01 to 1 [S1].

The learning rate $\eta_i$ at iteration $i$ is calculated as:
$$ \eta_i = \left( \lambda_w + (1 - \lambda_w) \sin\left(\frac{\pi}{2} \operatorname{clip}\left(\frac{i}{n_w}, 0, 1\right)\right) \right) \times \exp\left( \left(1 - \frac{i}{n}\right) \log(\eta_0) + \frac{i}{n} \log(\eta_n) \right) $$
where $\lambda_w = 0.01$ is the constant starting value of the warmup multiplier [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_learning_rate(iteration, total_iterations=1000000, warmup_iterations=2500, 
                          lr_start=5e-4, lr_end=5e-6):
    """
    Computes the learning rate for Mip-NeRF/JaxNeRF training.
    """
    i = iteration
    n = total_iterations
    n_w = warmup_iterations
    lambda_w = 0.01
    
    # Warmup multiplier: anneals from lambda_w to 1 during warmup
    clip_val = min(max(i / n_w, 0.0), 1.0)
    warmup_multiplier = lambda_w + (1 - lambda_w) * math.sin((math.pi / 2) * clip_val)
    
    # Base learning rate: logarithmic annealing
    base_lr = math.exp((1 - i / n) * math.log(lr_start) + (i / n) * math.log(lr_end))
    
    return warmup_multiplier * base_lr

def density_activation(x):
    """Shifted softplus for density."""
    return math.log(1 + math.exp(x - 1))

def color_activation(x, epsilon=0.001):
    """Widened sigmoid for color."""
    return (1 + 2 * epsilon) / (1 + math.exp(-x)) - epsilon
```

### Edge Cases and Pitfalls
*   **Catastrophic Failure:** Using standard ReLU for density can cause the MLP to emit negative values everywhere, leading to zero gradients and optimization failure. The shifted softplus prevents this [S1].
*   **Black/White Pixel Saturation:** Standard sigmoid saturation can cause optimization failure when trying to explain extreme pixel values. The widened sigmoid avoids this [S1].
*   **Identity Concatenation:** Keeping the identity concatenation (position/view direction appended to PE) makes the formalization of IPE features challenging and does not improve performance. It should be removed [S1].

## Subtopics

### [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md)
Mip-NeRF 360 optimization setup defines the training configuration for unbounded neural radiance fields, specifically addressing the challenge of rendering large-scale urban scenes with limited video memory by warping the infinite scene domain into a finite sphere. It distinguishes itself from standard NeRF training by combining a specific scene contraction function, a multi-stage proposal network sampling strategy, and a robust learning schedule that prevents aliasing artifacts in unbounded spaces.

### [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md)
Scale-Dependent Downweighting is a technique to anti-alias trilinear interpolation in neural radiance fields by downweighting features interpolated from grid cells that are small relative to the sampling Gaussian's spread, preventing aliasing artifacts from high-frequency content. Unlike naive trilinear interpolation which causes aliasing at high frequencies, this method uses the Gaussian CDF to weight samples, similar to Mip-NeRF's IPE features but applied to grid-based representations.

### [S-NeRF Training Configuration](15165-s-nerf-training-configuration.md)
S-NeRF defines the specific hyperparameters, network architectures, and training schedules required to optimize neural radiance fields for dynamic street scenes using LiDAR and RGB data. It distinguishes itself from standard NeRF variants by employing a coarse-fine training strategy with separate configurations for foreground vehicles and background street scenes, and by integrating confidence maps derived from multi-view geometric consistency to mitigate depth outliers.

### [Mip-Grid](21844-mip-grid.md)
Mip-Grid is a method for integrating anti-aliasing techniques into grid-based radiance field representations, mitigating aliasing artifacts that occur when rendering scenes at varying camera distances or resolutions. Unlike methods that store distinct grids for each scale or rely on heavy multisampling, Mip-Grid generates multi-scale grid representations from a single shared base grid using learnable depth-wise convolutions, allowing features to vary with camera distance while maintaining fast training times and minimal storage overhead. It sets itself apart from siblings like [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md) by operating directly on the grid structure rather than just the ray integration, and from [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md) by explicitly generating hierarchical feature grids rather than downweighting interpolation weights.

### Alternative Gaussian Positional Encoding
An alternative to Integrated Positional Encoding (IPE) involves concatenating the positional encodings of the mean and the (signed) square root of the covariance matrix elements [S1]. Specifically, positional encoding with $L=12$ is applied to the mean $\mu$, and positional encoding with $L=2$ is applied to $\operatorname{vec}(\operatorname{triu}(\operatorname{sign}(\Sigma) \circ \sqrt{|\Sigma|}))$ [S1]. This approach performs comparably to IPE in terms of PSNR and SSIM but is sensitive to the hyperparameters $L$ and the covariance parameterization design [S1]. IPE is preferred because it is hyperparameter-free and more compact [S1].

### Reflected Cone Derivation
A geometric derivation used in Mip-NeRF to handle reflections [S2]. To ensure consistent view-dependent appearance, the origin of the reflected ray $\mathbf{o}'$ is shifted so that its cone radius matches the incident ray's cone radius at the intersection point $\bar{\mathbf{x}}$ [S2]. The condition is:
$$ \dot{r}\|\bar{\mathbf{x}} - \mathbf{o}\| = \dot{r}'\|\bar{\mathbf{x}} - \mathbf{o}'\| $$
where $\dot{r}$ and $\dot{r}'$ are the incident and reflected ray radius parameters, $\mathbf{o}$ is the incident ray origin, and $\mathbf{o}'$ is the reflected ray origin [S2]. This equation, along with the constraint that $\mathbf{o}'$ lies on the reflected ray, determines the shifted origin [S2].

### Mip-NeRF Covariance Annealing
A technique to define the covariance matrix $\Sigma$ of the ray interval Gaussian distribution using a linearly annealed scale parameter $\lambda_\Sigma$, rather than deriving it from camera geometry [S3, unofficial]. $\Sigma$ is defined as $\Sigma = \lambda_\Sigma^2 I_3$, where $\lambda_\Sigma$ is linearly annealed from $5 \times 10^{-2}$ to $2 \times 10^{-3}$ over the first 5k steps of optimization (out of 15k total) [S3, unofficial].

### PL-MipNeRF
A variant of Mip-NeRF that integrates the piecewise linear opacity formulation into the volume rendering integral as a drop-in replacement [S4]. It shows significant improvement over baseline Mip-NeRF in difficult scenarios such as fine texture details (Chair) and grazing angle views (Mic) [S4].

## Operating characteristics
*   **Training Speed:** Mip-NeRF is 13x faster to train than a brute-force supersampling baseline (16 rays/pixel) [S1].
*   **Evaluation Speed:** Mip-NeRF is 16x faster to evaluate than the supersampling baseline [S1].
*   **Quality:** Mip-NeRF outperforms the 16x supersampling baseline in PSNR, SSIM, and LPIPS metrics on multiscale Blender datasets [S1].
*   **Stability:** The specific activation functions and learning rate warmup improve training stability, particularly with large learning rates, though they do not significantly change the final accuracy or speed compared to standard NeRF settings [S1].
*   **Memory:** The use of IPE and unified multiscale models reduces memory overhead compared to separate per-scale models [S1].

## Background
The details described herein originate from the Mip-NeRF paper (Barron et al., 2021) and the JaxNeRF codebase documentation [S1]. The "miscellaneous" changes, including the activation functions and identity removal, were introduced to simplify the method's formalization and increase robustness during optimization without altering the core accuracy or speed [S1]. The reflected cone derivation is detailed in the NeRF-Casting paper (2024) as an extension for consistent reflections [S2].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S2] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S3] DreamFusion: Text-to-3D using 2D Diffusion, https://doi.org/10.48550/arxiv.2209.14988, fetched 2026-10-27 (unofficial)
[S4] NeRF Revisited: Fixing Quadrature Instability in Volume Rendering, https://doi.org/10.52202/075280-1139, fetched 2026-10-26
