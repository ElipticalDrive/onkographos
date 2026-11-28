# Activation Feature Density Approximation

Also called: Approximate density extraction, f3 density functions

## Overview
Activation Feature Density Approximation estimates volumetric density $\hat{d}$ directly from intermediate activation features $f_\ell$ of a coordinate-based MLP, replacing the expensive evaluation of the coarse density network with a lightweight computation [S1]. It differs from sibling methods that modify network topology or input space by exploiting the statistical properties of existing hidden activations to infer scene structure, primarily targeting inference speedup in real-world scenes [S1].

## How it works
**Inputs and Outputs**
- **Input:** Intermediate activation features $f_\ell$ for a ray $r$ at layer $\ell$.
- **Output:** Estimated density values $\hat{d}$ and derived weights $\hat{w}$ for sample positions.

**Data Structures**
- **Activation Features ($A^{(\ell)}$):** A tensor of shape $N_s \times N_h$, where $N_s$ is the number of samples along the ray and $N_h$ is the number of hidden units in layer $\ell$. The values are recorded after the ReLU activation function [S1].
- **Feature Vector ($f_\ell$):** A vector of length $N_s$, computed per sample [S1].

**Steps**
1.  **Extract Activation Features:** For each sample $i$ along ray $r$, compute the mean activation over the hidden units $N_h$ [S1]:
    $$ f_\ell^{(i)} = \frac{1}{N_h} \sum_{j=1}^{N_h} A^{(\ell)}_{i,j} $$
    where $A^{(\ell)}_{i,j}$ is the activation of the $j$-th hidden unit for the $i$-th sample at layer $\ell$ [S1].

2.  **Compute Statistics:** Calculate the mean $\mu(f_\ell)$ and standard deviation $\sigma(f_\ell)$ of the feature vector $f_\ell$ along the last dimension (the $N_s$ samples) [S1]:
    $$ \mu(f_\ell) = \frac{1}{N_s} \sum_{i=1}^{N_s} f_\ell^{(i)} $$
    $$ \sigma(f_\ell) = \sqrt{\frac{1}{N_s} \sum_{i=1}^{N_s} (f_\ell^{(i)} - \mu(f_\ell))^2} $$

3.  **Estimate Density:** Apply one of three handcrafted functions $f_k$ ($k \in \{1, 2, 3\}$) to $f_\ell$ to produce the estimated density $\hat{d}$ [S1]:
    -   **f1 (Aggressive):**
        $$ \hat{d} = \text{ReLU}\left( (\mu(f_\ell) - \sigma(f_\ell)) - f_\ell \right) $$
    -   **f2 (Conservative):**
        $$ \hat{d} = \text{ReLU}\left( \left(\mu(f_\ell) - \frac{\sigma(f_\ell)}{2}\right) - f_\ell \right) $$
    -   **f3 (Squared Conservative):**
        $$ \hat{d} = \text{ReLU}\left( \left(\mu(f_\ell) - \frac{\sigma(f_\ell)}{2}\right) - f_\ell \right)^2 $$
    
    These functions are designed such that low activation values (indicating significant density) result in positive density estimates, while high activations result in zero [S1].

4.  **Normalize to Weights:** Convert the estimated density $\hat{d}$ to weights $\hat{w}$ using piecewise-constant PDF normalization [S1]:
    $$ \hat{w}_i = \frac{\hat{d}_i}{\sum_{j=1}^{N_s} \hat{d}_j} $$

5.  **Sample Positions:** Perform inverse transform sampling on the weights $\hat{w}$ to determine sample positions, as done in standard NeRF [S1].

**Parameters**
-   **Layer $\ell$:** The method is applied to the first 3 layers of the MLP. Layer 2 is identified as the optimal trade-off between speedup and quality [S1].
-   **Function Choice:** $f_2$ is recommended for Mip-NeRF as it performed best in experiments [S1].

**Edge Cases and Pitfalls**
-   **High Variance:** Function $f_1$ is prone to failure if the activation distribution along a ray exhibits high standard deviation [S1].
-   **Insufficient Detail:** Functions $f_2$ and $f_3$ are more conservative and may lead to insufficient detail, particularly as the interval between fine samples grows larger [S1].
-   **MLP Layer Selection:** Applying the method to earlier layers accelerates inference more substantially but may reduce reconstruction quality compared to later layers (e.g., layer 2 vs layer 1 or 3) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def relu(x):
    return max(0.0, x)

def activation_feature_density_approximation(activations, layer_index):
    """
    activations: list of lists, shape (N_s, N_h)
    layer_index: int, the layer from which activations are taken (1, 2, or 3)
    """
    N_s = len(activations)
    N_h = len(activations[0])
    
    # Step 1: Extract mean activation per sample
    f_l = []
    for i in range(N_s):
        mean_val = sum(activations[i]) / N_h
        f_l.append(mean_val)
        
    # Step 2: Compute mean and standard deviation of f_l
    mu_f = sum(f_l) / N_s
    variance = sum((x - mu_f) ** 2 for x in f_l) / N_s
    sigma_f = math.sqrt(variance)
    
    # Step 3: Estimate density using f2 (recommended)
    # f2: ReLU( (mu - sigma/2) - f )
    threshold = mu_f - (sigma_f / 2.0)
    d_hat = []
    for x in f_l:
        val = threshold - x
        d_hat.append(relu(val))
        
    # Step 4: Normalize to weights
    sum_d = sum(d_hat)
    if sum_d == 0:
        # Handle case where all densities are zero
        w_hat = [0.0] * N_s
    else:
        w_hat = [d / sum_d for d in d_hat]
        
    return w_hat
```

## Subtopics

### f1 Density Function
An aggressive approximation using $\mu - \sigma$. Prone to failure with high variance distributions [S1].

### f2 Density Function
A conservative approximation using $\mu - \sigma/2$. Balances speed and detail; best performance for Mip-NeRF [S1].

### f3 Density Function
A squared version of f2. More conservative than f1, potentially losing detail [S1].

## Operating characteristics
-   **Time Complexity:** Significantly reduces inference time by replacing the coarse MLP evaluation with simple statistical operations. Speedups of 40-58% were observed on NeRF and 72-76% on proposal networks [S1].
-   **Memory Complexity:** No additional memory overhead beyond storing intermediate activations, which are already required for backpropagation or visualization [S1].
-   **Performance:** On the Mip-NeRF LLFF dataset (1008x756), using layer 2 with $f_2$ achieved 25.44 PSNR, 0.75 SSIM, and 0.22 LPIPS [S1].
-   **Strengths:** Effective speedup for real-world scenes; leverages existing network internals without architectural changes [S1].
-   **Limitations:** Quality degradation if variance is high (f1) or if detail is lost (f2/f3); dependent on the specific layer chosen [S1].
-   **When to Use:** When inference speed is critical and the scene has well-behaved activation distributions; particularly effective for Mip-NeRF due to its higher sample count [S1].
-   **When Not to Use:** When high fidelity is required in scenes with complex, high-variance activation patterns; or when using layers that do not store structural information effectively [S1].

## Background
The method was proposed in "Analyzing the Internals of Neural Radiance Fields" (2024) [S1]. It is based on the observation that intermediate activations in coordinate-based MLPs exhibit structural information and that activation minima correlate with significant density [S1]. The approach was evaluated on NeRF, Mip-NeRF, and proposal network samplers (nerfacto) using Blender and LLFF datasets [S1].

## Sources
[S1] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
