# Distortion Loss with Power Transformation

Also called: Mip-NeRF 360 Distortion Loss

## Overview
A modification of the mip-NeRF 360 distortion loss that applies a power transformation to metric distance before computing the loss [S1]. This curves metric distance to aggressively penalize "floaters" (spurious geometry artifacts) near the camera, which significantly improves the quality of rendered videos [S1]. Unlike the standard distortion loss, which treats metric distance linearly, this method maps distance into a normalized space where the gradient is steep near the origin and tapers off, resembling log-distance behavior [S1].

## How it works
The method modifies the computation of the distortion loss term in the training objective by transforming the metric distance variable $t$ before integrating the loss [S1].

**Inputs:**
- Metric distance $t$ for each sample on a ray [S1].
- Transformation parameters: scaling factor $s$ and power exponent $\lambda$ [S1].
- Distortion loss multiplier $\alpha$ [S1].

**Outputs:**
- Transformed distortion loss value [S1].

**Data Structures:**
- No specific data structures are required beyond the standard ray sampling buffers (positions, weights, distances) used in differentiable volume rendering [G].

**Steps:**
1. Retrieve the metric distance $t$ for a sample [S1].
2. Scale the metric distance by a factor $s$ [S1].
3. Apply the power transformation $P$ to the scaled distance:
   $$ t' = P(t) = (s \cdot t)^{\lambda} $$
   Where $s$ is the scaling multiplier and $\lambda$ is the power exponent [S1].
4. Compute the standard distortion loss using the transformed distance $t'$ instead of the raw metric distance $t$ [S1]. The standard distortion loss typically involves integrating the absolute difference between the weighted average of distances and the individual distances, weighted by the transmittance and density contributions [G].
5. Multiply the resulting distortion loss by the multiplier $\alpha$ and add it to the total training loss [S1].

**Parameters:**
- Scaling factor $s$: 10000 [S1].
- Power exponent $\lambda$: -0.25 [S1].
- Distortion loss multiplier $\alpha$: 0.005 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_power_transformed_distortion_loss(distances, weights, alpha=0.005, s=10000.0, lambda_val=-0.25):
    # Step 2: Scale the metric distance
    scaled_distances = [s * t for t in distances]
    
    # Step 3: Apply power transformation P(t) = (s * t)^lambda
    # Guard against t=0 to avoid infinity when lambda_val < 0 [G]
    transformed_distances = [(s * t + 1e-8) ** lambda_val for t in distances]
    
    # Step 4: Compute standard distortion loss on transformed distances
    # Standard distortion loss: sum(w_i * |t_i - sum(w_j * t_j)|)
    mean_transformed_distance = sum(w * t for w, t in zip(weights, transformed_distances))
    
    distortion_loss = sum(w * abs(t - mean_transformed_distance) for w, t in zip(weights, transformed_distances))
    
    # Step 5: Multiply by alpha
    final_loss = alpha * distortion_loss
    
    return final_loss
```

**Edge Cases and Pitfalls:**
- If $t=0$, $t^\lambda$ with $\lambda < 0$ results in infinity [G]. In practice, metric distances are strictly positive for samples behind the camera, but care must be taken if samples are exactly at the camera origin [G].
- The transformation is non-linear, so the interpretation of the loss value changes [G]. It is not directly comparable to the standard distortion loss magnitude [G].

## Subtopics

### [Gradient Scaling](6887-gradient-scaling.md)
Gradient Scaling is a technique used in differentiable volume rendering to mitigate near-camera overfitting and the formation of "floaters" during radiance field optimization. It addresses gradient imbalance along rays, where samples far from the camera receive significantly smaller gradients than those near the camera due to the exponential decay of transmittance. Unlike [Distortion Loss with Power Transformation](4560-distortion-loss-with-power-transformation.md), which modifies the loss function to penalize metric distance, Gradient Scaling modifies the optimization process itself by rescaling gradients during backpropagation to ensure uniform gradient magnitude across all samples on a ray. Choose Gradient Scaling when gradient magnitude uniformity is the primary bottleneck, rather than metric distance penalization.

### [Weight Regularization](13442-weight-regularization.md)
Weight regularization is a loss term applied to neural radiance fields to suppress geometric noise and cloud-like semi-transparent artifacts that arise when training alters density distributions. It encourages a concentrated density distribution by penalizing pairs of samples with scattered large weights, forcing rays to either have zero weights along the entire ray or consolidate all weights into as small an area as possible. Unlike [Distortion Loss with Power Transformation](4560-distortion-loss-with-power-transformation.md), which optimizes distances to reduce floaters, Weight Regularization explicitly penalizes weight scattering to maintain clean geometry during optimization. Choose Weight Regularization when suppressing transparent artifacts and enforcing bimodal opacity distributions is required.

## Operating characteristics
- **Strengths:** Significantly improves the quality of rendered videos by reducing "floaters" near the camera [S1].
- **Limitations:** Does not significantly improve test-set metrics on the 360 dataset, as floaters do not tend to contribute much to error metrics computed on still images [S1].
- **Performance:** The transformation is computationally inexpensive, involving only basic arithmetic operations [G]. Adjusting the distortion weight controls the trade-off between quality and speed: higher weights concentrate opacity onto thinner, more opaque surfaces, speeding up rendering by allowing rays to terminate early [S2, unofficial]. The method operates at a marked point that keeps near-peak quality while capturing most of the speedup [S2, unofficial].
- **When to use:** When video quality is a priority and floaters near the camera are a noticeable artifact [G].
- **When not to use:** When optimizing for standard image-based metrics (e.g., PSNR, SSIM) on static datasets, as the improvement is negligible [S1].

## Background
The power transformation was introduced in the context of mip-NeRF 360 and its extensions, specifically in the Zip-NeRF paper [S1]. It was developed to address the issue of "floaters" near the camera in unbounded scene rendering [S1]. The method builds on the distortion loss introduced in mip-NeRF 360, adapting it to be more sensitive to errors close to the camera by curving the metric distance space [S1].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] Differentiable Voronoi Ray Tracing Beyond Rasterization Speeds, https://arxiv.org/abs/2608.17682, fetched 2026-10-27 (unofficial)
