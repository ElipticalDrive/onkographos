# Augmented Features (Avg, Var, Color)


## Overview
Augmented Features (Avg, Var, Color) enhances neural radiance fields for 3D object detection by augmenting the feature or cost volume with mean, variance, and color information [S1]. It addresses insufficient geometric detail in standard NeRF opacity fields by explicitly leveraging statistical moments and low-level color cues as geometry priors [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images, camera poses, and a base NeRF representation predicting opacity $\sigma$ and color $\mathbf{c}$ [S1].
*   **Outputs:** An augmented feature volume containing mean, variance, and color features for downstream 3D detection [S1].

**Data Structures:**
The method augments the cost volume or feature volume with three components per voxel/point [S1]:
1.  **Mean Volume:** The average feature representation along the ray or volume [S1].
2.  **Variance Volume:** The variance of features, explicitly used as a geometry prior [S1].
3.  **Color Volume:** Low-level color information integrated into the feature set [S1].

**Steps:**
1.  **Base NeRF Rendering:** Render the scene using a NeRF branch that models the opacity field with an MLP and a custom opacity function instead of a standard sigmoid [S1].
2.  **Feature Aggregation:** Compute the mean (average) features for each point along the ray [S1].
3.  **Variance Computation:** Compute the variance of the features along the ray to serve as a geometry prior [S1].
4.  **Color Augmentation:** Incorporate low-level color information into the feature set [S1].
5.  **Volume Integration:** Render the augmented features using standard volume integration [S1].
6.  **Training Note:** The augmented features are used for 3D object detection, but the detection branch does not benefit the NeRF branch; disabling it improves novel view synthesis by 0.43 dB [S1].

**Formulas:**
The rendered feature $\hat{\mathbf{F}}(r)$ for a ray $r$ follows the volume integration principle from the parent [Feature Fields](12995-feature-fields.md):
$$
\hat{\mathbf{F}}(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{f}(x_i)
$$
where $\mathbf{f}(x_i)$ contains the augmented mean, variance, and color features [S1]. The accumulated transmittance is $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and the opacity weight is $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [12995-feature-fields.md]. The opacity $\sigma$ is modeled using an MLP and a specific opacity function rather than a sigmoid [S1].

**Parameters:**
*   **Opacity Function:** Uses an MLP and a non-sigmoid opacity function [S1].
*   **Detection Branch:** A detection branch is used for 3D object detection but does not improve the NeRF branch [S1].

**Edge Cases and Pitfalls:**
*   **Detection Branch Interference:** The detection branch may erase low-level details needed for the NeRF branch, as disabling it yields a 0.43 dB improvement in novel view synthesis [S1].
*   **Hardware/Precision:** Evaluated on a V100 GPU [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_augmented_features(points, deltas, opacities, colors, features):
    """
    Renders mean, variance, and color features along a ray using volume integration.
    Follows the parent volume rendering formula [12995-feature-fields.md].
    """
    N = len(points)
    if N == 0:
        return [], [], []

    # Compute opacity weights alpha_i = 1 - exp(-sigma_i * delta_i)
    alphas = [1.0 - math.exp(-opacities[i] * deltas[i]) for i in range(N)]
    
    # Compute accumulated transmittance T_i
    T = [1.0] * N
    for i in range(1, N):
        T[i] = T[i-1] * math.exp(-opacities[i-1] * deltas[i-1])
        
    # Compute rendering weights w_i = T_i * alpha_i
    weights = [T[i] * alphas[i] for i in range(N)]
    total_weight = sum(weights)
    
    if total_weight == 0:
        return [], [], []

    dim = len(features[0])
    mean_feat = [0.0] * dim
    color_feat = [0.0, 0.0, 0.0]
    
    # First pass: compute weighted mean
    for i in range(N):
        w = weights[i] / total_weight
        for k in range(dim):
            mean_feat[k] += w * features[i][k]
        color_feat[0] += w * colors[i][0]
        color_feat[1] += w * colors[i][1]
        color_feat[2] += w * colors[i][2]
        
    # Second pass: compute variance around the weighted mean
    var_feat = [0.0] * dim
    for i in range(N):
        w = weights[i] / total_weight
        for k in range(dim):
            diff = features[i][k] - mean_feat[k]
            var_feat[k] += w * (diff * diff)
            
    return mean_feat, var_feat, color_feat
```

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the Waymo Open Dataset, the method achieves 61.48 mAP@.25 and 25.45 mAP@.50 with a latency of 0.554 s on a V100 GPU [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Variance features serve as a geometry prior, and incorporating color information provides additional geometry cues [S1]. The authors claim introducing variance improves performance by over 0.7 mAP@.25 and 0.6 mAP@.50 compared to using only average features [S1]. However, the ablation table reports Avg=50.1, Avg+Var=49.7, and Avg+Var+Color=50.1, which contradicts the text's claim if Avg is the baseline [S1].
*   **Limitations:** The detection branch does not benefit the NeRF branch and may erase low-level details needed for novel view synthesis, as disabling it improves performance by 0.43 dB [S1].
*   **When to use:** Multi-view 3D object detection where geometric priors from variance and color are desired [S1].
*   **When not:** When novel view synthesis quality is the primary objective, due to the interference from the detection branch [S1].

## Background
*   **Origin:** Introduced in "NeRF-Det: Learning Geometry-Aware Volumetric Representation for Multi-View 3D Object Detection" [S1].
*   **History and Development:** Builds on the NeRF formulation and integrates it with 3D object detection by augmenting the cost volume with mean, variance, and color features to improve scene geometry modeling [S1].
*   **Where it is used:** Multi-view 3D object detection [S1].

## Sources
[S1] NeRF-Det: Learning Geometry-Aware Volumetric Representation for Multi-View 3D Object Detection, https://doi.org/10.1109/iccv51070.2023.02131, fetched 2026-10-30
