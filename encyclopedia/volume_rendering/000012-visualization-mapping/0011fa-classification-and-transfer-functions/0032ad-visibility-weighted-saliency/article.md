# Visibility-Weighted Saliency

Also called: Saliency Field Computation, VWS metric, Visibility-weighted saliency field, conspicuity metric

## Overview
Visibility-Weighted Saliency (VWS) is a metric that quantifies the perceptual conspicuity of features in volume rendered images by combining feature visibility (contribution to the final 2D image) and local distinctiveness (saliency relative to surroundings) [S1, unofficial][S2]. Unlike feature visibility, which only measures occlusion, VWS accounts for how much a feature stands out from its context, solving the problem of objectively comparing transfer functions and viewpoints [S1, unofficial]. It serves as a quantitative basis for optimizing transfer functions to match user-defined target conspicuity [S2].

## How it works
**Inputs:**
*   Volume data $V$ with voxel values.
*   Transfer function defining opacity $\alpha_i$ and color for each voxel $i$.
*   Viewpoint (camera position and direction).
*   User-defined feature sets $F \subseteq V$ (via segmentation or transfer function ranges).
*   User-defined target importance $t_i$ for each feature $i$ (normalized, $\sum t_i = 1$).
*   Parameters: neighborhood radius $\sigma$, scale weights $w_1, w_2$, attribute weights $u_1, u_2$.

**Outputs:**
*   Visibility Field $V = \{v_i\}$.
*   Saliency Field $L$.
*   Visibility-Weighted Saliency Field $S = \{s_i\}$.
*   Feature Saliency Score $W_F$ (scalar in $[0,1]$).
*   Objective Function Value $F$ (for optimization).

**Data Structures:**
*   **Volume Grid:** 3D array of voxel values.
*   **Appearance Attributes:** Fields for Brightness ($O_b$) and Saturation ($O_s$) derived from voxel colors in CIELCh or CIELab space [S1, unofficial][S2].
*   **Visibility Field:** 3D array of floats $v_i \in [0,1]$.
*   **Saliency Field:** 3D array of floats $L(O, i, \sigma)$.

**Steps:**

1.  **Compute Appearance Attributes:**
    Convert voxel colors to a perceptually uniform color space (CIELCh or CIELab) to extract Brightness ($L^*$) and Saturation/Chroma ($C^*$ or $S$) [S1, unofficial][S2]. Let $O_b$ be the brightness field and $O_s$ be the saturation field [S2].

2.  **Compute Visibility Field ($v_i$):**
    Perform front-to-back volume rendering (slice-based or ray-casting) to compute accumulated opacity $A_i$ for each voxel $i$ along the view ray [S1, unofficial][S2].
    $$v_i = A_i - A_{i-1} = (1 - A_{i-1}) \alpha_i$$
    where $\alpha_i$ is the opacity of voxel $i$ and $A_{i-1}$ is the accumulated opacity of all voxels in front of $i$ [S1, unofficial][S2].
    *Note: In slice-based rendering, slices are processed parallel to the viewing plane [S2].*

3.  **Compute Saliency Field ($L$):**
    For each appearance attribute $O \in \{O_b, O_s\}$, compute the Gaussian-weighted average $G(O, i, \sigma)$ for each voxel $i$ within its neighborhood $N(i, \sigma) = \{ j \mid \| j - i \| < \sigma \}$ [S1, unofficial][S2]:
    $$G(O, i, \sigma) = \frac{\sum_{j \in N(i,\sigma)} O(j) g(i, j, \sigma)}{\sum_{k \in N(i,\sigma)} g(i, j, \sigma)}$$
    where the Gaussian weight is:
    $$g(i, j, \sigma) = \frac{\exp[-\| j - i \|^2 / (2\sigma)^2]}{\sum_{k \in N(i,\sigma)} \exp[-\| k - i \|^2 / (2\sigma)^2]}$$
    The saliency field is the absolute difference of Gaussian-weighted averages at fine scale $\sigma$ and coarse scale $2\sigma$ [S1, unofficial][S2]:
    $$L(O, i, \sigma) = |w_1 G(O, i, \sigma) - w_2 G(O, i, 2\sigma)|$$
    where $w_1, w_2$ are weights for the fine and coarse scales [S1, unofficial][S2].

