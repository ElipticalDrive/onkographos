# Color MLP with Primitive-Aware Bias

Also called: Color MLP, MLP Color Representation, Primitive-Aware Color MLP, tiny MLP for representing color

## Overview
A small Multi-Layer Perceptron (MLP) used to represent view-dependent color in the linear RAW color space, replacing Spherical Harmonics (SH) to enhance expressiveness and optimization stability [S1]. It addresses the inability of SH to adequately represent RAW linear color data, which causes strange color representations early in training and negatively affects final structural reconstruction [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Camera pose $v$ (concatenated with feature vector), Gaussian index $i$.
*   **Output:** View-dependent color $c_i$ in linear RAW space.

**Data Structures**
Each Gaussian $i$ maintains two learnable parameters:
1.  **Color Feature ($f_i$):** A vector sampled from a normal distribution.
2.  **Color Bias ($b_i$):** A scalar initialized based on projected pixel colors.

**Numbered Steps**
1.  **Initialization:** For each Gaussian $i$ derived from the scattered point set $S'$:
    *   Project the point $s'_i$ onto all training images to obtain the set of pixel colors $\{c^p_{pix}\}_i$.
    *   Initialize the bias $b_i^{(0)} = \log(\text{avg}(\{c^p_{pix}\}_i))$ [S1].
    *   Initialize the feature $f_i^{(0)} \sim \mathcal{N}(0, \sigma_f)$ [S1].
2.  **Cloning/Splitting:** During Gaussian cloning or splitting operations, copy and assign the current $f_i$ and $b_i$ values to the new Gaussians [S1].
3.  **Color Computation:** To render color for a specific view:
    *   Concatenate the camera pose $v$ with the feature vector $f_i$.
    *   Feed the concatenated vector into the tiny MLP $F_\theta$.
    *   Add the bias $b_i$ to the MLP output.
    *   Apply the exponential activation function to simulate the unbounded HDR color space [S1].

**Formulas**
The final color $c_i$ is computed as:
$$
c_i = \exp(F_\theta(f_i, v) + b_i)
$$
where:
*   $F_\theta$ is the tiny MLP with parameters $\theta$ [S1].
*   $f_i$ is the color feature vector [S1].
*   $v$ is the camera pose [S1].
*   $b_i$ is the color bias [S1].
*   $\exp$ is the exponent activation function [S1].

**Parameters**
*   $\sigma_f$: Standard deviation for the normal distribution sampling $f_i^{(0)}$. Value not given in the sources.

**Edge Cases and Pitfalls**
*   **SH Inadequacy:** Using Spherical Harmonics instead of the Color MLP results in strange color representations early in training because SH cannot adequately represent the RAW linear color space [S1]. This negatively impacts the final structural reconstruction, even if the rendered image appears similar initially [S1].
*   **HDR Range:** The exponential activation is necessary because the HDR color space theoretically has no upper bound on color values [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_color_mlp_gaussian(point_s_prime, training_images):
    """
    Initialize color feature and bias for a single Gaussian.
    
    Args:
        point_s_prime: The 3D point coordinates.
        training_images: List of training image views containing the point.
        
    Returns:
        f_i: Color feature vector.
        b_i: Color bias scalar.
    """
    # 1. Project point onto all training images to get pixel colors
    # Note: Implementation of projection depends on camera intrinsics/extrinsics
    # which are assumed available in the broader context.
    projected_colors = []
    for view in training_images:
        if point_s_prime is visible in view:
            color = get_pixel_color_at_point(view, point_s_prime)
            projected_colors.append(color)
    
    # 2. Initialize bias b_i^(0) = log(avg({c^p_pix}_i))
    if len(projected_colors) > 0:
        avg_color = np.mean(projected_colors, axis=0)
        # Assuming avg_color is in linear space, take log component-wise or scalar?
        # Source implies scalar log of average, but color is vector.
        # Standard practice for HDR bias is often per-channel or scalar magnitude.
        # Source text: "b_i^(0) is setted by the log value of the average of {c^p_pix}_i"
        # We assume scalar bias applied to the MLP output which produces a vector?
        # Or bias is vector? Source says "color bias b_i". 
        # Formula: exp(F_theta(...) + b_i). If F_theta outputs vector, b_i must be vector.
        # However, source says "b_i^(0) = log(avg(...))". 
        # Let's assume b_i is a vector of same dimension as MLP output (e.g., 3 for RGB).
        b_i = np.log(avg_color + 1e-8) # Guard against log(0) [G]
    else:
        # Fallback if not visible in any training image
        b_i = np.zeros(3) # [G]
        
    # 3. Initialize feature f_i^(0) ~ N(0, sigma_f)
    sigma_f = 0.1 # Not given in sources [G]
    f_i = np.random.normal(0, sigma_f, size=3) # [G] Assuming 3D feature vector
    
    return f_i, b_i

def render_color_mlp(f_i, b_i, v, mlp_model):
    """
    Compute view-dependent color using the Color MLP.
    
    Args:
        f_i: Color feature vector.
        b_i: Color bias vector.
        v: Camera pose vector.
        mlp_model: The tiny MLP F_theta.
        
    Returns:
        c_i: Rendered color in linear RAW space.
    """
    # Concatenate feature and camera pose
    input_vector = np.concatenate([f_i, v])
    
    # Forward pass through MLP
    mlp_output = mlp_model(input_vector)
    
    # Add bias and apply exponential activation
    # Source: c_i = exp(F_theta(f_i, v) + b_i)
    c_i = np.exp(mlp_output + b_i)
    
    return c_i
```

## Operating characteristics
*   **Expressiveness:** Provides better stability and linear color space representation compared to Spherical Harmonics [S1].
*   **Optimization:** Introduces greater stability during the optimization process [S1].
*   **HDR Support:** Enables realistic exposure variation and HDR tone-mapping in real-time by rendering in the linear color space [S1].
*   **Structural Impact:** Avoids the negative impact on final structural reconstruction caused by SH's inability to handle RAW data [S1].

## Background
This method is part of the LE3D framework, which is based on 3D Gaussian Splatting (3DGS) [S1]. It was introduced to address issues with missing distant points in nighttime scenes and to improve HDR view synthesis [S1]. The approach replaces the standard SH color representation in Gaussian attributes with a tiny MLP [S1].

## Sources
[S1] Lighting Every Darkness with 3DGS: Fast Training and Real-Time Rendering for HDR View Synthesis, https://doi.org/10.52202/079017-2549, fetched 2026-10-26
