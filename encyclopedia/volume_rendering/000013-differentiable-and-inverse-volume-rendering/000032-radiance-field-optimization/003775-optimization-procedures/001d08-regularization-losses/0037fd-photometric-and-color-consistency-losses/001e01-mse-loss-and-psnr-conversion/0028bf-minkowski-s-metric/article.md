# Minkowski's Metric
Also called: Lgamma Loss, Lp Norm Loss

## Overview
Minkowski's Metric is a family of full-reference image quality metrics that treats an image as a 2D signal to calculate the dissimilarity between a reference image and a test image [S1, unofficial]. It generalizes the Mean Squared Error (MSE) by introducing an order parameter $\gamma$, allowing for flexible weighting of pixel-wise errors [S1, unofficial]. Unlike MSE, which is the most common distortion measure, Minkowski's metric offers a broader class of norms where $\gamma$ ranges from 1 to $\infty$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Reference image $x$, Test image $y$, Order parameter $\gamma$.
- Output: Scalar dissimilarity score $E_{\gamma}$.

**Data Structures:**
- Images $x$ and $y$ are treated as arrays of $N$ samples, where each sample corresponds to a pixel (or voxel in volume rendering contexts) value.
- $N$ is the total number of samples in the image.

**Formula:**
The dissimilarity $E_{\gamma}$ is calculated as:
$$E_{\gamma} = \left( \frac{1}{N} \sum_{i=1}^N |x_i - y_i|^{\gamma} \right)^{\frac{1}{\gamma}}$$
where:
- $x_i$ and $y_i$ are the $i$-th samples of images $x$ and $y$ respectively.
- $\gamma$ is the order parameter, with $1 \le \gamma < \infty$ [S1, unofficial].

**Special Case (MSE):**
When $\gamma = 2$, the metric relates to the Mean Squared Error (MSE). Specifically:
$$E_2 = \left( \frac{1}{N} \sum_{i=1}^N |x_i - y_i|^2 \right)^{\frac{1}{2}} = \sqrt{MSE}$$
This means $E_2$ is the square root of the MSE [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minkowski_metric(reference_image, test_image, gamma):
    """
    Calculate Minkowski's metric between two images.
    
    Args:
        reference_image (list or array): Flattened list of pixel values for reference image.
        test_image (list or array): Flattened list of pixel values for test image.
        gamma (float): Order parameter, must be >= 1.
        
    Returns:
        float: The Minkowski distance E_gamma.
    """
    # Ensure images are of the same length
    n = len(reference_image)
    if n != len(test_image):
        raise ValueError("Images must have the same number of samples.")
    
    if n == 0:
        return 0.0
        
    # Calculate the sum of absolute differences raised to the power of gamma
    sum_diff = 0.0
    for i in range(n):
        diff = abs(reference_image[i] - test_image[i])
        sum_diff += diff ** gamma
        
    # Average the sum
    avg_diff = sum_diff / n
    
    # Take the gamma-th root
    result = avg_diff ** (1.0 / gamma)
    
    return result
```

**Edge Cases and Pitfalls:**
- If $\gamma$ is very large, the metric approaches the Chebyshev distance (maximum absolute difference), which can be sensitive to outliers [S1, unofficial].
- Precision issues may arise if $\gamma$ is extremely large or small, though standard floating-point arithmetic usually handles typical values (e.g., 1, 2) well.
- The metric does not account for human visual system characteristics, correlating poorly with perceived image quality [S1, unofficial].

## Subtopics

### [Mean Square Error (MSE)](6038-mean-square-error-mse.md)
MSE is a specific case of Minkowski's metric where $\gamma=2$. It is the most commonly used mathematical distortion measure for comparing images but correlates poorly with human visual perception.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of pixels/samples in the image.
- **Memory Complexity:** $O(1)$ auxiliary space if images are processed in a streaming fashion, or $O(N)$ if images are stored in memory.
- **Strengths:** Simple to implement and compute; provides a flexible family of norms by varying $\gamma$.
- **Limitations:** Poor correlation with human visual perception; sensitive to outliers when $\gamma$ is large; does not account for structural or perceptual similarities [S1, unofficial].
- **When to use:** When a simple, mathematically grounded dissimilarity measure is needed, or when specific error characteristics (via $\gamma$) are desired. Not recommended when perceptual quality is the primary concern.

## Background
Minkowski's metric is a full-reference method used to quantify the dissimilarity between a reference image and a test image [S1, unofficial]. It is part of a broader class of mathematical distortion measures, with MSE being the most common specific instance [S1, unofficial]. The metric is derived from the Minkowski distance in mathematics, generalized to image signals.

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
