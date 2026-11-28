# Pearson's Correlation Coefficient (PCC)

## Overview
Pearson's Correlation Coefficient (PCC) is an image fidelity score used to evaluate the quality of 3D reconstructions and novel 2D views by measuring the linear correlation between rendered pixels and ground truth [S1, unofficial]. It complements pixel-wise error metrics like PSNR and MSE by assessing how well the relative intensity variations are preserved, rather than just absolute values [S1, unofficial].

## How it works
**Inputs:**
- Rendered image vector $\mathbf{r}$ of length $N$ (flattened pixel intensities).
- Ground truth image vector $\mathbf{t}$ of length $N$.

**Data Structures:**
- Vectors $\mathbf{r}$ and $\mathbf{t}$ containing floating-point intensity values.

**Steps:**
1. Compute the mean of the rendered image: $\bar{r} = \frac{1}{N} \sum_{i=1}^{N} r_i$.
2. Compute the mean of the ground truth image: $\bar{t} = \frac{1}{N} \sum_{i=1}^{N} t_i$.
3. Compute the covariance of $\mathbf{r}$ and $\mathbf{t}$:
   $$ \text{cov}(\mathbf{r}, \mathbf{t}) = \frac{1}{N} \sum_{i=1}^{N} (r_i - \bar{r})(t_i - \bar{t}) $$
4. Compute the standard deviation of $\mathbf{r}$:
   $$ \sigma_r = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (r_i - \bar{r})^2} $$
5. Compute the standard deviation of $\mathbf{t}$:
   $$ \sigma_t = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (t_i - \bar{t})^2} $$
6. Calculate PCC:
   $$ \text{PCC} = \frac{\text{cov}(\mathbf{r}, \mathbf{t})}{\sigma_r \sigma_t} $$

**Parameters:**
- $N$: Total number of pixels.

**Edge Cases:**
- If $\sigma_r = 0$ or $\sigma_t = 0$ (constant intensity images), the denominator is zero. The metric is undefined or requires a guard value [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pearson_correlation_coefficient(rendered: list[float], ground_truth: list[float]) -> float:
    n = len(rendered)
    if n == 0:
        return 0.0
    
    mean_r = sum(rendered) / n
    mean_t = sum(ground_truth) / n
    
    cov = 0.0
    var_r = 0.0
    var_t = 0.0
    
    for i in range(n):
        diff_r = rendered[i] - mean_r
        diff_t = ground_truth[i] - mean_t
        
        cov += diff_r * diff_t
        var_r += diff_r * diff_r
        var_t += diff_t * diff_t
        
    std_r = var_r ** 0.5
    std_t = var_t ** 0.5
    
    if std_r == 0 or std_t == 0:
        # Guard against division by zero for constant images [G]
        return 0.0 if std_r == 0 and std_t == 0 else float('nan')
        
    return cov / (std_r * std_t)
```

## Subtopics

### Diffuse Correlation Metric
A quality metric based on the Pearson correlation coefficient between ground-truth diffuse lighting and approximated diffuse lighting to detect variability in reconstruction quality [S2]. It calculates the ground-truth diffuse component as $\mathbf{n}_i \cdot \mathbf{l}$ and the approximation as $\hat{\mathbf{n}}_i \cdot \mathbf{l}$, then computes the Pearson coefficient between these two vectors [S2]. The light direction $\mathbf{l}$ is typically set to $(1, 1, 1)$ [S2]. This helps detect different degrees of variability among reconstruction methods where standard error metrics might fail to represent sample variance [S2].

## Operating characteristics
PCC ranges from -1 to 1, where 1 indicates perfect linear correlation [S1, unofficial]. It is sensitive to linear scaling and bias but invariant to them, meaning it measures structural similarity in intensity trends rather than absolute accuracy [S1, unofficial]. It is computationally efficient, requiring $O(N)$ time and memory for $N$ pixels [S1, unofficial]. It is used alongside SSIM, PSNR, and MSE to evaluate sparse-view 3D CBCT reconstruction quality [S1, unofficial].

## Background
PCC was introduced as an image fidelity score for evaluating 3D reconstructions and novel 2D views in the context of differentiable voxel-based X-ray rendering [S1, unofficial]. It is part of a suite of metrics including SSIM, PSNR, and MSE [S1, unofficial]. The Diffuse Correlation Metric variant was derived to address the inability of other metrics to represent the variance of samples in gradient estimation for volume rendering on unstructured meshes [S2].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
[S2] A Comparison of Gradient Estimation Methods for Volume Rendering on Unstructured Meshes, https://www.computer.org/csdl/journal/tg/2011/03/ttg2011030305/13rRUx0xPi5, fetched 2026-11-01
