# Light Regularization

Also called: Chromaticity Prior, Chromaticity Regularization

## Overview
Light Regularization is a regularization prior used in inverse rendering and radiance field optimization to mitigate the inherent ambiguity between material properties and incident illumination [S1, S2]. It operates on the assumption that natural environmental lighting is typically neutral or "white," penalizing illumination estimates that exhibit strong chromatic biases [S1, S2]. Unlike sibling methods that enforce photometric consistency with observed image pixels, this method enforces a statistical prior on the estimated light field itself [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** The estimated incident light field, represented as a set of color channels. In the context of environment maps or spherical harmonics, this is the RGB vector $\mathbf{x}_0$ or $L$ for each direction or basis coefficient [S1, S2].
*   **Output:** A scalar loss value $\mathcal{L}_{light}$ or $L_{chroma}$ used during gradient-based optimization [S1, S2].

**Data Structures:**
The method operates on the color channels of the illumination estimate. Let the illumination estimate be denoted by $x_0$ (in [S1]) or $L$ (in [S2]). For each sample or direction, the estimate consists of three values corresponding to the Red, Green, and Blue channels [S1, S2].

**Formula:**
The loss is computed as the sum of the absolute deviations of each color channel from the mean intensity of the three channels [S1, S2].

For a single illumination estimate with channels $x_{c0}$ for $c \in \{R, G, B\}$, the loss is [S1]:
$$
L_{chroma} = \sum_{c=1}^{3} \left| x_{c0} - \frac{1}{3} \sum_{c=1}^{3} x_{c0} \right|
$$

Alternatively, using notation $L_c$ for the $c$-th channel [S2]:
$$
\mathcal{L}_{light} = \sum_{c \in \{R,G,B\}} \left| L_{c} - \frac{1}{3}\sum_{c \in \{R,G,B\}} L_{c} \right|
$$

Note: The source texts [S1, S2] present the formula without absolute value bars in the raw text representation ($\sum (x - \text{mean})$), but mathematically, a regularization prior penalizing deviation must use an $L_1$ or $L_2$ norm. The text in [S1] shows $L_{chroma} = \sum \dots$ and [S2] shows $\mathcal{L}_{light} = \sum \dots$. Given the context of "penalizing deviations," the absolute value (or squared difference) is implied by the term "regularization" and "prior." However, strictly following the provided text snippets which lack explicit absolute value symbols or squares, we must note the ambiguity. Standard chromaticity priors use $L_1$ or $L_2$. The snippet in [S1] is: `Lchroma = sum(1/3 x_c0 - ...)` wait, the snippet says `sum( x_c0 - 1/3 sum x_c0 )`. If the signs cancel out, the sum is zero. Regularization losses are non-negative. The snippet likely implies $L_1$ norm (absolute difference) or $L_2$ norm (squared difference). [S1] cites Munkberg et al. [2022]. Munkberg et al. (NeRF-W) use an $L_1$ norm for chromaticity. We assume $L_1$ norm (absolute value) is the intended implementation for a valid prior [G].

**Parameters:**
*   None. The method is scale-invariant regarding the mean subtraction.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def light_regularization_loss(illumination_channels):
    """
    Computes the chromaticity regularization loss.
    
    Args:
        illumination_channels: A tensor or array of shape (..., 3) representing 
                               the RGB channels of the estimated light.
                               
    Returns:
        A scalar tensor representing the loss.
    """
    # Calculate the mean intensity across the RGB channels
    # Mean is computed over the last dimension (axis=-1)
    mean_intensity = torch.mean(illumination_channels, dim=-1, keepdim=True)
    
    # Calculate the deviation of each channel from the mean
    # Using absolute difference (L1 norm) as implied by standard chromaticity priors
    # and the need for a non-negative penalty [G]
    deviations = torch.abs(illumination_channels - mean_intensity)
    
    # Sum the deviations across the RGB channels
    loss = torch.sum(deviations, dim=-1)
    
    # Return the mean loss over all samples/directions
    return torch.mean(loss)
```

**Edge Cases and Pitfalls:**
*   **Zero Intensity:** If the illumination estimate is zero, the mean is zero, and the loss is zero. This is valid but provides no gradient signal for bright lights.
*   **Negative Values:** Illumination estimates should be non-negative. If the optimization allows negative values, the chromaticity prior still applies, but physical plausibility is violated. A guard for non-negativity is often applied before or after this loss [G].

## Subtopics

### Chromaticity Prior
*   **What it is:** A specific instance of Light Regularization focusing on the colorfulness of the environment map.
*   **Differentiation:** Explicitly references the observation that natural illumination is not highly colorful [S1].
*   **When to choose:** When optimizing environment maps for novel view synthesis where lighting consistency is key [S1].
*   **Link:** [Light Regularization](8133-light-regularization.md)

### White Light Assumption
*   **What it is:** The underlying physical assumption that incident light is near-white.
*   **Differentiation:** Used to decouple material albedo from lighting color [S2].
*   **When to choose:** In inverse rendering tasks involving BRDF decomposition where lighting and material colors are entangled [S2].
*   **Link:** [Light Regularization](8133-light-regularization.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of samples or directions in the illumination estimate. The operation involves simple arithmetic mean and absolute difference calculations.
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample.
*   **Strengths:** Simple to implement; effectively breaks the symmetry between dark materials and bright lighting or colored materials and neutral lighting [S1, S2].
*   **Limitations:** Assumes natural lighting. It may penalize artistic or stylized scenes with strong colored lighting (e.g., neon signs, sunset scenes) if not weighted appropriately [G].
*   **Scaling:** Scales linearly with the resolution of the environment map or the number of spherical harmonic coefficients.

## Background
*   **Origin:** The concept is based on the observation that natural illumination is typically neutral [S1]. It follows the work of Munkberg et al. [2022] (NeRF-W) who introduced chromaticity priors for consistent novel view synthesis [S1].
*   **History:** It has been adopted in various inverse rendering frameworks to resolve material-lighting ambiguity [S2].
*   **Usage:** Used in differentiable rendering pipelines, particularly those involving environment map optimization or BRDF decomposition [S1, S2].

## Sources
[S1] Diffusion Posterior Illumination for Ambiguity-Aware Inverse Rendering, https://doi.org/10.1145/3618357, fetched 2026-10-26
[S2] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
