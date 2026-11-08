# Minkowski's Metric
Also called: Lgamma Loss, Lp Norm Loss

## Overview
Minkowski's metric is a family of distance measures that treats an image as a 2D signal to calculate the dissimilarity between a reference image and a test image [S1, unofficial]. It generalizes the Mean Squared Error (MSE) by allowing the order parameter $\gamma$ to vary, enabling different sensitivities to pixel-wise errors [S1, unofficial]. Unlike its sibling [PSNR](2742-psnr-metric-calculation.md), which normalizes error by signal power to provide a logarithmic decibel scale, Minkowski's metric provides a raw, unnormalized magnitude of error [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Two images, $x$ (reference) and $y$ (test), each containing $N$ samples (pixels).
- Input: Order parameter $\gamma \in [1, \infty)$.
- Output: The Minkowski distance $E_{\gamma}$.

**Data Structures:**
- Images $x$ and $y$ are treated as vectors of length $N$.
- $\gamma$ is a scalar float.

**Steps:**
1. Calculate the absolute difference for each sample $i$: $d_i = |x_i - y_i|$.
2. Raise each difference to the power of $\gamma$: $d_i^{\gamma}$.
3. Sum the powered differences: $S = \sum_{i=1}^N d_i^{\gamma}$.
4. Compute the mean: $\bar{d}^{\gamma} = \frac{1}{N} S$.
5. Take the $\gamma$-th root: $E_{\gamma} = (\bar{d}^{\gamma})^{\frac{1}{\gamma}}$.

**Formula:**
$$E_{\gamma} = \left( \frac{1}{N} \sum_{i=1}^N |x_i - y_i|^{\gamma} \right)^{\frac{1}{\gamma}}$$

Where:
- $N$ is the number of samples in the image.
- $x_i$ and $y_i$ are the $i$-th samples of images $x$ and $y$.
- $\gamma$ is the order parameter, where $1 \le \gamma < \infty$.

**Parameters:**
- $\gamma = 2$: Yields the root-mean-square error (RMSE). Note that the standard Mean Squared Error (MSE) is $E_2^2$ [S1, unofficial].
- $\gamma = 1$: Yields the Mean Absolute Error (MAE).
- $\gamma \to \infty$: Approaches the Chebyshev distance (maximum absolute difference).

**Edge Cases and Pitfalls:**
- When $\gamma=2$, the formula simplifies to $\sqrt{MSE}$ [S1, unofficial]. Implementers should be aware that MSE is often used directly as a loss function without the square root, whereas Minkowski's metric explicitly includes the root for $\gamma > 1$.
- For large $\gamma$, numerical overflow may occur if differences are large; scaling inputs to $[0, 1]$ is recommended.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minkowski_metric(x, y, gamma):
    """
    Calculate Minkowski distance between two images x and y.
    
    Args:
        x: Reference image (array-like of shape (N,))
        y: Test image (array-like of shape (N,))
        gamma: Order parameter (float, >= 1)
        
    Returns:
        E_gamma: Minkowski distance (float)
    """
    import numpy as np
    
    # Ensure inputs are numpy arrays for vectorized operations
    x = np.asarray(x)
    y = np.asarray(y)
    
    # Number of samples
    N = x.size
    
    # Calculate absolute differences
    diff = np.abs(x - y)
    
    # Raise to power gamma
    diff_gamma = np.power(diff, gamma)
    
    # Sum and average
    mean_diff_gamma = np.mean(diff_gamma)
    
    # Take gamma-th root
    E_gamma = np.power(mean_diff_gamma, 1.0 / gamma)
    
    return E_gamma
```

## Subtopics

### [Mean Square Error (MSE)](6038-mean-square-error-mse.md)
Mean Square Error (MSE) is a specific instance of Minkowski's metric where the order parameter $\gamma=2$, used to quantify the dissimilarity between a reference image and a rendered test image by averaging the squared differences of their pixel values. It serves as the foundational photometric loss for differentiable volume rendering optimization, prioritizing pixel-wise accuracy over perceptual similarity. Unlike its sibling [PSNR](52-name.md), which normalizes MSE by signal power to provide a logarithmic decibel scale, MSE provides a raw, unnormalized magnitude of error.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of pixels.
- **Memory Complexity:** $O(N)$ to store the difference array.
- **Strengths:** Simple to compute; provides a direct measure of pixel-wise error magnitude.
- **Limitations:** Does not account for perceptual similarity; sensitive to outliers when $\gamma$ is large.
- **When to use:** When a raw, unnormalized error metric is needed for optimization or logging, and perceptual metrics are not required.

## Background
Minkowski's metric is a generalization of the $L_p$ norm, named after Hermann Minkowski. In the context of image quality assessment, it is used as a full-reference method to calculate dissimilarities between a reference and a test image [S1, unofficial]. The Mean Squared Error (MSE) is the most commonly used instance of this metric [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