4.  **Compute Visibility-Weighted Saliency Field ($s_i$):**
    Combine visibility and saliency for each attribute [S1, unofficial]:
    $$s_i(O, i, \sigma) = v_i L(O, i, \sigma)$$
    The total visibility-weighted saliency field $S$ is the set $\{ s_i \mid i \in V \}$ [S1, unofficial].

5.  **Compute Feature Saliency ($W_F$):**
    For a feature $F \subseteq V$, calculate the normalized sum of saliency values [S1, unofficial]:
    $$W_F(O, i, \sigma) = \frac{\sum_{i \in F} s_i(O, i, \sigma)}{\sum_{i \in V} s_i(O, i, \sigma)}$$
    This value ranges in $[0, 1]$ [S1, unofficial].

6.  **Compute Weighted Feature Saliency ($W_F$):**
    Combine saliency from multiple attributes (typically brightness and saturation) using weights $u_1, u_2$ (where $u_1 + u_2 = 1$) [S1, unofficial][S2]:
    $$W_F = u_1 W_F(O_b, i, \sigma) + u_2 W_F(O_s, i, \sigma)$$
    Default weights are equal ($u_1 = u_2 = 0.5$) [S2].

7.  **Compute Objective Function (for Optimization):**
    To optimize transfer functions, minimize the squared difference between computed feature saliency $W_i$ and user-defined target importance $t_i$ [S2]:
    $$F = \sum_{i=1}^{n} (W_i - t_i)^2$$
    where $n$ is the number of features [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_visibility_field(volume, transfer_function, viewpoint):
    """
    Computes the visibility field v_i for all voxels i.
    volume: 3D array of voxel values
    transfer_function: function mapping voxel value -> (alpha, color)
    viewpoint: camera parameters
    """
    # Initialize accumulated opacity array
    A_prev = 0.0
    visibility_field = {}
    
    # Iterate through voxels in view order (front-to-back)
    for voxel in volume.get_voxels_in_view_order(viewpoint):
        alpha, color = transfer_function(voxel.value)
        # Visibility is the contribution of this voxel to the final image
        v_i = (1.0 - A_prev) * alpha
        visibility_field[voxel.id] = v_i
        # Update accumulated opacity
        A_prev = A_prev + (1.0 - A_prev) * alpha
        
    return visibility_field

def compute_gaussian_weighted_average(attribute_field, voxel, sigma):
    """
    Computes G(O, i, sigma) for a specific voxel.
    attribute_field: 3D array of appearance attributes (brightness or saturation)
    """
    neighborhood = voxel.get_neighbors(radius=sigma)
    numerator = 0.0
    denominator = 0.0
    
    for neighbor in neighborhood:
        dist_sq = voxel.distance_sq(neighbor)
        weight = math.exp(-dist_sq / (2.0 * sigma**2))
        numerator += attribute_field[neighbor.id] * weight
        denominator += weight
        
    return numerator / denominator if denominator > 0 else 0.0

def compute_saliency_field(attribute_field, volume, sigma, w1, w2):
    """
    Computes the saliency field L(O, i, sigma) for all voxels.
    """
    saliency_field = {}
    for voxel in volume:
        # Fine scale average
        G_fine = compute_gaussian_weighted_average(attribute_field, voxel, sigma)
        # Coarse scale average
        G_coarse = compute_gaussian_weighted_average(attribute_field, voxel, 2 * sigma)
        
        # Absolute difference
        L = abs(w1 * G_fine - w2 * G_coarse)
        saliency_field[voxel.id] = L
        
    return saliency_field

def compute_visibility_weighted_saliency(visibility_field, saliency_field, volume):
    """
    Computes s_i = v_i * L(O, i, sigma) for all voxels.
    """
    vws_field = {}
    for voxel in volume:
        v_i = visibility_field[voxel.id]
        L_i = saliency_field[voxel.id]
        vws_field[voxel.id] = v_i * L_i
    return vws_field

def compute_feature_saliency(vws_field, feature_voxel_ids, volume_voxel_ids):
    """
    Computes W_F = sum(s_i for i in F) / sum(s_i for i in V).
    """
    sum_feature = sum(vws_field[i] for i in feature_voxel_ids)
    sum_volume = sum(vws_field[i] for i in volume_voxel_ids)
    
    if sum_volume == 0:
        return 0.0
    return sum_feature / sum_volume

def compute_weighted_feature_saliency(feature_voxel_ids, volume_voxel_ids, 
                                      brightness_vws, saturation_vws, u1, u2):
    """
    Computes W_F = u1 * W_F(O_b) + u2 * W_F(O_s).
    """
    W_b = compute_feature_saliency(brightness_vws, feature_voxel_ids, volume_voxel_ids)
    W_s = compute_feature_saliency(saturation_vws, feature_voxel_ids, volume_voxel_ids)
    return u1 * W_b + u2 * W_s

def compute_objective_function(feature_saliencies, target_importances):
    """
    Computes F = sum((W_i - t_i)^2).
    feature_saliencies: list of W_i for each feature
    target_importances: list of t_i for each feature
    """
    total_error = 0.0
    for W_i, t_i in zip(feature_saliencies, target_importances):
        total_error += (W_i - t_i)**2
    return total_error
```

**Parameters:**
*   $\sigma$: Neighborhood radius for Gaussian weighting. Not given in the sources.
*   $w_1, w_2$: Weights for fine and coarse scale Gaussian averages. Not given in the sources.
*   $u_1, u_2$: Weights for brightness and saturation attributes. Default: $u_1 = u_2 = 0.5$ [S2].
*   Color Space: CIELCh or CIELab [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** The visibility field and saliency field must be recomputed at every optimization step if opacity values are updated, making the objective function evaluation expensive [S2].
*   **Opacity in Saliency:** Opacity is not used in the saliency field calculation because the visibility field already accounts for opacity contributions [S1, unofficial].
*   **Feature Definition:** Features must be defined as contiguous subsets of voxels, typically via transfer function ranges or segmentation [S2].

## Subtopics

### [Viewpoint Saliency](7185-viewpoint-saliency.md)
A measure of importance used to select the best viewpoints for volume visualization by maximizing the visible saliency of the volume data [7185-viewpoint-saliency.md]. It uses VWS to evaluate how well different viewpoints reveal the most salient features.

### 2D Feature Saliency (2DFS)
A method to estimate the visual saliency of 3D features by using an inverse distance weighting between pixels of the final rendered image and individual 2D feature images [S2].
*   **Definition:** Estimates 3D feature saliency from 2D renderings [S2].
*   **Steps:**
    1.  Render individual 2D feature images from the same view, isolating only individual features [S2].
    2.  Calculate the weighted total saliency of each 2D feature saliency map using inverse distance weighting to estimate 3D feature saliency [S2].
*   **Comparison:** Unlike VWS, which computes a 3D field, 2DFS operates on 2D image planes and uses inverse distance weighting [S2].

## Operating characteristics
*   **Time Complexity:** High. The visibility field requires a full volume rendering pass (front-to-back compositing) for every evaluation. The saliency field requires Gaussian smoothing over neighborhoods for all voxels. In optimization contexts, this must be repeated at every iteration, making it computationally expensive [S2].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, for storing the visibility field, saliency fields, and VWS field.
*   **Strengths:** Provides an objective, quantitative measure of feature conspicuity that combines both visibility (occlusion) and saliency (distinctiveness) [S1, unofficial]. It is view-independent for the saliency component but view-dependent for the visibility component, allowing for viewpoint optimization [S1, unofficial][S2].
*   **Limitations:** Computationally intensive for iterative optimization [S2]. Requires user-defined feature sets or transfer function ranges [S2].
*   **When to use:** When objective comparison of transfer functions or viewpoints is needed, or when automatically optimizing transfer functions to match user-defined conspicuity targets [S1, unofficial][S2].
*   **When not to use:** When real-time interactive performance is critical and pre-computation is not feasible [S2].

## Background
Visibility-Weighted Saliency was proposed by Shengzhou Luo and John Dingliana in 2015 [S1, unofficial]. It builds upon the concept of feature visibility [WZC*11] and saliency fields [KV06]. The method was extended in 2018 by Luo and Dingliana to include automatic transfer function optimization based on a combined model of visibility and saliency [S2]. It is used in volume visualization to guide users in choosing appropriate transfer functions and viewpoints [S1, unofficial].

## Sources
[S1] Visibility-Weighted Saliency for Volume Visualization, https://doi.org/10.2312/cgvc.20151243, fetched 2026-10-26 (unofficial)
[S2] Transfer function optimization based on a combined model of visibility and saliency, https://doi.org/10.1145/3154353.3154357, fetched 2026-11-01
