# Self-Supervised Depth Losses
## Overview
Self-supervised depth losses constrain the estimated depth and confidence intervals using monocular depth estimates and image gradients to address the difficulty of obtaining ground truth depth [S1].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered depth map $\hat{D}$, monocular depth estimate $\tilde{D}_{mono}$, aligned RGB image $I$, and image dimensions $H, W$ [S1].
*   **Outputs:** Scalar loss value $\mathcal{L}_{smooth}$ [S1].

**Data Structures:**
*   $\hat{D}$: Array of shape $(H, W)$ containing per-pixel depth values [S1].
*   $\tilde{D}_{mono}$: Array of shape $(H, W)$ containing per-pixel monocular depth/disparity values [S1].
*   $I$: Array of shape $(H, W, 3)$ containing RGB pixel values [S1].

**Steps:**
1. Compute spatial gradients of $\hat{D}$, $I$, and $\tilde{D}_{mono}$ in horizontal ($x$) and vertical ($y$) directions [S1].
2. Compute exponential weights from the gradient magnitudes of $I$ and $\tilde{D}_{mono}$ [S1].
3. Weight the gradient magnitudes of $\hat{D}$ by the computed weights and sum over all pixels [S1].

**Formula:**
The smoothness loss $\mathcal{L}_{smooth}$ is defined as [S1]:

$$
\mathcal{L}_{smooth} = \frac{1}{HW} \sum_{r} \left( \|\partial_x \hat{D}(r)\| (e^{-|\partial_x I(r)|} + e^{-|\partial_x \tilde{D}_{mono}(r)|}) + \|\partial_y \hat{D}(r)\| (e^{-|\partial_y I(r)|} + e^{-|\partial_y \tilde{D}_{mono}(r)|}) \right)
$$

Where:
*   $r$ iterates over all pixels $(x, y)$ [S1].
*   $\|\cdot\|$ denotes the magnitude of the gradient [S1].
*   $e^{-|\cdot|}$ acts as a weight that suppresses the loss at edges (high gradient magnitude) in the RGB image or monocular depth map [S1].
*   $H, W$ are the height and width of the image [S1].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Gradients near zero are handled naturally by the exponential decay [G].
*   **Boundaries:** Gradient computation at image borders requires padding or boundary handling [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_smoothness_loss(rendered_depth, mono_depth, rgb_image):
    H = len(rendered_depth)
    W = len(rendered_depth[0])
    total_loss = 0.0
    for y in range(H):
        for x in range(W):
            dx_d = rendered_depth[y][x + 1] - rendered_depth[y][x] if x < W - 1 else 0.0
            dy_d = rendered_depth[y + 1][x] - rendered_depth[y][x] if y < H - 1 else 0.0
            dx_i = math.sqrt(sum((rgb_image[y][x + 1][c] - rgb_image[y][x][c]) ** 2 for c in range(3))) if x < W - 1 else 0.0
            dy_i = math.sqrt(sum((rgb_image[y + 1][x][c] - rgb_image[y][x][c]) ** 2 for c in range(3))) if y < H - 1 else 0.0
            dx_m = mono_depth[y][x + 1] - mono_depth[y][x] if x < W - 1 else 0.0
            dy_m = mono_depth[y + 1][x] - mono_depth[y][x] if y < H - 1 else 0.0
            total_loss += abs(dx_d) * (math.exp(-dx_i) + math.exp(-dx_m)) + abs(dy_d) * (math.exp(-dy_i) + math.exp(-dy_m))
    return total_loss / (H * W)
```
## Subtopics

### Gradient-Aware Depth Loss
A loss function that combines RGB image gradient weighting with L1 loss between rendered depth $\hat{D}$ and ground truth depth $D$ to regularize depth estimation [S2]. It is set apart by its use of an L1 norm and explicit ground truth supervision rather than smoothness priors on monocular estimates [S2]. Formula: $\mathcal{L}_{\hat{D}} = g_{\text{rgb}} \frac{1}{|\hat{D}|} \sum \log(1+\|\hat{D}-D\|_1)$, where $g_{\text{rgb}} = \exp(-\nabla I)$ and $|\hat{D}|$ is the total number of pixels [S2].

## Operating characteristics
**Time and Memory Complexity:**
Linear in the number of pixels $HW$ for gradient computation and summation [S1, S2].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Scales linearly with image resolution [S1, S2].

**Strengths and Limitations:**
*   **Strengths:** Self-supervised, does not require ground truth depth [S1].
*   **Limitations:** Relies on the quality of monocular depth estimates [S1].

**When to Use:**
Use when ground truth depth is unavailable and monocular depth estimates are reliable [S1].

**When Not to Use:**
Not given in the sources.
## Background
Originated in NeRF-SDP (GRF, MVSNeRF, ENeRF, IBRNet) presented at MMAsia ’23 [S1]. Developed to efficiently generalize neural radiance fields with scene depth perception without ground truth [S1]. Also used in DN-Splatter for Gaussian Splatting and Meshing [S2].
## Sources
[S1] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S2] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
