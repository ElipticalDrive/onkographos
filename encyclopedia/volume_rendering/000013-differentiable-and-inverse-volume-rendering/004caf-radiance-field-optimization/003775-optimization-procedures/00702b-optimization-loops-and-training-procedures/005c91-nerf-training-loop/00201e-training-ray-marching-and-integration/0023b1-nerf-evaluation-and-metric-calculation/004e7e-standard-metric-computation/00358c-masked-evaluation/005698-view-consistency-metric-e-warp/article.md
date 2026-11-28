# View-Consistency Metric (E_warp)
Also called: Warped View Consistency Metric

## Overview
The View-Consistency Metric ($E_{\text{warp}}$) evaluates geometric and photometric consistency in radiance fields by measuring the mean squared error between a rendered view and a neighboring view warped into the target coordinate system [S1]. It differs from standard metrics by explicitly masking occluded regions using optical flow, ensuring errors are only computed where scene content is visible in both views [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Rendered images $V_i$ and $V_{i+\delta}$ from a pre-trained NeRF at camera poses separated by offset $\delta$, an optical flow estimator (RAFT), and an occlusion detection method (Ruder et al.) [S1].
- **Outputs:** A scalar consistency error $E_{\text{warp}}$ [S1].

**Data Structures:**
- $V_{i+\delta}$: Rendered image at pose $i+\delta$ (target view) [S1].
- $\bar{V}_{i+\delta}$: Warped image at pose $i+\delta$ (source view $V_i$ warped to $i+\delta$) [S1].
- $O$: Binary occlusion mask of the same spatial dimensions as the images, where $O(x,y)=1$ if the pixel is visible in both views and $0$ otherwise [S1].
- $|O|$: The number of pixels where $O(x,y)=1$ [S1].

**Steps:**
1. Render images $V_i$ and $V_{i+\delta}$ from the NeRF at camera poses separated by offset $\delta$ [S1].
2. Estimate the optical flow field from $V_i$ to $V_{i+\delta}$ using a pre-trained optical flow network (RAFT) [S1].
3. Derive the occlusion mask $O$ using the occlusion detection method from Ruder et al. [S1].
4. Warp $V_i$ into the coordinate system of $V_{i+\delta}$ using the estimated flow to produce $\bar{V}_{i+\delta}$ [S1].
5. Compute the Mean Squared Error (MSE) between $\bar{V}_{i+\delta}$ and $V_{i+\delta}$ restricted to pixels where $O=1$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_e_warp(V_i, V_ip, flow_net, occlusion_net, delta):
    # 1. Render V_i and V_ip at poses separated by delta (handled externally)
    # 2. Estimate optical flow from V_i to V_ip
    flow = flow_net(V_i, V_ip)
    # 3. Derive occlusion mask O
    O = occlusion_net(V_i, V_ip, flow)
    # 4. Warp V_i to V_ip coordinate system using bilinear sampling
    V_bar_ip = sample_with_flow(V_i, flow)
    # 5. Compute MSE over valid pixels
    mask = O > 0
    if not mask.any():
        return 0.0
    error = mean((V_bar_ip[mask] - V_ip[mask]) ** 2)
    return error

def sample_with_flow(image, flow):
    # Standard bilinear warping: maps each pixel (x,y) in target to source coordinates
    # using the flow vector, then samples RGB values
    return bilinear_sample(image, flow)
```

**Formula:**
$$
E_{\text{warp}}(V_{i+\delta}, \bar{V}_{i+\delta}) = \frac{1}{|O|} \left\| \bar{V}_{i+\delta} - V_{i+\delta} \right\|_2^2
$$
Where:
- $V_{i+\delta}$ is the rendered view at pose $i+\delta$ [S1].
- $\bar{V}_{i+\delta}$ is the warped view from pose $i$ to $i+\delta$ [S1].
- $O$ is the occlusion mask [S1].
- $\| \cdot \|_2^2$ denotes the squared $L_2$ norm (sum of squared differences) [S1].

**Parameters:**
- $\delta$: The offset between viewpoints [S1].
  - Short-range consistency: $\delta = 1$ [S1].
  - Long-range consistency: $\delta = 7$ [S1].

**Edge Cases and Pitfalls:**
- **Occlusion Handling:** Pixels occluded in one view but visible in the other must be masked out ($O=0$) to prevent the metric from penalizing valid geometric changes as errors [S1].
- **Pre-trained Requirement:** The optical flow must be estimated on a video sequence rendered from a *pre-trained* NeRF to ensure the flow corresponds to the scene geometry rather than training artifacts [S1].
- **Flow Estimation Limits:** Incorrect flow in textureless regions or large displacements can warp pixels incorrectly, producing false consistency errors [G].

## Subtopics

### [Warped LPIPS Metric](16418-warped-lpips-metric.md)
The Warped LPIPS Metric is a no-reference evaluation method used to assess geometric and photometric consistency across viewpoints in novel view synthesis, particularly for stylized or super-resolved NeRFs. It quantifies consistency by computing the Learned Perceptual Image Patch Similarity (LPIPS) between a target rendered view and a source view warped into the target's coordinate system using estimated depth. Unlike standard metrics that require ground truth, it measures how well the underlying 3D structure supports consistent appearance from nearby angles. It distinguishes itself from the base View-Consistency Metric ($E_{\text{warp}}$) by using LPIPS instead of MSE, making it more sensitive to perceptual artifacts and texture consistency.

## Operating characteristics
- **Time Complexity:** Dominated by NeRF rendering and the inference of the optical flow network and occlusion detector [S1]. The warping and error computation scale linearly with the number of pixels [G].
- **Memory Complexity:** Requires storing the rendered images, the optical flow field, and the occlusion mask [G].
- **Strengths:** Provides a geometrically aware consistency check that does not require ground truth multi-view data, making it suitable for self-supervised training or evaluation of unstructured captures [S1].
- **Limitations:** Relies on the accuracy of external models (RAFT, occlusion detector) and degrades when flow estimation is ambiguous [S1].
- **When to use:** Evaluating geometric coherence in NeRFs, particularly in video stylization or recoloring tasks where temporal or spatial consistency is critical [S1].
- **When not to use:** When ground truth multi-view data is available and direct photometric error is preferred, or when the scene contains significant non-rigid motion that violates the optical flow assumption [G].

## Background
The metric is based on the view-consistency evaluation introduced in SNeRF [31], which generalized the approach from Lai et al. [22] to the NeRF setting [S1]. It was adopted in LAENeRF for evaluating 3D-aware stylization and recoloring consistency, using short-range ($\delta=1$) and long-range ($\delta=7$) offsets [S1].

## Sources
[S1] LAENeRF: Local Appearance Editing for Neural Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.00475, fetched 2026-10-26
