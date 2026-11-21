# Minkowski's Metric

Also called: Lgamma Loss, Lp Norm Loss

## Overview
Minkowski's Metric (also known as the Lp norm or Lgamma loss) is a family of photometric consistency losses used in radiance-field optimization to quantify the dissimilarity between a rendered image and a ground-truth reference [S1, unofficial]. It generalizes the Mean Squared Error (MSE) by allowing the exponent $\gamma$ to vary, enabling a trade-off between sensitivity to large errors and robustness to outliers [S1, unofficial]. Unlike MSE, which penalizes large deviations quadratically, Minkowski's Metric with $\gamma < 2$ (e.g., L1) is less sensitive to outliers, while $\gamma > 2$ increases sensitivity to large errors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Two images (or image patches) represented as vectors of pixel values: reference image $x$ and test image $y$.
- **Parameter:** $\gamma \in [1, \infty)$, the order of the norm [S1, unofficial].
- **Output:** A scalar loss value $E_{\gamma}$ representing the dissimilarity [S1, unofficial].

**Data Structures:**
- Images $x$ and $y$ are treated as 1D vectors of length $N$, where $N$ is the total number of samples (pixels) in the image [S1, unofficial].

**Formula:**
The loss is computed as [S1, unofficial]:
$$E_{\gamma} = \left( \frac{1}{N} \sum_{i=1}^N |x_i - y_i|^{\gamma} \right)^{\frac{1}{\gamma}}$$
where:
- $N$ is the number of samples in the image [S1, unofficial].
- $x_i$ and $y_i$ are the $i$-th samples (pixel values) of images $x$ and $y$ respectively [S1, unofficial].
- $| \cdot |$ denotes the absolute value [S1, unofficial].
- $\gamma$ is the exponent parameter [S1, unofficial].

**Special Cases:**
- When $\gamma = 2$, the metric relates to the Mean Squared Error (MSE). Specifically, $E_2 = \sqrt{MSE}$ [S1, unofficial]. Note that MSE is typically defined as the squared term without the root, i.e., $MSE = \frac{1}{N} \sum_{i=1}^N (x_i - y_i)^2$. Thus, $E_2$ is the root-mean-square error (RMSE) [S1, unofficial].
- When $\gamma = 1$, the metric becomes the Mean Absolute Error (MAE) or L1 loss [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minkowski_metric(x, y, gamma):
    """
    Computes Minkowski's Metric (Lgamma loss) between two images x and y.
    
    Args:
        x: Tensor of shape (N,) or (H, W, C) flattened, reference image values.
        y: Tensor of shape (N,) or (H, W, C) flattened, test image values.
        gamma: Float, the exponent parameter (>= 1).
        
    Returns:
        Scalar loss value E_gamma.
    """
    # Ensure inputs are flattened to 1D vectors of length N
    # [G] Assuming x and y are tensors of the same shape.
    # Flatten to handle multi-channel images if necessary.
    x_flat = x.flatten()
    y_flat = y.flatten()
    
    N = x_flat.shape[0]
    
    # Compute absolute differences
    diff = torch.abs(x_flat - y_flat)
    
    # Raise to the power of gamma
    diff_gamma = torch.pow(diff, gamma)
    
    # Compute mean
    mean_diff_gamma = torch.mean(diff_gamma)
    
    # Raise to the power of 1/gamma
    E_gamma = torch.pow(mean_diff_gamma, 1.0 / gamma)
    
    return E_gamma
```

**Edge Cases and Pitfalls:**
- **Precision:** For large $\gamma$, the term $|x_i - y_i|^\gamma$ can lead to numerical overflow if differences are large. Clipping or normalization of input values may be necessary [G].
- **Boundary Conditions:** $\gamma$ must be $\ge 1$ for the metric to satisfy the triangle inequality (norm property). Values $< 1$ are quasi-norms and may cause optimization issues due to non-convexity [G].
- **Degenerate Input:** If $x$ and $y$ have different sizes, the metric is undefined. Inputs must be aligned [G].

## Subtopics

### [Mean Square Error (MSE)](6038-mean-square-error-mse.md)
Mean Square Error (MSE) is a photometric consistency loss used in radiance-field optimization to quantify the dissimilarity between a rendered image and a ground-truth reference. It serves as the squared case of Minkowski's Metric, providing a differentiable objective function for gradient-based optimization. Unlike perceptual metrics, MSE operates purely on pixel-wise intensity differences, making it computationally efficient but potentially misaligned with human visual perception.

### RGB Loss (Lc)
RGB Loss (Lc) is a variant used in ray-distance volume rendering for neural scene reconstruction. It comprises two components: color loss with density derived from SRDF ($\mathcal{L}_c^{\text{SRDF}}$) and color loss with density derived from SDF ($\mathcal{L}_c^{\text{SDF}}$) [S2]. The total loss is the sum of these two components [S2].

**Formula:**
$$\mathcal{L}_c = \mathcal{L}_c^{\text{SRDF}} + \mathcal{L}_c^{\text{SDF}}$$
$$\mathcal{L}_c^{\text{SRDF}} = \frac{1}{N_s} \sum_{\mathbf{r} \in \mathcal{S}^2} \| \mathbf{C}^{\text{SRDF}} - \bar{\mathbf{C}} \|_1$$
$$\mathcal{L}_c^{\text{SDF}} = \frac{1}{N_s} \sum_{\mathbf{r} \in \mathcal{S}^2} \| \mathbf{C}^{\text{SDF}} - \bar{\mathbf{C}} \|_1$$
where:
- $\mathcal{S}^2$: Set of rays in the current minibatch [S2].
- $\bar{\mathbf{C}}$: Ground truth color image [S2].
- $\mathbf{C}^{\text{SRDF}}$: Rendered color using density derived from SRDF (computed per Eq. 5 in original paper) [S2].
- $\mathbf{C}^{\text{SDF}}$: Rendered color using density derived from SDF (computed per Eq. 3 in original paper) [S2].
- $N_s$: Number of points sampled per minibatch [S2].
- $\| \cdot \|_1$: L1 norm [S2].

This variant specifically uses the L1 norm (a case of Minkowski's Metric with $\gamma=1$) for color consistency, applied to colors rendered from two different density representations (SRDF and SDF) [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of pixels/samples. The computation involves element-wise operations and a sum, which are linear in the number of samples [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store the input images and intermediate differences [G].
- **Performance:** MSE ($\gamma=2$) is the most commonly used variant due to its differentiability and ease of optimization [S1, unofficial]. L1 ($\gamma=1$) is more robust to outliers but may produce blurrier results [G].
- **Strengths:** Simple to implement, differentiable, and computationally efficient [S1, unofficial].
- **Limitations:** MSE is sensitive to outliers and may not align well with human visual perception [S1, unofficial]. L1 may lead to less smooth reconstructions [G].
- **When to Use:** Use MSE for general photometric consistency when outliers are not a major concern. Use L1 when robustness to outliers is desired. Use higher $\gamma$ values if large errors need to be penalized more heavily [G].

## Background
Minkowski's Metric originates from mathematical distance measures and has been widely adopted in image processing and computer vision for assessing image quality [S1, unofficial]. In the context of volume rendering and radiance-field optimization, it serves as a fundamental photometric loss function. The MSE variant, corresponding to $\gamma=2$, is particularly prevalent due to its historical use in signal processing and its favorable optimization properties [S1, unofficial]. The RGB Loss variant, using L1 norm, has been applied in specific neural scene reconstruction frameworks like Ray-Distance Volume Rendering to enforce color consistency across different density representations [S2].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
