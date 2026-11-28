# Minkowski's Metric

Also called: Lgamma Loss, Lp Norm Loss

## Overview
Minkowski's Metric is a mathematical metric that treats an image as a 2D signal and calculates the dissimilarities between a reference image $x$ and a test image $y$ [S1, unofficial]. It generalizes the Mean Squared Error (MSE) by allowing the exponent $\gamma$ to vary within the range $[1, \infty)$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Reference image $x$ and test image $y$, treated as vectors of pixel values [S1, unofficial].
- **Parameter:** $\gamma \in [1, \infty)$ [S1, unofficial].
- **Output:** A scalar dissimilarity value $E_{\gamma}$ [S1, unofficial].

**Data Structures:**
- Images $x$ and $y$ are treated as 1D vectors of length $N$, where $N$ is the total number of samples (pixels) in the image [S1, unofficial].

**Formula:**
The loss is computed as [S1, unofficial]:
$$E_{\gamma} = \left( \frac{1}{N} \sum_{i=1}^N |x_i - y_i|^{\gamma} \right)^{\frac{1}{\gamma}}$$
where:
- $N$ is the number of samples in the image [S1, unofficial].
- $x_i$ and $y_i$ are the $i$-th samples of images $x$ and $y$ respectively [S1, unofficial].
- $| \cdot |$ denotes the absolute value [S1, unofficial].
- $\gamma$ is a value in the range between 1 and $\infty$ [S1, unofficial].

**Special Cases:**
- When $\gamma = 2$, the metric relates to the Mean Squared Error (MSE). Specifically, $E_2 = \sqrt{MSE}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minkowski_metric(x, y, gamma):
    # x and y are 1D lists or arrays of length N
    N = len(x)
    diff = [abs(x[i] - y[i]) for i in range(N)]
    sum_diff_gamma = sum(d ** gamma for d in diff)
    mean_diff_gamma = sum_diff_gamma / N
    E_gamma = mean_diff_gamma ** (1.0 / gamma)
    return E_gamma
```

**Edge Cases and Pitfalls:**
- **Input Alignment:** $x$ and $y$ must have the same length $N$; otherwise, the metric is undefined [G].
- **Parameter Range:** $\gamma$ must be $\ge 1$ to satisfy the triangle inequality for a norm [G].

## Subtopics

### [Mean Square Error (MSE)](6038-mean-square-error-mse.md)
Mean Square Error (MSE) is a photometric consistency loss that quantifies the dissimilarity between a rendered image and a reference image. It is a specific case of Minkowski's Metric where $\gamma=2$, defined by summing squared differences without the square root. While it correlates poorly with human visual perception, it remains the most commonly used metric for this purpose.

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
Not given in the sources.

## Background
Minkowski's Metric originates from mathematical distance measures and is used to calculate dissimilarities between reference and test images [S1, unofficial]. In volume rendering, it serves as a photometric loss function [S1, unofficial]. The RGB Loss variant applies the L1 norm to enforce color consistency across different density representations in neural scene reconstruction [S2].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
