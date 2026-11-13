# Distortion Loss with Power Transformation

Also called: Power-transformed distortion loss, Curved distortion loss

## Overview
Distortion Loss with Power Transformation is a regularization technique used in differentiable volume rendering to suppress "floaters"—spurious, semi-transparent geometry artifacts that appear near the camera during radiance field optimization [S1]. It modifies the standard mip-NeRF 360 distortion loss by applying a non-linear power transformation to the metric distance before computing the penalty [S1]. This transformation curves metric distance into a normalized space with a steep gradient near the origin, aggressively penalizing samples close to the camera while tapering off for distant samples [S1]. Unlike the standard distortion loss, which may ignore near-camera floaters due to the exponential decay of transmittance, this method ensures that histogram bins near the camera are correctly penalized [S1].

## How it works
The method modifies the computation of the distortion loss by introducing a power transformation function $P$ applied to the metric distance $t$.

**Inputs and Outputs:**
*   **Input:** Metric distance $t$ of a sample point along a ray.
*   **Output:** Transformed distance $t'$ used in the distortion loss calculation.

**Data Structures:**
No specific data structures are required beyond the standard ray sampling buffers. The transformation is applied element-wise to the distance values.

**Parameters:**
*   **Multiplier ($\lambda_{dist}$):** The weight of the distortion loss term in the total objective. Set to $0.005$ in all experiments [S1].
*   **Transformation Parameters:** The power transformation $P$ is defined by a scaling factor $s$ and an exponent $\alpha$. In the cited experiments, $s = 10000$ and $\alpha = -0.25$ [S1].

**Algorithm:**
1.  Retrieve the metric distance $t$ for a sample point.
2.  Scale the distance by the factor $s$: $t_{scaled} = s \cdot t$.
3.  Apply the power transformation with exponent $\alpha$: $t' = P(t_{scaled}) = t_{scaled}^{\alpha}$.
    *   Note: The source describes this as curving metric distance to have a steep gradient near the origin [S1]. The specific function form implied by "power transformation" with parameters $(10000x, -0.25)$ is $P(x) = x^{-0.25}$ [S1].
4.  Compute the distortion loss using the transformed distances $t'$ instead of the original metric distances $t$. The standard distortion loss formula involves the weights $w_k$ and distances $t_k$:
    $$ L_{dist} = \sum_k w_k |t_k - \sum_j w_j t_j| $$
    In this variant, $t_k$ and $t_j$ are replaced by their transformed counterparts $t'_k$ and $t'_j$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_power_transformed_distortion_loss(weights, distances, s=10000.0, alpha=-0.25):
    """
    Computes the distortion loss using power-transformed metric distances.
    
    Args:
        weights: Array of sample weights w_k (contribution to final color).
        distances: Array of metric distances t_k.
        s: Scaling factor for the power transformation.
        alpha: Exponent for the power transformation.
        
    Returns:
        The scalar distortion loss value.
    """
    # Step 1: Scale the metric distances
    distances_scaled = distances * s
    
    # Step 2: Apply power transformation
    # P(x) = x^alpha
    # Note: If distances_scaled is 0, this results in infinity. 
    # In practice, a small epsilon or clamping might be needed, 
    # but the source does not specify this guard [G].
    transformed_distances = distances_scaled ** alpha
    
    # Step 3: Compute the expected transformed distance (mean)
    # E[t'] = sum(w_k * t'_k)
    expected_transformed_distance = sum(weights * transformed_distances)
    
    # Step 4: Compute the distortion loss
    # L_dist = sum(w_k * |t'_k - E[t']|)
    distortion_loss = sum(weights * abs(transformed_distances - expected_transformed_distance))
    
    return distortion_loss
```

**Edge Cases and Pitfalls:**
*   **Singularity at Zero:** The power transformation $x^{-0.25}$ is undefined at $x=0$. If a sample has metric distance $t=0$ (at the camera origin), the transformed distance becomes infinite. The source does not specify how this is handled, but it is a critical implementation detail [G].
*   **Impact on Metrics:** The source notes that this modification does not significantly improve test-set metrics on still images because floaters do not contribute much to error metrics computed on static views [S1]. Its primary benefit is visual quality in rendered videos [S1].

## Subtopics

### [Gradient Scaling](6887-gradient-scaling.md)
Gradient Scaling is a technique used in differentiable volume rendering to mitigate near-camera overfitting and the formation of "floaters" during radiance field optimization. It addresses the problem of gradient imbalance along rays, where samples far from the camera receive significantly smaller gradients than those near the camera due to the exponential decay of transmittance. Unlike Distortion Loss with Power Transformation, which modifies the loss function to penalize metric distance, Gradient Scaling modifies the optimization process itself by rescaling gradients during backpropagation to ensure uniform gradient magnitude across all samples on a ray.

## Operating characteristics
*   **Time Complexity:** The addition of the power transformation adds negligible computational overhead to the distortion loss calculation, as it involves only basic arithmetic operations per sample [S1].
*   **Memory Complexity:** No additional memory is required beyond storing the transformed distances if computed on the fly, or minimal temporary storage [S1].
*   **Performance:** The method significantly improves the visual quality of rendered videos by suppressing near-camera floaters [S1]. However, it does not significantly improve quantitative test-set metrics on still images [S1].
*   **Strengths:** Effectively targets the specific artifact of near-camera floaters by curving the distance metric to have a steep gradient near the origin [S1].
*   **Limitations:** Does not improve standard image-based metrics; requires careful handling of the singularity at $t=0$ [S1].

## Background
The Distortion Loss with Power Transformation was introduced in the context of mip-NeRF 360 extensions, specifically in the Zip-NeRF paper [S1]. It builds upon the distortion loss regularization proposed in mip-NeRF 360 [S1]. The motivation was to improve the quality of rendered videos by addressing floaters that are not well-penalized by the standard distortion loss due to the nature of transmittance decay [S1].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] NeRF-Art: Text-Driven Neural Radiance Fields Stylization, https://doi.org/10.1109/tvcg.2023.3283400, fetched 2026-10-26
