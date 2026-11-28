# View-Dependent Feature MLP

Also called: $\psi$

## Overview
The View-Dependent Feature MLP computes view-dependent features $F_i'$ by taking semantic features $F_i$ and relative view changes $\Delta v$ as inputs to model view-dependent effects such as lighting and view direction [S1]. It sets itself apart from its siblings by explicitly encoding the *change* in viewpoint between an input view and a target view, rather than just conditioning on the absolute camera pose, ensuring geometric consistency across views [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Semantic feature $F_i$ from an input view $i$, normalized view direction $v_i$ from the point to the camera center of view $i$, and normalized view direction $v_t$ from the point to the camera center of the target view $t$ [S1].
*   **Outputs:** View-dependent feature $F_i'$ [S1].

**Data Structures:**
*   $v_i, v_t \in \mathbb{R}^3$: Normalized vectors representing the direction from a 3D point to the camera center [S1].
*   $\Delta v \in \mathbb{R}^4$: Relative view direction change vector [S1].
*   $\delta(\cdot)$: A two-layer MLP mapping $\Delta v$ to a 32-dimensional vector [S1].
*   $\psi(\cdot)$: A two-layer MLP mapping the concatenated features to the final output [S1].

**Steps:**
1.  Calculate the relative view direction change $\Delta v$ using the normalized view directions $v_i$ and $v_t$:
    $$ \Delta v = \left[ \frac{v_i - v_t}{\|v_i - v_t\|}, v_i \cdot v_t \right], \quad v_i, v_t \in \mathbb{R}^3 $$
    [S1]
2.  Map $\Delta v$ to a 32-dimensional vector using the MLP $\delta$:
    $$ \delta_{vec} = \delta(\Delta v) $$
    [S1]
3.  Compute the view-dependent feature $F_i'$ using the MLP $\psi$ on the semantic feature $F_i$ and the mapped view change $\delta_{vec}$:
    $$ F_i' = \psi(F_i, \delta_{vec}) $$
    [S1]

**Parameters:**
*   MLP $\delta$: Two-layer MLP [S1].
*   MLP $\psi$: Two-layer MLP [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_view_dependent_feature(F_i, v_i, v_t, delta_mlp, psi_mlp):
    """
    Computes view-dependent features Fi' using the psi method.
    
    Args:
        F_i: Semantic feature vector from input view i.
        v_i: Normalized view direction from point to camera center of view i (shape: (3,)).
        v_t: Normalized view direction from point to camera center of target view t (shape: (3,)).
        delta_mlp: Two-layer MLP function mapping 4D input to 32D output [S1].
        psi_mlp: Two-layer MLP function mapping concatenated features to output [S1].
        
    Returns:
        F_i_prime: View-dependent feature vector.
    """
    # Step 1: Calculate relative view direction change delta_v
    # Formula: delta_v = [(v_i - v_t) / ||v_i - v_t||, v_i . v_t]
    diff = v_i - v_t
    norm_diff = np.linalg.norm(diff)
    
    # Guard against division by zero if views are identical [G]
    if norm_diff < 1e-8:
        delta_v = np.array([0.0, 0.0, 0.0, 1.0])
    else:
        normalized_diff = diff / norm_diff
        dot_product = np.dot(v_i, v_t)
        delta_v = np.concatenate([normalized_diff, [dot_product]])
        
    # Step 2: Map delta_v to 32-dimension vector using MLP delta
    delta_vec = delta_mlp(delta_v)
    
    # Step 3: Compute view-dependent feature Fi' using MLP psi
    # Input to psi is the concatenation of semantic feature Fi and delta_vec
    psi_input = np.concatenate([F_i, delta_vec])
    F_i_prime = psi_mlp(psi_input)
    
    return F_i_prime
```

**Edge Cases and Pitfalls:**
*   **Identical Views:** If $v_i = v_t$, the denominator $\|v_i - v_t\|$ becomes zero [G]. The implementation must handle this singularity, typically by setting the directional difference component to zero while retaining the dot product of 1.0 [G].
*   **Precision:** The normalization of the difference vector requires careful handling of floating-point precision to avoid NaNs in gradient computation during backpropagation [G].

## Subtopics

### Feature Bank / View-Adaptive Feature Encoding
Generates a view-adaptive feature vector for each anchor by blending a base feature with downsampled versions, weighted by camera-relative position and direction to capture different scene granularities [S2]. It differs from the standard View-Dependent Feature MLP by using a "feature bank" of multi-resolution features (base, downsampled 1, downsampled 2) rather than a single semantic feature, allowing the model to capture different scene granularities based on view distance and direction [S2].

*   **Definition:** Updates the anchor feature through a view-dependent encoding by predicting weights for a feature bank [S2].
*   **Steps:**
    1.  Compute the relative distance $\delta_{vc}$ and viewing direction $\vec{d}_{vc}$ between the camera and the anchor point [S2].
    2.  Predict a weight vector $w \in \mathbb{R}^3$ using a tiny MLP $F_w$:
        $$ (w, w_1, w_2) = \text{Softmax}(F_w(\delta_{vc}, \vec{d}_{vc})) $$
        [S2]
    3.  Encode the view direction information to the anchor feature $f_v$ by compositing the feature bank:
        $$ \hat{f}_v = w \cdot f_v + w_1 \cdot f_{v\downarrow 1} + w_2 \cdot f_{v\downarrow 2} $$
        [S2]
*   **Parameters:** $F_w$ is a tiny MLP serving as the view encoding function [S2].
*   **Layout:** The feature bank is constructed by slicing and repeating the anchor feature $f$ to produce the set $\{f_v, f_{v\downarrow 1}, f_{v\downarrow 2}\}$ [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Ensures geometric consistency across views by modeling relative view changes rather than absolute poses, which helps with lighting and specularities [S1].
*   **Limitations:** Requires access to both input and target view camera poses for every point, increasing data dependency during rendering [S1].
*   **When to use:** When rendering novel views from a set of input views where lighting and view-dependent effects are significant and geometric consistency is critical [S1].

## Background
The method was introduced in a neural point cloud rendering pipeline for novel view synthesis [S1]. It addresses inconsistency between multiple views caused by variations in lighting and view direction [S1]. The pipeline transforms points to target view coordinates based on camera poses and renders them using a differentiable splatting renderer [S1].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
[S2] Scaffold-GS: Structured 3D Gaussians for View-Adaptive Rendering, https://doi.org/10.1109/cvpr52733.2024.01952, fetched 2026-10-26
