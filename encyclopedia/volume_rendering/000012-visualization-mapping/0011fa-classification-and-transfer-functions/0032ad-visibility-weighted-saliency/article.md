# Visibility-Weighted Saliency

Also called: Saliency Field Computation, VWS metric, Visibility-weighted saliency field, conspicuity metric

## Overview
Visibility-Weighted Saliency (VWS) is a metric that quantifies the perceptual conspicuity of features in volume visualization by combining viewpoint-dependent occlusion (visibility) with local distinctiveness (saliency) [S1, unofficial][S2]. It solves the problem of objectively measuring and comparing feature visibility across different transfer functions and viewpoints, addressing the limitation of methods that rely solely on geometric complexity or camera distance [S1, unofficial]. Unlike [Feature Visibility](6229-binary-and-soft-classification.md) which only measures occlusion/contribution, VWS also measures conspicuity (standing out from surroundings) via a center-surround operator on appearance attributes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$, a transfer function defining opacity $\alpha$ and color for each voxel, a viewpoint (camera position and orientation), and a set of user-defined features $F \subseteq V$ (defined by transfer function ranges or segmentation).
*   **Output:** A visibility-weighted saliency field $S$ (scalar field per voxel) and/or a feature saliency value $W_F$ (scalar in $[0, 1]$).

**Data Structures:**
*   **Volume Grid:** 3D array of voxels. Each voxel $i$ stores position $\mathbf{x}_i$, appearance attributes $O_i$ (brightness $O_b$, saturation $O_s$), and opacity $\alpha_i$.
*   **Visibility Field:** 3D array $V = \{v_i\}$ storing the visibility contribution of each voxel.
*   **Saliency Field:** 3D array $L = \{L(O, i, \sigma)\}$ storing the local distinctiveness of each voxel.
*   **Feature Mask:** Binary mask or list of voxel indices identifying feature $F$.

**Parameters:**
*   $\sigma$: Neighborhood radius for Gaussian weighting. Typical value: Not given in the sources.
*   $w_1, w_2$: Weights for fine and coarse scale Gaussian averages in saliency calculation. Typical value: Not given in the sources.
*   $u_1, u_2$: Weights for combining brightness and saturation saliency. Default: $u_1 = u_2 = 0.5$ [S2].
*   $t_i$: User-defined target importance for feature $i$, normalized such that $\sum t_i = 1$ [S2].

**Steps:**

1.  **Compute Visibility Field:**
    Perform front-to-back compositing along rays from the viewpoint. For each voxel $i$ along a ray, calculate accumulated opacity $A_i$ and visibility $v_i$:
    $$ A_i = (1 - A_{i-1})\alpha_i + A_{i-1} $$
    $$ v_i = A_i - A_{i-1} = (1 - A_{i-1})\alpha_i $$
    where $A_{i-1}$ is the accumulated opacity of voxels in front of $i$ [S1, unofficial][S2].
    *Implementation Note:* This can be done via slice-based rendering with slices parallel to the viewing plane, subtracting accumulated opacity of previous slice from current [S2].

2.  **Compute Saliency Field:**
    For each voxel $i$, compute the Gaussian-weighted average of appearance attribute $O$ (brightness or saturation) over its neighborhood $N(i, \sigma) = \{ j \mid \| j - i \| < \sigma \}$:
    $$ G(O, i, \sigma) = \frac{\sum_{j \in N(i,\sigma)} O(j) g(i, j, \sigma)}{\sum_{k \in N(i,\sigma)} g(i, j, \sigma)} $$
    where the Gaussian weight is:
    $$ g(i, j, \sigma) = \frac{\exp[-\| j - i \|^2 / (2\sigma)^2]}{\sum_{k \in N(i,\sigma)} \exp[-\| k - i \|^2 / (2\sigma)^2]} $$
    The saliency field $L(O, i, \sigma)$ is the absolute difference of Gaussian-weighted averages at fine scale $\sigma$ and coarse scale $2\sigma$:
    $$ L(O, i, \sigma) = |w_1 G(O, i, \sigma) - w_2 G(O, i, 2\sigma)| $$
    Use perceptually uniform color spaces like CIELCh or CIELab for $O$ [S1, unofficial].

