# Shape-Shifting Splats

Also called: Context Translation for Gaussian Splatting

## Overview
Shape-Shifting Splats is a dynamic reconstruction method that predicts all 3D Gaussian parameters via shallow, fully-connected MLPs conditioned on a control parameter, allowing all attributes to be changed at render time [S1]. It builds on [Deformable 3D Gaussian Splatting](10060-deformable-3d-gaussian-splatting.md) and [Gaussian Splatting Initialization and Parameterization](52-gaussian-splatting-initialization-and-parameterization.md) by replacing explicit storage of scale, rotation, offset, and opacity with learned functions, enabling context translation (e.g., RGB to thermal) and stylization without retraining the scene geometry [S1].

## How it works
**Inputs:** Posed images (RGB, thermal, or stylized), sparse input point cloud from Structure-from-Motion (SfM) pipeline (e.g., COLMAP) [S1].
**Outputs:** Rendered images via tile-based rasterization [S1].

**Data Structures:**
The scene is represented by a set of anchors. For each anchor $v$:
*   $a_v \in \mathbb{R}^3$: Anchor point position.
*   $f_a \in \mathbb{R}^{32}$: Anchor feature vector.
*   MLP weights: Separate shallow, fully-connected MLPs for each Gaussian parameter (scale, rotation, offset, color, opacity) [S1].
*   Control encoding parameters: $\sigma$ (standard deviation), $s$ (sampling interval), $d$ (embedding dimension) [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize the scene with a sparse input point cloud derived via an SfM pipeline such as COLMAP [S1].
2.  **Voxelization:** Separate the scene into voxels. Each voxel has one anchor point $a_v$, one feature vector $f_a$, a voxel scaling parameter $l_v \in \mathbb{R}^3$, and $k$ offsets $o_{kv} \in \mathbb{R}^3$ [S1]. *Note: The feature-bank implementation of Scaffold-GS is omitted to avoid training instability [S1].*
3.  **Control Encoding:** Define a control parameter $x \in \mathbb{R}$ to encode the current modality or style (e.g., $x=0$ for RGB, $x=1$ for thermal) [S1]. Map $x$ to a higher-dimensional embedding space via $d$-dimensional encoding $\Psi: \mathbb{R} \to \mathbb{R}^d$ using a 1D Gaussian embedder [S1]:
    $$ \psi(t, x) = \exp\left(-\frac{\|t - x\|^2}{2\sigma^2}\right) $$
    $$ \Psi(x) = [\psi(0, x), \psi(s, x), \dots, \psi((d - 1)s, x)]^\top $$
    where $t$ is the sampling point, $\sigma$ is the standard deviation, and $s$ is the sampling interval [S1].
4.  **Viewing Direction:** For each anchor $a_v$ and camera position $x_c$, compute the viewing direction vector $\delta_{x_c, a_v}$:
    $$ \delta_{x_c, a_v} = \frac{a_v - x_c}{\|a_v - x_c\|_2} $$
    [S1].
5.  **Parameter Prediction:** Predict each Gaussian parameter using respective shallow, fully-connected MLPs. The inputs to the MLPs are the anchor features $f_a$, the control encoding $\Psi(x)$, and (for color only) the viewing direction $\delta_{x_c, a_v}$ [S1].
    *   **Color:** $c = F_{\text{color}}(f_a \oplus \delta_{x_c, a_v} \oplus \Psi(x))$, where $F_{\text{color}}$ is the color MLP and $\oplus$ denotes concatenation [S1].
    *   **Position:** $\mu = a_v + o_{kv} l_v$, where $\mu$ is the Gaussian position, $o_{kv}$ is the offset, and $l_v$ is the voxel scaling parameter [S1]. *Note: Two additional shallow, fully-connected MLPs are introduced to predict voxel scales $l_v$ and offsets $o_{kv}$ [S1].*
    *   **Covariance:** The covariance matrix $\Sigma$ is parameterized by a scaling matrix $S$ and a rotation matrix $R$:
        $$ \Sigma = R S S^T R^T $$
        [S1].
    *   **Opacity:** Predicted by the opacity MLP [S1].
    *   Each anchor feature $f_a$ predicts $k$ Gaussians, with $k=10$ in the original Scaffold-GS implementation [S1].
6.  **Rendering:** Perform standard 3DGS workflow. Render images via tile-based rasterization. The final color $c_p$ at pixel $p$ is calculated by alpha-blending $N$ sorted Gaussians in the view frustum:
    $$ c_p = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $c_i$ and $\alpha_i$ are the color and opacity of Gaussian $i$, and $j < i$ [S1].
7.  **Pruning:** Prune anchors if Gaussians spawned by them do not contribute to the scene, using anchor-wise opacity and view thresholds [S1].
8.  **Stylization (Optional):** To change style, apply a stylization loss using CLIP features [S1].
    *   Compute direction loss $\Delta T$ from style source $S$ (text prompt or image) and reference "A Photo":
        $$ \Delta T = \Phi_{\text{CLIP}}(S) - \Phi_{\text{CLIP}}(\text{"A Photo"}) $$
        [S1].
    *   Compute image loss $\Delta I$ from rendered image patch $\hat{I}_{Rn}$ and ground truth $I_{GT}$:
        $$ \Delta I = \Phi_{\text{CLIP}}(R(\hat{I}_{Rn})) - \Phi_{\text{CLIP}}(I_{GT}) $$
        where $R(\cdot)$ denotes random cropping and perspective distortion [S1].
    *   Calculate cosine distance $p_n = D(\Delta T, \Delta I)$ and loss $L_{\text{CLIP}} = \frac{1}{N} \sum p_n$ [S1].
    *   Pass a segmentation mask to exclude certain parts from stylization; loss is only calculated on pixels outside the mask [S1].
    *   Apply background loss $L_{BG}$ (L1 distance between background pixels of rendered image and GT image) as additional regularization [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_embedder(t, x, sigma):
    """1D Gaussian embedder psi(t, x)."""
    return np.exp(-np.square(t - x) / (2 * sigma**2))

def compute_control_encoding(x, d, s, sigma):
    """Compute d-dimensional control encoding Psi(x)."""
    t_values = np.arange(d) * s
    return np.array([gaussian_embedder(t, x, sigma) for t in t_values])

def compute_viewing_direction(anchor, camera_pos):
    """Compute viewing direction delta_xc_av."""
    diff = anchor - camera_pos
    return diff / np.linalg.norm(diff)

def predict_gaussian_parameters(anchor_features, control_encoding, viewing_direction, mlps):
    """
    Predict Gaussian parameters using MLPs.
    mlps: dict containing 'color', 'scale', 'rotation', 'offset', 'opacity' MLPs.
    """
    # Color MLP takes anchor features, viewing direction, and control encoding
    color_input = np.concatenate([anchor_features, viewing_direction, control_encoding])
    color = mlps['color'](color_input)
    
    # Scale, Rotation, Offset, Opacity MLPs take anchor features and control encoding
    base_input = np.concatenate([anchor_features, control_encoding])
    scale = mlps['scale'](base_input)
    rotation = mlps['rotation'](base_input)
    offset = mlps['offset'](base_input)
    opacity = mlps['opacity'](base_input)
    
    return color, scale, rotation, offset, opacity

def render_gaussians(gaussians, camera_params, image_shape):
    """
    Render images via tile-based rasterization.
    gaussians: list of Gaussian objects with mu, Sigma, alpha, c.
    """
    # Standard 3DGS tile-based rasterization
    # 1. Project Gaussians to 2D
    # 2. Sort by depth
    # 3. Alpha blend
    # Implementation details omitted as per standard 3DGS workflow [S1]
    pass

def train_step(images, gt_images, anchors, mlps, control_encoding, loss_weights):
    """
    Training step including L3DGS, L_reg, and optional L_CLIP.
    """
    # 1. Predict parameters for each anchor
    predicted_gaussians = []
    for anchor in anchors:
        # ... predict parameters ...
        pass
    
    # 2. Render
    rendered_images = render_gaussians(predicted_gaussians, camera_params, image_shape)
    
    # 3. Compute L3DGS
    l1_loss = np.mean(np.abs(rendered_images - gt_images))
    ssim_loss = compute_ssim(rendered_images, gt_images)
    l_3dgs = loss_weights['lambda_3dgs'] * l1_loss + (1 - loss_weights['lambda_3dgs']) * ssim_loss
    
    # 4. Compute L_reg
    # L_reg = (lambda_alpha / N) * sum(|alpha_i|) + (lambda_S / N) * sum(mean(|S_i|))
    l_reg = compute_regularization(predicted_gaussians, loss_weights['lambda_alpha'], loss_weights['lambda_S'])
    
    total_loss = l_3dgs + l_reg
    
    # 5. Optional Stylization Loss
    if stylization_required:
        # ... compute L_CLIP and L_BG ...
        pass
        
    return total_loss
```

**Parameters:**
*   $k$: Number of offsets per voxel, set to $k = 10$ [S1].
*   $\sigma$: Standard deviation of the Gaussian embedder [S1].
*   $s$: Sampling interval of the Gaussian embedder [S1].
*   $d$: Dimensionality of the control encoding [S1].
*   $\lambda_{3DGS}$: Weight for L1 loss in the 3DGS loss function [S1].
*   $\lambda_\alpha$: Hyperparameter for opacity regularization [S1].
*   $\lambda_S$: Hyperparameter for scale regularization [S1].
*   $\Phi_{\text{CLIP}}$: CLIP encoder [S1].
*   $R(\cdot)$: Random cropping and perspective distortion applied to the rendered image [S1].

**Edge Cases and Pitfalls:**
*   **Training Instability:** The feature-bank implementation of Scaffold-GS is omitted because it introduced instability during training [S1].
*   **Interpolation:** Using 1D Gaussian embedders allows interpolation between encodings, avoiding the issue where encodings far apart in input space might be closer in embedded space (as seen in frequency encoding) [S1].
*   **Inference:** During inference, only the forward flow is necessary. The color MLP must be re-evaluated whenever the viewing angle shifts, while other MLPs do not need to be inferred for every viewing direction [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Memory requirements are significantly reduced compared to storing all Gaussian parameters explicitly, as only anchor positions, corresponding features, and MLP weights need to be stored [S1].
*   **Performance:** Demonstrated on VR devices such as Meta Quest 2, Meta Quest 3, and Apple Vision Pro, where computational limits are reached early [S1].
*   **Strengths:** Allows changing all attributes of 3D Gaussians at render time; reduces memory requirements; enables context translation and stylization [S1].
*   **Limitations:** Requires MLP inference for each Gaussian parameter at render time; color MLP must be re-evaluated for every viewing direction [S1].
*   **When to use:** When dynamic context translation (e.g., RGB to thermal) or stylization is required, and memory/compute constraints on devices like VR headsets are a concern [S1].
*   **When not to use:** When static scene representation is sufficient and MLP inference overhead is unacceptable [S1].

## Background
Shape-Shifting Splats was introduced in 2026 by authors of the paper "Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR" [S1]. It builds on 3DGS [S19] and Scaffold-GS [S33], aiming to address the computational limits of VR devices by reducing resource usage while maintaining visual quality [S1].

## Sources
[S1] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
