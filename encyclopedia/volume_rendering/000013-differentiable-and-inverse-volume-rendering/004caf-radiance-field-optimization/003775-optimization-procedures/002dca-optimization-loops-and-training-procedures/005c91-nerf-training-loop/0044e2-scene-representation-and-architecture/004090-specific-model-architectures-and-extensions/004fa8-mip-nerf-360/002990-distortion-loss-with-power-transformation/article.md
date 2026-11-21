# Distortion Loss with Power Transformation

Also called: Power-transformed distortion loss

## Overview
A modification of the mip-NeRF 360 distortion loss that applies a power transformation to metric distance before computing the loss [S1]. This curves metric distance to aggressively penalize "floaters" (spurious geometry artifacts) near the camera, which significantly improves the quality of rendered videos [S1]. Unlike the standard distortion loss, which treats metric distance linearly, this method maps distance into a normalized space where the gradient is steep near the origin and tapers off, resembling log-distance behavior [S1].

## How it works
The method modifies the computation of the distortion loss term in the training objective by transforming the metric distance variable $t$ before integrating the loss.

**Inputs:**
- Metric distance $t$ for each sample on a ray.
- Transformation parameters: scaling factor $s$ and power exponent $\lambda$.
- Distortion loss multiplier $\alpha$.

**Outputs:**
- Transformed distortion loss value.

**Data Structures:**
- No specific data structures are required beyond the standard ray sampling buffers (positions, weights, distances) used in differentiable volume rendering.

**Steps:**
1. Retrieve the metric distance $t$ for a sample.
2. Scale the metric distance by a factor $s$.
3. Apply the power transformation $P$ to the scaled distance:
   $$ t' = P(t) = (s \cdot t)^{\lambda} $$
   Where $s$ is the scaling multiplier and $\lambda$ is the power exponent.
4. Compute the standard distortion loss using the transformed distance $t'$ instead of the raw metric distance $t$. The standard distortion loss typically involves integrating the absolute difference between the weighted average of distances and the individual distances, weighted by the transmittance and density contributions.
5. Multiply the resulting distortion loss by the multiplier $\alpha$ and add it to the total training loss.

**Parameters:**
- Scaling factor $s$: 10000 [S1].
- Power exponent $\lambda$: -0.25 [S1].
- Distortion loss multiplier $\alpha$: 0.005 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_power_transformed_distortion_loss(distances, weights, alpha=0.005, s=10000.0, lambda_val=-0.25):
    """
    Computes the distortion loss using power-transformed metric distances.
    
    Args:
        distances: Array of metric distances t for samples on a ray.
        weights: Array of normalized weights (density * transmittance) for samples.
        alpha: Multiplier for the distortion loss term.
        s: Scaling factor for the power transformation.
        lambda_val: Power exponent for the transformation.
        
    Returns:
        The computed distortion loss value.
    """
    # Step 2: Scale the metric distance
    scaled_distances = s * distances
    
    # Step 3: Apply power transformation P(t) = (s * t)^lambda
    # Note: Ensure distances are non-negative to avoid complex numbers if lambda is fractional
    transformed_distances = scaled_distances ** lambda_val
    
    # Step 4: Compute standard distortion loss on transformed distances
    # Standard distortion loss is typically: sum(w_i * |t_i - sum(w_j * t_j)|)
    # Here, we use transformed_distances as t_i
    
    # Calculate the weighted average of transformed distances
    # weights must be normalized such that sum(weights) == 1.0 for this to be a valid expectation
    mean_transformed_distance = np.sum(weights * transformed_distances)
    
    # Compute the absolute difference between each transformed distance and the mean
    abs_diffs = np.abs(transformed_distances - mean_transformed_distance)
    
    # Weight the differences by the sample weights
    distortion_loss = np.sum(weights * abs_diffs)
    
    # Step 5: Multiply by alpha
    final_loss = alpha * distortion_loss
    
    return final_loss
```

**Edge Cases and Pitfalls:**
- If $t=0$, $t^\lambda$ with $\lambda < 0$ results in infinity. In practice, metric distances are strictly positive for samples behind the camera, but care must be taken if samples are exactly at the camera origin. A small epsilon might be needed, though the source does not specify this [G].
- The transformation is non-linear, so the interpretation of the loss value changes. It is not directly comparable to the standard distortion loss magnitude.

## Subtopics

### [Gradient Scaling](6887-gradient-scaling.md)
Gradient Scaling is a technique used in differentiable volume rendering to mitigate near-camera overfitting and the formation of "floaters" (spurious geometry artifacts) during radiance field optimization. It addresses the problem of gradient imbalance along rays, where samples far from the camera receive significantly smaller gradients than those near the camera due to the exponential decay of transmittance. Unlike [Distortion Loss with Power Transformation](4560-distortion-loss-with-power-transformation.md), which modifies the loss function to penalize metric distance, Gradient Scaling modifies the optimization process itself by rescaling gradients during backpropagation to ensure uniform gradient magnitude across all samples on a ray.

### [Weight Regularization](13442-weight-regularization.md)
Weight regularization is a loss term applied to neural radiance fields (NeRFs) to suppress geometric noise and cloud-like semi-transparent artifacts that arise when training alters density distributions. It encourages a concentrated density distribution by penalizing pairs of samples with scattered large weights, forcing rays to either have zero weights along the entire ray or consolidate all weights into as small an area as possible [2, unofficial]. Unlike mip-NeRF 360, which optimizes distances to reduce floaters, this method explicitly penalizes weight scattering to maintain clean geometry during optimization.

## Operating characteristics
- **Strengths:** Significantly improves the quality of rendered videos by reducing "floaters" near the camera [S1].
- **Limitations:** Does not significantly improve test-set metrics on the 360 dataset, as floaters do not tend to contribute much to error metrics computed on still images [S1].
- **Performance:** The transformation is computationally inexpensive, involving only basic arithmetic operations.
- **When to use:** When video quality is a priority and floaters near the camera are a noticeable artifact.
- **When not to use:** When optimizing for standard image-based metrics (e.g., PSNR, SSID) on static datasets, as the improvement is negligible [S1].

## Background
The power transformation was introduced in the context of mip-NeRF 360 and its extensions, specifically in the Zip-NeRF paper [S1]. It was developed to address the issue of "floaters" near the camera in unbounded scene rendering. The method builds on the distortion loss introduced in mip-NeRF 360, adapting it to be more sensitive to errors close to the camera by curving the metric distance space [S1].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
