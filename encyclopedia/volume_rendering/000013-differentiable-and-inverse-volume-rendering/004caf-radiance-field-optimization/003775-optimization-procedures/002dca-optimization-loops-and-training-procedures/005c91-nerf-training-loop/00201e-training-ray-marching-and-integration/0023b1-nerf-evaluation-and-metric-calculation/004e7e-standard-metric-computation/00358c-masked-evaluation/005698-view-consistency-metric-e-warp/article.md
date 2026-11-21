# View-Consistency Metric (E_warp)
Also called: Warped View Consistency Metric

## Overview
The View-Consistency Metric (E_warp) evaluates the geometric and photometric consistency of a radiance field by measuring the reconstruction error between a rendered view and a neighboring view warped into the target viewpoint. It serves as a self-supervised regularizer or evaluation metric that penalizes inconsistencies arising from incorrect depth or color predictions, leveraging optical flow and occlusion masks to isolate valid correspondences. It differs from standard PSNR or LPIPS by explicitly accounting for viewpoint changes and occlusions, ensuring that errors are only measured where scene content is visible in both views.

## How it works
**Inputs and Outputs:**
- **Inputs:** A sequence of rendered images $V$ from a pre-trained NeRF at camera poses $i$ and $i+\delta$; an occlusion detection method; an optical flow estimator.
- **Outputs:** A scalar consistency error $E_{\text{warp}}$.

**Data Structures:**
- $V_{i+\delta}$: Rendered image at pose $i+\delta$ (target view).
- $\bar{V}_{i+\delta}$: Warped image at pose $i+\delta$ (source view $V_i$ warped to $i+\delta$).
- $O$: Binary occlusion mask of the same spatial dimensions as the images, where $O(x,y)=1$ if the pixel is visible in both views, $0$ otherwise.
- $|O|$: The number of pixels where $O(x,y)=1$.

**Steps:**
1. **Render Sequence:** Render images $V_i$ and $V_{i+\delta}$ from the NeRF at camera poses separated by a temporal or spatial offset $\delta$.
2. **Estimate Optical Flow:** Compute the optical flow field from $V_i$ to $V_{i+\delta}$ using a pre-trained optical flow network (e.g., RAFT) [S1].
3. **Derive Occlusion Mask:** Generate the occlusion mask $O$ using an occlusion detection algorithm (e.g., Ruder et al. [35]) based on the estimated flow and image consistency [S1].
4. **Warp Source View:** Use the optical flow to warp $V_i$ into the coordinate system of $V_{i+\delta}$, producing $\bar{V}_{i+\delta}$.
5. **Compute Error:** Calculate the Mean Squared Error (MSE) between the warped view and the target view, restricted to the valid pixels in $O$.

**Formula:**
$$
E_{\text{warp}}(V_{i+\delta}, \bar{V}_{i+\delta}) = \frac{1}{|O|} \left\| \bar{V}_{i+\delta} - V_{i+\delta} \right\|_2^2
$$
Where:
- $V_{i+\delta}$ is the rendered view at pose $i+\delta$.
- $\bar{V}_{i+\delta}$ is the warped view from pose $i$ to $i+\delta$.
- $O$ is the occlusion mask.
- $\| \cdot \|_2^2$ denotes the squared $L_2$ norm (sum of squared differences).

**Parameters:**
- $\delta$: The offset between viewpoints.
  - Short-range consistency: $\delta = 1$ [S1].
  - Long-range consistency: $\delta = 7$ [S1].

**Edge Cases and Pitfalls:**
- **Occlusion Handling:** Pixels occluded in one view but visible in the other must be masked out ($O=0$) to prevent the metric from penalizing valid geometric changes as errors [S1].
- **Flow Accuracy:** The quality of the warp depends heavily on the accuracy of the optical flow estimator. Large displacements or textureless regions may lead to incorrect flow, causing false errors [S1].
- **Pre-trained Requirement:** The optical flow must be estimated on a video sequence rendered from a *pre-trained* NeRF to ensure the flow corresponds to the scene geometry rather than training artifacts [S1].

## Subtopics

### Warped LPIPS Metric
A variant of the view-consistency metric that uses Learned Perceptual Image Patch Similarity (LPIPS) instead of MSE to assess consistency. It is defined as $E_{\text{warp}}(I_v, I_{v'}) = \text{LPIPS}(M_{v,v'} \cdot I_v, W(I_{v'}))$, where $I_v$ and $I_{v'}$ are renders from nearby viewpoints $v$ and $v'$, $W(\cdot)$ is the warping function, and $M_{v,v'}$ is the warping mask [S2]. The warping process involves backprojecting pixels from $I_{v'}$ to a point cloud in world space using predicted depths, then rasterizing this point cloud into viewpoint $v$ to produce $W(I_{v'})$ [S2]. The LPIPS score is computed between the target image $I_v$ and the warped image $W(I_{v'})$ over the masked regions [S2]. In experiments, $v'$ is selected as the 3rd nearest test pose from $v$ [S2]. This metric is used as a no-reference evaluator for super-resolution quality [S2].

## Operating characteristics
- **Time Complexity:** Dominated by the rendering of the NeRF views and the inference of the optical flow network (e.g., RAFT) and occlusion detector. The warping and MSE/LPIPS computation are linear in the number of pixels.
- **Memory Complexity:** Requires storing the rendered images and the optical flow field. The occlusion mask is typically binary and compact.
- **Strengths:** Provides a geometrically aware consistency check that is robust to lighting changes if the flow is accurate. It does not require ground truth multi-view data, making it suitable for self-supervised training or evaluation of unstructured captures.
- **Limitations:** Relies on the accuracy of external models (RAFT, occlusion detector). Fails in textureless regions or areas with large occlusions where flow estimation is ambiguous. Long-range consistency ($\delta=7$) may suffer from accumulated warping errors [S1].
- **When to use:** Evaluating the geometric coherence of NeRFs, particularly in video stylization or super-resolution tasks where temporal or spatial consistency is critical [S1, S2].
- **When not to use:** When ground truth multi-view data is available and direct photometric error is preferred, or when the scene contains significant non-rigid motion that violates the optical flow assumption [S1].

## Background
The metric is based on the view-consistency evaluation introduced in SNeRF [31], which generalized the approach from Lai et al. [22] to the NeRF setting [S1]. It was adopted in LAENeRF for evaluating 3D-aware stylization and recoloring consistency, using short-range ($\delta=1$) and long-range ($\delta=7$) offsets [S1]. It was also employed in DiSR-NeRF as a no-reference evaluator for super-resolution, utilizing a point-cloud-based warping strategy and LPIPS [S2].

## Sources
[S1] LAENeRF: Local Appearance Editing for Neural Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.00475, fetched 2026-10-26
[S2] DiSR-NeRF: Diffusion-Guided View-Consistent Super-Resolution NeRF, https://doi.org/10.1109/cvpr52733.2024.01943, fetched 2026-10-26
