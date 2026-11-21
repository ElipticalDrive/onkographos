# Mip-NeRF Optimization and Activation Details

Also called: JaxNeRF Training Procedure, Mip-NeRF Misc. Changes

## Overview
This article defines the specific hyperparameters, activation functions, and input preprocessing steps used to train Mip-NeRF and JaxNeRF models. These details address implementation-specific challenges such as training stability, catastrophic failure modes in density prediction, and optimization robustness, rather than introducing new geometric or rendering primitives. The method sets itself apart from standard NeRF configurations by replacing the identity concatenation in MLP inputs with pure Integrated Positional Encoding (IPE), using shifted softplus and widened sigmoid activations for density and color respectively, and employing a specific warmup learning rate schedule. It builds on [Method-Specific Training Configurations](6267-method-specific-training-configurations.md).

## How it works
The training procedure minimizes photometric error using the Adam optimizer. The following details define the network input, activation functions, and optimization schedule.

### Inputs and Outputs
*   **Input:** Integrated Positional Encoding (IPE) of the ray mean and covariance. The original position and view direction are **not** concatenated with the encoding [S1].
*   **View Direction:** The ray direction $d$ is normalized, positionally encoded with $L=4$, and injected into the last layer of the MLP after density $\tau$ is predicted but before color $c$ is predicted [S1].
*   **Output:** Density $\tau$ and color $c$ for each sample point.

### Data Structures
*   **MLP Input:** A vector containing only the IPE features.
*   **MLP Architecture:** Standard MLP as defined in the parent Mip-NeRF method. The view direction encoding is added to the final layer weights/biases or concatenated before the final output layer [S1].

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
*   **Warmup:** The first $n_w = 2500$ iterations use a warmup scale factor $\lambda_w$ annealed from 0.01 to 1 [S1].

The learning rate $\eta_i$ at iteration $i$ is calculated as:
$$ \eta_i = \left( \lambda_w + (1 - \lambda_w) \sin\left(\frac{\pi}{2} \operatorname{clip}\left(\frac{i}{n_w}, 0, 1\right)\right) \right) \times \exp\left( \left(1 - \frac{i}{n}\right) \log(\eta_0) + \frac{i}{n} \log(\eta_n) \right) $$
where $\lambda_w$ is the current warmup scale factor, which is linearly interpolated from 0.01 to 1 during the first $n_w$ iterations [S1].

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
    
    # Calculate warmup scale factor lambda_w
    # Annealed from 0.01 to 1 during warmup
    clip_val = min(max(i / n_w, 0.0), 1.0)
    lambda_w = 0.01 + (1.0 - 0.01) * (0.5 * (1 - math.cos(math.pi * clip_val)))
    # Note: The source uses sin(pi/2 * clip). 
    # sin(pi/2 * x) goes from 0 to 1 as x goes from 0 to 1.
    # lambda_w = 0.01 + (1 - 0.01) * sin(pi/2 * clip_val)
    # Let's stick strictly to the formula in [S1]:
    # lambda_w is annealed from 0.01 to 1. The term multiplying the base LR is:
    # (lambda_w + (1 - lambda_w) * sin(...))
    # Wait, the formula in [S1] is:
    # eta_i = (lambda_w + (1 - lambda_w) * sin(...)) * base_lr
    # And lambda_w is annealed from 0.01 to 1.
    # This implies lambda_w is a variable that changes.
    # Let's re-read carefully: "scale factor that is smoothly annealed between lambda_w = 0.01 and 1"
    # The formula uses lambda_w inside. This suggests lambda_w in the formula IS the annealed value.
    # Let's define lambda_w_current as the annealed value.
    
    # Re-evaluating the formula structure from [S1]:
    # eta_i = (lambda_w + (1 - lambda_w) * sin((pi/2) * clip(i/n_w, 0, 1))) * base_lr
    # If lambda_w is the annealed value, it appears on both sides? No.
    # "we scale the basic learning rate by an additional scale factor that is smoothly annealed between lambda_w = 0.01 and 1"
    # This phrasing is slightly ambiguous. Usually, warmup is a multiplier.
    # Let's look at the formula again:
    # eta_i = (lambda_w + (1 - lambda_w) * sin(...)) * exp(...)
    # If lambda_w is constant 0.01, the term is 0.01 + 0.99*sin(...).
    # If lambda_w is the *current* value being annealed, it's recursive?
    # Standard interpretation: lambda_w is a parameter that is annealed.
    # However, the text says "scale factor ... annealed between lambda_w=0.01 and 1".
    # It is most likely that the term in the parenthesis IS the warmup multiplier.
    # Let's assume the formula defines the multiplier M_i.
    # M_i = lambda_w_current + (1 - lambda_w_current) * sin(...)
    # But lambda_w_current is defined as being annealed.
    # Actually, looking at JaxNeRF docs or standard practices:
    # Often lambda_w is a fixed hyperparameter for the warmup curve shape, or it's the value at the end.
    # Let's look at the limits.
    # If i=0, sin(0)=0. M_0 = lambda_w.
    # If i=n_w, sin(pi/2)=1. M_nw = lambda_w + (1-lambda_w) = 1.
    # So the multiplier goes from lambda_w to 1.
    # The text says "annealed between lambda_w = 0.01 and 1".
    # This implies lambda_w is the STARTING value of the multiplier, and it ends at 1.
    # So lambda_w is a constant 0.01 in the formula?
    # "we scale the basic learning rate by an additional scale factor that is smoothly annealed between lambda_w = 0.01 and 1"
    # This implies the SCALE FACTOR is the thing annealing.
    # The formula calculates the scale factor.
    # So lambda_w in the formula is the constant 0.01.
    
    lambda_w_const = 0.01
    
    # Warmup multiplier
    sin_term = math.sin((math.pi / 2) * clip_val)
    warmup_multiplier = lambda_w_const + (1 - lambda_w_const) * sin_term
    
    # Base learning rate (logarithmic annealing)
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
Mip-NeRF 360 optimization setup defines the training configuration for unbounded neural radiance fields, specifically addressing the challenge of rendering large-scale urban scenes with limited video memory by warping the infinite scene domain into a finite sphere. It distinguishes itself from standard NeRF training by combining a specific scene contraction function, a multi-stage proposal network sampling strategy, and a robust learning schedule that prevents aliasing artifacts in unbounded spaces. [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md)