3.  **Compute Visibility-Weighted Saliency Field:**
    Combine visibility and saliency for each voxel $i$:
    $$ s_i(O, i, \sigma) = v_i L(O, i, \sigma) $$
    The field is $S = \{ s_i(O, i, \sigma) \mid i \in V \}$ [S1, unofficial].

4.  **Compute Feature Saliency (Optional):**
    For a feature $F \subseteq V$, compute the normalized sum of saliency values:
    $$ W_F(O, i, \sigma) = \frac{\sum_{i \in F} s_i(O, i, \sigma)}{\sum_{i \in V} s_i(O, i, \sigma)} $$
    $W_F$ ranges in $[0, 1]$ [S1, unofficial].

5.  **Combine Attributes (Optional):**
    Combine saliency from brightness ($O_b$) and saturation ($O_s$):
    $$ W_F = u_1 W_F(O_b, i, \sigma) + u_2 W_F(O_s, i, \sigma) $$
    where $u_1 + u_2 = 1$ [S1, unofficial][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_visibility_field(volume, transfer_function, camera):
    """
    Computes the visibility field V for all voxels in volume.
    volume: 3D array of voxel data (position, opacity, color)
    transfer_function: function mapping voxel value to (opacity, color)
    camera: object with method raycast(origin, direction) returning list of (voxel_index, distance)
    """
    V = {} # Dictionary mapping voxel_index to visibility value
    for ray in camera.generate_rays():
        samples = camera.raycast(ray.origin, ray.direction)
        accumulated_opacity = 0.0
        for voxel_index, distance in samples:
            # Apply transfer function to get opacity
            voxel = volume[voxel_index]
            opacity = transfer_function(voxel.value)
            
            # Compute visibility contribution
            visibility = (1.0 - accumulated_opacity) * opacity
            
            V[voxel_index] = V.get(voxel_index, 0.0) + visibility
            accumulated_opacity += visibility
            
    return V

def compute_saliency_field(volume, attribute='brightness', sigma=1.0, w1=1.0, w2=1.0):
    """
    Computes the saliency field L for all voxels.
    volume: 3D array of voxel data
    attribute: 'brightness' or 'saturation'
    sigma: neighborhood radius
    w1, w2: weights for fine and coarse scales
    """
    L = {}
    dimensions = volume.dimensions
    
    for voxel_index in volume.all_voxels():
        voxel = volume[voxel_index]
        O_i = voxel.get_attribute(attribute)
        
        # Compute Gaussian-weighted average at scale sigma
        sum_weighted_O = 0.0
        sum_weights = 0.0
        for neighbor in volume.get_neighbors(voxel_index, sigma):
            dist_sq = voxel.distance_sq(neighbor)
            weight = math.exp(-dist_sq / (2.0 * sigma**2))
            sum_weighted_O += neighbor.get_attribute(attribute) * weight
            sum_weights += weight
            
        G_fine = sum_weighted_O / sum_weights if sum_weights > 0 else 0.0
        
        # Compute Gaussian-weighted average at scale 2*sigma
        sum_weighted_O_coarse = 0.0
        sum_weights_coarse = 0.0
        for neighbor in volume.get_neighbors(voxel_index, 2.0 * sigma):
            dist_sq = voxel.distance_sq(neighbor)
            weight = math.exp(-dist_sq / (2.0 * (2.0 * sigma)**2))
            sum_weighted_O_coarse += neighbor.get_attribute(attribute) * weight
            sum_weights_coarse += weight
            
        G_coarse = sum_weighted_O_coarse / sum_weights_coarse if sum_weights_coarse > 0 else 0.0
        
        # Saliency is absolute difference
        L[voxel_index] = abs(w1 * G_fine - w2 * G_coarse)
        
    return L

def compute_visibility_weighted_saliency(visibility_field, saliency_field):
    """
    Computes visibility-weighted saliency field S.
    visibility_field: dict mapping voxel_index to visibility v_i
    saliency_field: dict mapping voxel_index to saliency L_i
    """
    S = {}
    for voxel_index in visibility_field:
        if voxel_index in saliency_field:
            S[voxel_index] = visibility_field[voxel_index] * saliency_field[voxel_index]
    return S

def compute_feature_saliency(S, feature_mask):
    """
    Computes normalized saliency for a feature F.
    S: visibility-weighted saliency field
    feature_mask: set of voxel indices belonging to feature F
    """
    sum_feature = sum(S.get(i, 0.0) for i in feature_mask)
    sum_volume = sum(S.values())
    
    if sum_volume == 0:
        return 0.0
    return sum_feature / sum_volume
```

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulated opacity $A_i$ can suffer from precision issues if many transparent voxels are present. Use standard alpha blending order [S1, unofficial].
*   **Boundaries:** Neighborhood $N(i, \sigma)$ must handle boundary voxels by only considering existing neighbors [S1, unofficial].
*   **Performance:** Recomputing visibility field and saliency field at each optimization step is computationally expensive [S2].

## Subtopics

### [Viewpoint Saliency](7185-viewpoint-saliency.md)
Viewpoint Saliency is a metric used to select optimal viewpoints for volume visualization by maximizing the visible perceptual importance of the data. It builds on [Visibility-Weighted Saliency](6824-visibility-weighted-saliency.md) by aggregating voxel-level saliency values weighted by their geometric visibility from a candidate viewpoint. Unlike methods that rely solely on geometric complexity or camera distance, this approach prioritizes views where the most semantically or structurally significant features are unoccluded.

### 2D Feature Saliency (2DFS)
2D Feature Saliency (2DFS) is a method to estimate the visual saliency of 3D features by using an inverse distance weighting between pixels of the final rendered image and individual 2D feature images [S2]. It renders individual 2D feature images from the same view, isolating only individual features, and calculates the weighted total saliency of each 2D feature saliency map using inverse distance weighting to estimate 3D feature saliency [S2].

## Operating characteristics
*   **Time Complexity:** Computing the visibility field requires a full volume render (front-to-back compositing), which is $O(N)$ where $N$ is the number of voxels along rays. Computing the saliency field involves Gaussian smoothing over neighborhoods, which is $O(N \cdot |N(i, \sigma)|)$. The total complexity is dominated by the rendering step.
*   **Memory Complexity:** Requires storage for the volume, visibility field, and saliency fields, all $O(N)$.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The evaluation of the objective function is computationally expensive because the visibility field and visibility-weighted saliency need to be recomputed at each optimization step after feature opacity values are updated [S2].
*   **Strengths:** Provides an objective, quantitative measure of feature conspicuity that accounts for both occlusion and local distinctiveness [S1, unofficial]. Can be used to guide users in choosing appropriate transfer functions and viewpoints [S1, unofficial].
*   **Limitations:** Computationally expensive for iterative optimization [S2]. Depends on the quality of the transfer function and viewpoint [S1, unofficial].
*   **When to use:** When objective comparison of transfer functions or viewpoints is needed, or when optimizing transfer functions to match user-defined conspicuity targets [S1, unofficial][S2].
*   **When not to use:** When real-time performance is critical and pre-computation is not feasible [S2].

## Background
Visibility-Weighted Saliency was proposed by Shengzhou Luo and John Dingliana in 2015 [S1, unofficial]. It builds on earlier work on feature visibility [WZC* 11] and saliency fields [KV06]. The technique is used to objectively compare transfer functions and viewpoints for volume rendering, specifically to identify which features are most salient/visible [S1, unofficial]. It has been extended to automatic transfer function optimization [S2].

## Sources
[S1] Visibility-Weighted Saliency for Volume Visualization, https://doi.org/10.2312/cgvc.20151243, fetched 2026-10-26 (unofficial)
[S2] Transfer function optimization based on a combined model of visibility and saliency, https://doi.org/10.1145/3154353.3154357, fetched 2026-11-01
