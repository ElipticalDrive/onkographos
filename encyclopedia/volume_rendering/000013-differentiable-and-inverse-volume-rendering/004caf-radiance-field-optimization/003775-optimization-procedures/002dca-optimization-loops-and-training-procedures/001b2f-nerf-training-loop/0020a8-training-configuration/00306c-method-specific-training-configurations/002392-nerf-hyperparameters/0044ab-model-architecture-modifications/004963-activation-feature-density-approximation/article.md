# Activation Feature Density Approximation

Also called: Approximate density extraction, f3 density functions

## Overview
Activation Feature Density Approximation estimates volumetric density $\hat{d}$ directly from intermediate activation features $f_\ell$ of a coordinate-based MLP, replacing the expensive evaluation of the coarse network $\Theta_{coarse}$ with a lightweight computation [S1]. It enables significant inference speedups in real-world scenes by leveraging the structural information naturally stored in early MLP layers [S1]. Unlike standard NeRF which requires full forward passes for density estimation, this method uses statistical properties of activations to predict where density exists, allowing for adaptive sampling strategies [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Intermediate activation features $f_\ell$ for a ray $r$ at layer $\ell$.
- **Output:** Estimated density vector $\hat{\vec{d}}$ and estimated weights $\hat{\vec{w}}$ for inverse transform sampling.

**Data Structures:**
- $A^{(\ell)} \in \mathbb{R}^{N_s \times N_h}$: Activation matrix for layer $\ell$, where $N_s$ is the number of samples along the ray and $N_h$ is the number of hidden units. Values are recorded after the ReLU activation function [S1].
- $f_\ell \in \mathbb{R}^{N_s}$: Feature vector for the ray, computed as the mean over hidden units [S1].

**Steps:**
1.  **Extract Activation Features:** Compute the mean activation $f_\ell$ for each sample along the ray $r$ [S1]:
    $$
    f_\ell = \frac{1}{N_h} \sum_{i=1}^{N_h} A^{(\ell)}_i
    $$
    where $A^{(\ell)}_i$ is the activation vector for the $i$-th hidden unit at layer $\ell$ [S1].

2.  **Compute Statistics:** Calculate the mean $\mu(f_\ell)$ and standard deviation $\sigma(f_\ell)$ of the feature vector $f_\ell$ along the last dimension (the $N_s$ samples) [S1].

3.  **Estimate Density:** Apply one of three handcrafted functions $f_1, f_2,$ or $f_3$ to $f_\ell$ to produce the estimated density $\hat{\vec{d}}$ [S1]:
    $$
    \hat{\vec{d}} = f_k(f_\ell), \quad k \in \{1, 2, 3\}
    $$
    The functions are defined as [S1]:
    $$
    f_1(f_\ell) = \text{ReLU}\left( (\mu(f_\ell) - \sigma(f_\ell)) - f_\ell \right)
    $$
    $$
    f_2(f_\ell) = \text{ReLU}\left( \left(\mu(f_\ell) - \frac{\sigma(f_\ell)}{2}\right) - f_\ell \right)
    $$
    $$
    f_3(f_\ell) = \text{ReLU}\left( \left(\mu(f_\ell) - \frac{\sigma(f_\ell)}{2}\right) - f_\ell \right)^2
    $$
    These functions are designed such that activation minima correspond to significant density, transforming $f_\ell$ into a vector of mostly zeros except where density is significant [S1].

4.  **Normalize Weights:** Convert estimated density $\hat{\vec{d}}$ to weights $\hat{\vec{w}}$ using piecewise-constant PDF normalization [S1]:
    $$
    \hat{w}_i = \frac{\hat{d}_i}{\sum_{j=1}^{N_s} \hat{d}_j}
    $$

5.  **Sample Positions:** Perform inverse transform sampling on weights $\hat{\vec{w}}$ to determine sample positions, as done in standard NeRF [S1].

**Parameters:**
- **Layer $\ell$:** The method is typically applied to the first 3 layers of the MLP. Layer 2 ($\ell=2$) is often preferred for balancing speedup and quality [S1].
- **Function Choice:** $f_2$ generally performs best for Mip-NeRF [S1].

**Edge Cases and Pitfalls:**
- **High Standard Deviation:** $f_1$ is prone to failures if the distribution along a ray exhibits high standard deviation [S1].
- **Insufficient Detail:** $f_2$ and $f_3$ extract density more conservatively, which might lead to insufficient detail as the interval between fine samples grows larger [S1].
- **Layer Selection:** Applying the method to earlier layers accelerates inference more substantially but may impact reconstruction quality [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def activation_feature_density_approximation(activations, layer_idx, method='f2'):
    """
    Estimate density from intermediate activation features.
    
    Args:
        activations: np.ndarray of shape (N_layers, N_samples, N_hidden)
                     Activations after ReLU.
        layer_idx: int, index of the layer to use (e.g., 1 for layer 2).
        method: str, 'f1', 'f2', or 'f3'.
        
    Returns:
        weights: np.ndarray of shape (N_samples,), normalized weights for sampling.
    """
    # Step 1: Extract activation features f_l
    # Mean over hidden units N_h
    f_l = np.mean(activations[layer_idx], axis=-1)  # Shape: (N_samples,)
    
    # Step 2: Compute mean and standard deviation along the ray (samples)
    mu = np.mean(f_l)
    sigma = np.std(f_l)
    
    # Step 3: Apply handcrafted function to estimate density
    if method == 'f1':
        # ReLU( (mu - sigma) - f_l )
        raw_density = np.maximum(0.0, (mu - sigma) - f_l)
    elif method == 'f2':
        # ReLU( (mu - sigma/2) - f_l )
        raw_density = np.maximum(0.0, (mu - sigma / 2.0) - f_l)
    elif method == 'f3':
        # ReLU( (mu - sigma/2) - f_l )^2
        raw_density = np.maximum(0.0, (mu - sigma / 2.0) - f_l) ** 2
    else:
        raise ValueError("Method must be 'f1', 'f2', or 'f3'")
        
    # Step 4: Normalize weights
    # Handle case where sum is zero to avoid division by zero
    density_sum = np.sum(raw_density)
    if density_sum == 0:
        # Uniform weights if no density detected
        weights = np.ones_like(raw_density) / raw_density.shape[0]
    else:
        weights = raw_density / density_sum
        
    return weights
```

## Operating characteristics
- **Time Complexity:** Significantly reduces inference time by replacing the coarse MLP evaluation with simple statistical operations and a small subset of layers. Speedups of 40-76% are reported depending on the dataset and layer used [S1].
- **Memory Complexity:** Requires storing intermediate activations $A^{(\ell)}$ during the forward pass, which increases memory usage compared to standard NeRF [S1].
- **Performance:** On the Mip-NeRF LLFF dataset (1008x756), using layer 2 with $f_2$ achieved 25.44 PSNR, 0.75 SSIM, and 0.22 LPIPS, with a 40% speedup [S1]. On the Blender dataset, layer 2 with $f_2$ achieved 29.04 PSNR, 0.93 SSIM, and 0.07 LPIPS, with a 49% speedup [S1].
- **Strengths:** Effective for real-world scenes; leverages structural information in early MLP layers; compatible with proposal network samplers [S1].
- **Limitations:** Prone to failure with high variance in activation distributions ($f_1$); may lose detail in complex scenes ($f_2, f_3$); requires storing activations [S1].
- **When to use:** When inference speed is critical and the scene has well-behaved activation distributions; particularly effective for Mip-NeRF due to its expensive coarse network evaluation [S1].
- **When not to use:** When high detail preservation is required in scenes with high activation variance; when memory is constrained due to activation storage [S1].

## Background
The method was proposed in "Analyzing the Internals of Neural Radiance Fields" [S1]. It is based on the observation that intermediate activations of coordinate-based MLPs exhibit notable structural information and that activation feature space minima indicate samples with significant density [S1]. The approach was evaluated on NeRF, Mip-NeRF, and Mip-NeRF 360 using the Blender and LLFF datasets [S1].

## Sources
[S1] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