### [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md)
Scale-Dependent Downweighting is a technique to anti-alias trilinear interpolation in neural radiance fields by downweighting features interpolated from grid cells that are small relative to the sampling Gaussian's spread, preventing aliasing artifacts from high-frequency content. Unlike naive trilinear interpolation which causes aliasing at high frequencies, this method uses the Gaussian CDF to weight samples, similar to Mip-NeRF's IPE features but applied to grid-based representations. [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md)

### [S-NeRF Training Configuration](15165-s-nerf-training-configuration.md)
S-NeRF defines the specific hyperparameters, network architectures, and training schedules required to optimize neural radiance fields for dynamic street scenes using LiDAR and RGB data. It distinguishes itself from standard NeRF variants by employing a coarse-fine training strategy with separate configurations for foreground vehicles and background street scenes, and by integrating confidence maps derived from multi-view geometric consistency to mitigate depth outliers. [S-NeRF Training Configuration](15165-s-nerf-training-configuration.md)

### Alternative Gaussian Positional Encoding
An alternative to Integrated Positional Encoding (IPE) involves concatenating the positional encodings of the mean and the (signed) square root of the covariance matrix elements. Specifically, positional encoding with $L=12$ is applied to the mean $\mu$, and positional encoding with $L=2$ is applied to $\operatorname{vec}(\operatorname{triu}(\operatorname{sign}(\Sigma) \circ \sqrt{|\Sigma|}))$. This approach performs comparably to IPE in terms of PSNR and SSIM but is sensitive to the hyperparameters $L$ and the covariance parameterization design. IPE is preferred because it is hyperparameter-free and more compact [S1].

### Reflected Cone Derivation
A geometric derivation used in Mip-NeRF to handle reflections. To ensure consistent view-dependent appearance, the origin of the reflected ray $\mathbf{o}'$ is shifted so that its cone radius matches the incident ray's cone radius at the intersection point $\bar{\mathbf{x}}$. The condition is:
$$ \dot{r}\|\bar{\mathbf{x}} - \mathbf{o}\| = \dot{r}'\|\bar{\mathbf{x}} - \mathbf{o}'\| $$
where $\dot{r}$ and $\dot{r}'$ are the incident and reflected ray radius parameters, $\mathbf{o}$ is the incident ray origin, and $\mathbf{d}'$ is the reflected direction. This equation, along with the constraint that $\mathbf{o}'$ lies on the reflected ray, determines the shifted origin [S2].

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
