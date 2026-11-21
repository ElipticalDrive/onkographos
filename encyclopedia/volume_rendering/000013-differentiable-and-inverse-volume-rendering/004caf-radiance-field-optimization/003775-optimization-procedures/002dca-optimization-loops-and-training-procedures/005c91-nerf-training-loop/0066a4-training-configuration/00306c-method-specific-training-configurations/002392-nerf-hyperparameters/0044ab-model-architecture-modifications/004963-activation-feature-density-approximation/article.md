# Activation Feature Density Approximation

Also called: Approximate density extraction, f3 density functions

## Overview
Activation Feature Density Approximation estimates volumetric density $\hat{d}$ directly from intermediate activation features $f_\ell$ of a coordinate-based MLP, replacing the expensive evaluation of the coarse network $\Theta_{coarse}$ with a lightweight computation [S1]. It differs from sibling methods like [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md) by retaining the standard density-based volume rendering formulation while optimizing the *extraction* of density values, rather than changing the underlying geometric representation or the rendering integral itself.

## How it works
**Inputs and Outputs**
*   **Input**: Intermediate activation features $A^{(\ell)}$ from layer $\ell$ of the coarse MLP $\Theta_{coarse}$ for a set of sample points along a ray $r$.
*   **Output**: Estimated density values $\hat{d}$ and derived weights $\hat{w}$ for inverse transform sampling.

**Data Structures**
*   $A^{(\ell)} \in \mathbb{R}^{N_s \times N_h}$: Activation matrix for layer $\ell$, where $N_s$ is the number of samples along the ray and $N_h$ is the number of hidden units in the layer. Values are recorded after the ReLU activation function has been applied [S1].

**Steps**
1.  **Feature Extraction**: Compute the scalar feature vector $f_\ell \in \mathbb{R}^{N_s}$ for the ray by averaging the activation values across the hidden dimension $N_h$ [S1]:
    $$ f_\ell = \frac{1}{N_h} \sum_{i=1}^{N_h} A^{(\ell)}_i $$
    where $A^{(\ell)}_i$ is the $i$-th hidden unit's activation vector across all $N_s$ samples [S1].

2.  **Statistical Computation**: Calculate the mean $\mu(f_\ell)$ and standard deviation $\sigma(f_\ell)$ of the feature vector $f_\ell$ along the sample dimension (last dimension, size $N_s$) [S1].

3.  **Density Estimation**: Apply one of three handcrafted functions $f_k$ ($k \in \{1, 2, 3\}$) to $f_\ell$ to produce the estimated density $\hat{d}$ [S1]:
    *   **f1 (Aggressive)**:
        $$ \hat{d} = \text{ReLU}\left( (\mu(f_\ell) - \sigma(f_\ell)) - f_\ell \right) $$
    *   **f2 (Conservative)**:
        $$ \hat{d} = \text{ReLU}\left( \left(\mu(f_\ell) - \frac{\sigma(f_\ell)}{2}\right) - f_\ell \right) $$
    *   **f3 (Squared Conservative)**:
        $$ \hat{d} = \text{ReLU}\left( \left(\mu(f_\ell) - \frac{\sigma(f_\ell)}{2}\right) - f_\ell \right)^2 $$
    
    These functions are designed such that activation minima correspond to significant density [S1].

4.  **Weight Normalization**: Convert estimated densities $\hat{d}$ to weights $\hat{w}$ using piecewise-constant PDF normalization [S1]:
    $$ \hat{w}_i = \frac{\hat{d}_i}{\sum_{j=1}^{N_s} \hat{d}_j} $$

5.  **Sampling**: Perform inverse transform sampling on weights $\hat{w}$ to determine sample positions for the fine network evaluation, as in standard NeRF [S1].

**Parameters**
*   **Layer $\ell$**: The method is typically applied to the first 3 layers of the MLP. Layer 2 ($\ell=2$) is recommended for Mip-NeRF as it provides the best trade-off between quality and speed [S1]. Earlier layers allow for greater acceleration but may reduce quality [S1].
*   **Function Choice**: $f_2$ is generally preferred for Mip-NeRF [S1]. $f_1$ is aggressive and prone to failure with high variance; $f_2$ and $f_3$ are conservative [S1].

**Edge Cases and Pitfalls**
*   **High Variance**: Equation $f_1$ is prone to failures if the activation distribution along a ray exhibits high standard deviation [S1].
*   **Detail Loss**: Equations $f_2$ and $f_3$ extract density more conservatively, which may lead to insufficient detail if the interval between fine samples grows larger [S1].
*   **Activation Function**: The method assumes activations are recorded *after* the ReLU function is applied [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def relu(x):
    return max(0.0, x)

def activation_density_approximation(activations, layer_index, method='f2'):
    """
    activations: list of lists, activations[layer_index][sample_idx][hidden_idx]
    Returns: estimated_density, weights
    """
    # Step 1: Extract feature vector f_l by averaging over hidden units
    # activations[layer_index] has shape (N_s, N_h)
    layer_acts = activations[layer_index]
    N_s = len(layer_acts)
    N_h = len(layer_acts[0])
    
    f_l = []
    for i in range(N_s):
        sum_hidden = 0.0
        for j in range(N_h):
            sum_hidden += layer_acts[i][j]
        f_l.append(sum_hidden / N_h)
        
    # Step 2: Compute mean and standard deviation along the sample dimension
    mean_f = sum(f_l) / N_s
    
    variance_sum = 0.0
    for val in f_l:
        diff = val - mean_f
        variance_sum += diff * diff
    std_f = math.sqrt(variance_sum / N_s)
    
    # Step 3: Apply density function
    estimated_density = []
    threshold = 0.0
    
    if method == 'f1':
        threshold = mean_f - std_f
    elif method == 'f2':
        threshold = mean_f - (std_f / 2.0)
    elif method == 'f3':
        threshold = mean_f - (std_f / 2.0)
    else:
        raise ValueError("Unknown method")
        
    for val in f_l:
        diff = threshold - val
        if method == 'f3':
            estimated_density.append(relu(diff) ** 2)
        else:
            estimated_density.append(relu(diff))
            
    # Step 4: Normalize to weights
    density_sum = sum(estimated_density)
    weights = []
    if density_sum > 0:
        for d in estimated_density:
            weights.append(d / density_sum)
    else:
        # Fallback if all densities are zero
        weights = [1.0 / N_s] * N_s
        
    return estimated_density, weights
```

## Subtopics

### Layer Selection
Choosing layer $\ell$ affects the balance between acceleration and quality. Layer 1 and 2 are most common; earlier layers offer more speedup but potentially lower fidelity [S1].

### Function Variant Selection
$f_1$ is aggressive and sensitive to variance; $f_2$ and $f_3$ are conservative. $f_2$ is empirically best for Mip-NeRF [S1].

## Operating characteristics
*   **Speedup**: Achieves significant inference speedups by avoiding the full coarse MLP evaluation. On the Mip-NeRF LLFF dataset (1008x756), using layer 2 with $f_2$ achieved a 40% speedup [S1]. On the Blender dataset, speedups ranged from 49% to 58% depending on the layer and function used [S1]. For proposal network samplers (nerfacto†), speedups reached 76% on Mip-NeRF 360 and 72% on Deep Blending [S1].
*   **Quality**: On Mip-NeRF LLFF, layer 2 with $f_2$ achieved 25.44 PSNR, 0.75 SSIM, and 0.22 LPIPS [S1]. On Blender, layer 2 with $f_2$ achieved 29.04 PSNR, 0.93 SSIM, and 0.07 LPIPS [S1].
*   **Hardware**: Results reported on an NVIDIA Quadro RTX 8000 [S1].
*   **Strengths**: Reduces inference time significantly, especially for methods with expensive coarse networks like Mip-NeRF [S1].
*   **Limitations**: May lose detail due to conservative density estimation ($f_2, f_3$) or fail with high variance ($f_1$) [S1].

## Background
The method was proposed in "Analyzing the Internals of Neural Radiance Fields" [S1]. It leverages the observation that intermediate activations in coordinate-based MLPs contain structural information and that activation minima correlate with significant density [S1]. It is primarily used to accelerate NeRF and Mip-NeRF inference and to speed up proposal network samplers [S1].

## Sources
[S1] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
