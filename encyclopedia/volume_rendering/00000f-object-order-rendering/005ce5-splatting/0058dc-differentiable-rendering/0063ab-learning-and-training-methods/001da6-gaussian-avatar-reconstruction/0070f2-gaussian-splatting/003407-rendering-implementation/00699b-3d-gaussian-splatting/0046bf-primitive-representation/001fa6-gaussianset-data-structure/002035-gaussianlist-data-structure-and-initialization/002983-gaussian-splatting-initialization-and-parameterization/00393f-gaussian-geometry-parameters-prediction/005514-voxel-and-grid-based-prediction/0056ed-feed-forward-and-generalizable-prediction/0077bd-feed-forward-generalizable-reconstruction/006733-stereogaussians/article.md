# StereoGaussians


## Overview
StereoGaussians is a feed-forward method that predicts a metric 3D Gaussian Splatting (3DGS) representation from a single calibrated stereo pair, enabling nearby-view extrapolation without per-scene optimization [S1, unofficial]. It differs from its siblings by explicitly transferring frozen stereo features into Gaussian attribute prediction and using a two-layer Gaussian structure with an expanded image canvas to handle disoccluded and outside-field-of-view content [S1, unofficial]. Unlike methods that synthesize images directly (e.g., [LVSM](16423-splatt3r.md)) or refine inverse depth through a composer (e.g., [SHARP](12847-anysplat-pipeline.md)), StereoGaussians fixes the front-layer stereo depth and bounds image-plane offsets to provide a structural prior for geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A calibrated rectified stereo pair $(I_L, I_R)$, camera intrinsics $K$ (focal lengths $f_x, f_y$, principal point $c_x, c_y$), and baseline $B$.
*   **Outputs:** A set of $N$ Gaussians $\mathcal{G} = \{(x_i, s_i, q_i, \alpha_i, c_i)\}$ in the left-camera frame, where $x_i$ is position, $s_i$ is scale, $q_i$ is rotation (quaternion), $\alpha_i$ is opacity, and $c_i$ is color (Spherical Harmonics coefficients) [S1, unofficial].

**Data Structures:**
The method predicts two layers of Gaussians (front and back) on an expanded canvas.
*   **Front Layer:** Anchored to the stereo-derived depth.
*   **Back Layer:** Anchored to a local maximum depth in a neighborhood plus a learned offset.
*   **Gaussian Attributes:** Predicted as residuals relative to a base defined by geometry and reference color.
    *   **Scale:** Isotropic base scale $s_{0,\ell}$ followed by multiplicative updates.
    *   **Rotation:** Additive quaternion updates.
    *   **Opacity:** Updates in logit space.
    *   **Color:** Degree-2 Spherical Harmonic (SH) residuals around a reference-image DC base [S1, unofficial].

**Numbered Steps:**

1.  **Stereo Feature Extraction:**
    Pass the stereo pair $(I_L, I_R)$ through a frozen pretrained stereo encoder (e.g., FoundationStereo) to obtain:
    *   Left-reference disparity map $d$.
    *   Aligned multiscale features $\{F_s\}$ [S1, unofficial].

2.  **Metric Depth Calculation:**
    Compute metric depth $z(p)$ for each pixel $p=(u,v)$ using the horizontal focal length $f_x$ and baseline $B$:
    $$ z(p) = \text{clip}\left( \frac{f_x B}{d(p)}, z_{\min}, z_{\max} \right) $$
    where $z_{\min}, z_{\max}$ are clipping bounds [S1, unofficial].

3.  **Descriptor Generation:**
    Construct a dense full-resolution descriptor $\Phi$ by concatenating features at each scale $s$ with resized reference RGB and depth:
    $$ \Phi = D \left[ \{ F_s, \downarrow_s I_L, \downarrow_s z \}_s \right] \in \mathbb{R}^{C \times H \times W} $$
    where $D$ is a DPT-style decoder, $[\cdot, \cdot]$ denotes channel concatenation, and $\downarrow_s$ resizes inputs to feature scale $s$ (e.g., $s=4$ implies $H/4 \times W/4$ resolution) [S1, unofficial].
    *   *Implementation Detail:* At each scale, concatenate encoder features with resized RGB (bilinear) and depth (nearest-neighbor). Project to 128 channels via $1\times1$ convolution, Group Normalization, and GELU. Use residual convolution units and coarse-to-fine fusion to yield the 128-channel descriptor $\Phi$ at original resolution [S1, unofficial].

4.  **Canvas Extension (Extrapolation-Aware Prediction):**
    Extend the reference RGB $I_L$, processed depth $z$, and descriptor $\Phi$ by edge-replication padding $P$ on each side to size $(H + 2P) \times (W + 2P)$. Denote extended inputs as $\tilde{I}_L, \tilde{z}, \tilde{\Phi}$.
    *   Update canvas intrinsics: Focal lengths remain unchanged. Principal point shifts to $\tilde{c}_x = c_x + P, \tilde{c}_y = c_y + P$ [S1, unofficial].

5.  **Gaussian Attribute Prediction:**
    Pass extended inputs through a convolutional predictor $H$:
    $$ (\theta_{\text{front}}, \theta_{\text{back}}, \Delta z) = H([\tilde{I}_L, \tilde{z}, \tilde{\Phi}]) $$
    *   **Regressor:** Two $3\times3$ convolutions with intermediate GELU map inputs to a 128-channel tensor $h$.
    *   **Output Branches:** Each branch processes $[h, \tilde{I}_L, \tilde{\Phi}]$ using a $3\times3$ convolution, Group Normalization, GELU, and a final $3\times3$ convolution.
    *   **Front/Back Branches:** Output 37 channels each: opacity (1), image-plane offset (2), scale (3), quaternion (4), degree-2 SH coefficients (27) [S1, unofficial].
    *   **Back-Depth Branch:** Outputs a scalar $\Delta z$ [S1, unofficial].

6.  **Front Layer Geometry:**
    *   **Depth:** $z_{\text{front}} = \tilde{z}$.
    *   **Image-Plane Offsets:** Compute $(\delta_{u,\ell}, \delta_{v,\ell}) = \sigma(o_\ell) - 1/2$, where $o_\ell$ is the 2-channel offset output for layer $\ell$. Bound offsets to half a canvas pixel per axis [S1, unofficial].
    *   **Center:** Calculate center $x_{\text{front}}(p)$ using canvas intrinsics $\tilde{K}$:
        $$ x_{\text{front}}(p) = z_{\text{front}}(p) \tilde{K}^{-1} [u + \delta_{u,\text{front}}, v + \delta_{v,\text{front}}, 1]^\top $$
    *   **Base Scale:** Isotropic base scale $s_{0,\text{front}} = \kappa z_{\text{front}} / (f_x f_y)$ [S1, unofficial].

7.  **Back Layer Geometry:**
    *   **Depth:** Compute local maximum depth in a $9\times9$ neighborhood $\mathcal{N}_9(p)$ plus a learned positive offset:
        $$ z_{\text{back}}(p) = \max_{q \in \mathcal{N}_9(p)} \tilde{z}(q) + \text{softplus}(\Delta z(p)) $$
    *   **Center:** Calculate center $x_{\text{back}}(p)$ using $z_{\text{back}}$ and its own image-plane offsets (bounded similarly to front) [S1, unofficial].
    *   **Base Scale:** $s_{0,\text{back}} = \kappa z_{\text{back}} / (f_x f_y)$ [S1, unofficial].

8.  **Attribute Composition:**
    *   **Scale:** Apply bounded multiplicative updates to $s_{0,\ell}$.
    *   **Rotation:** Apply additive quaternion updates to a base rotation.
    *   **Opacity:** Apply updates in logit space to base opacity.
    *   **Color:** Add degree-2 SH residuals to a reference-image DC color base [S1, unofficial].

9.  **Rendering and Training:**
    *   Concatenate front and back layers.
    *   Render target views using differentiable Gaussian rasterization.
    *   **No Edge Mask:** The back layer is not restricted by an edge mask; learned attributes and visibility determine contributions [S1, unofficial].
    *   **Loss:** Minimize $L = \text{MSE}(I_T, \hat{I}_T) + 0.05 \text{LPIPS}_{\text{VGG}}(I_T, \hat{I}_T)$, where $I_T$ is the target image and $\hat{I}_T$ is the rendered prediction [S1, unofficial].
    *   **Training:** Train the predictor $H$ and decoder $D$ using AdamW. Keep the stereo encoder frozen [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class StereoGaussians:
    def __init__(self, encoder, predictor, decoder, padding=16):
        self.encoder = encoder # Frozen
        self.predictor = predictor # Trainable
        self.decoder = decoder # Trainable
        self.padding = padding
        self.kappa = 1.0 # Constant for base scale

    def forward(self, I_L, I_R, K, B, z_min=0.1, z_max=100.0):
        # 1. Stereo Feature Extraction
        # Returns disparity d, multiscale features {F_s}
        d, F_s = self.encoder(I_L, I_R)
        
        # 2. Metric Depth Calculation
        fx = K[0, 0]
        # Clip depth to valid range
        z = torch.clamp((fx * B) / d, z_min, z_max)
        
        # 3. Descriptor Generation
        # Resize RGB and Depth for each scale s
        # Concatenate F_s, downsampled I_L, downsampled z
        # Pass through DPT-style decoder D
        # D projects to 128 channels, uses GN, GELU, residual units
        Phi = self.decoder(F_s, I_L, z)
        
        # 4. Canvas Extension
        P = self.padding
        I_L_ext = F.pad(I_L, (P, P, P, P), mode='replicate')
        z_ext = F.pad(z, (P, P, P, P), mode='replicate')
        Phi_ext = F.pad(Phi, (P, P, P, P), mode='replicate')
        
        # Update intrinsics for extended canvas
        # cx_new = cx + P, cy_new = cy + P
        # fx, fy remain unchanged
        K_ext = K.clone()
        K_ext[0, 2] += P
        K_ext[1, 2] += P
        
        # 5. Gaussian Attribute Prediction
        # Predictor H takes [I_L_ext, z_ext, Phi_ext]
        theta_front, theta_back, delta_z = self.predictor(I_L_ext, z_ext, Phi_ext)
        
        # theta_front/back shape: [B, 37, H, W]
        # delta_z shape: [B, 1, H, W]
        
        # 6. Front Layer Geometry
        z_front = z_ext
        # Extract offsets from theta_front
        # Channels: opacity(1), offset(2), scale(3), quat(4), SH(27)
        offset_front = theta_front[:, 1:3, :, :] # 2 channels
        # Sigmoid to map to [0,1], then shift to [-0.5, 0.5]
        delta_uv_front = torch.sigmoid(offset_front) - 0.5
        # Bound to half canvas pixel (implementation specific bound)
        # delta_uv_front = torch.clamp(delta_uv_front, -0.5, 0.5)
        
        # Compute centers using extended intrinsics
        # x = z * K_inv * [u + du, v + dv, 1]
        # This requires meshgrid of u, v coordinates
        centers_front = self.compute_centers(z_front, K_ext, delta_uv_front)
        
        # Base scale
        s0_front = self.kappa * z_front / (fx * K[1, 1]) # fy = K[1,1]
        
        # 7. Back Layer Geometry
        # Local max in 9x9 neighborhood
        # Note: F.max_pool2d with kernel 9x9, stride 1, padding 4
        z_max_pool = F.max_pool2d(z_ext, kernel_size=9, stride=1, padding=4)
        # Softplus for positive offset
        delta_z_sp = F.softplus(delta_z)
        z_back = z_max_pool + delta_z_sp
        
        # Extract offsets from theta_back
        offset_back = theta_back[:, 1:3, :, :]
        delta_uv_back = torch.sigmoid(offset_back) - 0.5
        # delta_uv_back = torch.clamp(delta_uv_back, -0.5, 0.5)
        
        centers_back = self.compute_centers(z_back, K_ext, delta_uv_back)
        s0_back = self.kappa * z_back / (fx * K[1, 1])
        
        # 8. Attribute Composition
        # Extract remaining attributes from theta_front/back
        # Scale updates, quat updates, opacity logits, SH residuals
        # Apply composition rules (multiplicative scale, additive quat, logit opacity, SH residual)
        # Base color is DC from I_L
        # Return full Gaussian set
        gaussians = self.compose_gaussians(
            centers_front, s0_front, theta_front[:, 3:, :, :], # scale, quat, SH
            centers_back, s0_back, theta_back[:, 3:, :, :], # scale, quat, SH
            theta_front[:, 0, :, :], # opacity front
            theta_back[:, 0, :, :], # opacity back
            I_L_ext # For DC color base
        )
        
        return gaussians

    def compute_centers(self, z, K_inv, delta_uv):
        # z: [B, 1, H, W]
        # delta_uv: [B, 2, H, W]
        # K_inv: [3, 3]
        # Returns: [B, 3, H, W]
        B, _, H, W = z.shape
        # Create meshgrid for u, v
        u = torch.arange(W, device=z.device).float().view(1, 1, 1, W)
        v = torch.arange(H, device=z.device).float().view(1, 1, H, 1)
        
        # Add offsets
        u_shifted = u + delta_uv[:, 0, :, :]
        v_shifted = v + delta_uv[:, 1, :, :]
        
        # Homogeneous coordinates
        coords = torch.stack([u_shifted, v_shifted, torch.ones_like(u)], dim=1) # [B, 3, H, W]
        
        # Apply K_inv
        # K_inv is [3,3], coords is [B, 3, H, W]
        # Result is [B, 3, H, W]
        xyz = torch.matmul(K_inv.unsqueeze(0).unsqueeze(-1).unsqueeze(-1), coords.unsqueeze(-1)).squeeze(-1)
        
        # Multiply by depth
        xyz = xyz * z
        
        return xyz

    def compose_gaussians(self, ...):
        # Placeholder for attribute composition logic
        # Returns list of Gaussian objects or tensors
        pass
```

**Parameters:**
*   **Padding $P$:** 16 pixels (default) [S1, unofficial].
*   **Learning Rate:** $4 \times 10^{-4}$ for AdamW [S1, unofficial].
*   **Weight Decay:** 0.01 [S1, unofficial].
*   **Momentum ($\beta_1, \beta_2$):** (0.9, 0.999) [S1, unofficial].
*   **Scheduler:** OneCycleLR with 2,700 warm-up steps (9% of training), initial LR division factor 25, final LR division factor 100 [S1, unofficial].
*   **Training Steps:** 30,000 [S1, unofficial].
*   **Batch Size:** 32 [S1, unofficial].
*   **Decoder Channels:** 128 [S1, unofficial].
*   **Output Channels per Branch:** 37 (1 opacity, 2 offset, 3 scale, 4 quaternion, 27 SH) [S1, unofficial].
*   **Input Normalization:** ImageNet normalization after rescaling RGB from [0, 1] to [0, 255] [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Padding:** Without padding, empty target-border bands appear in rendered views; padding fills these newly exposed regions [S1, unofficial].
*   **Fixed Front-Layer Depth:** Limits stereo-error correction [S1, unofficial].
*   **Teacher Artifacts:** Can persist despite reconstruction-quality filtering [S1, unofficial].
*   **Descriptor Removal:** Without the DPT descriptor, tabletop distortions and foreground-boundary artifacts occur [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Inference is feed-forward, dependent on the encoder and predictor network depth. Training requires differentiable rendering, which is computationally intensive.
*   **Memory Complexity:** Requires storing two layers of Gaussians and intermediate features for the decoder and predictor.
*   **Performance:**
    *   On StereoNVS: 21.261 PSNR, 0.9447 SSIM, 0.0818 LPIPS [S1, unofficial].
    *   On ReplicaGS: 30.266 PSNR, 0.9447 SSIM, 0.0818 LPIPS [S1, unofficial].
*   **Scaling:** Uses 16-pixel padding by default. 64-pixel padding improves StereoNVS PSNR/LPIPS but lowers ReplicaGS PSNR [S1, unofficial].
*   **Strengths:** Handles disoccluded content (back layer) and outside-field-of-view content (padding). Generalizes to unseen scenes.
*   **Limitations:** Fixed front-layer depth limits error correction. Performance degrades without padding or descriptor. Evaluation focuses on nearby views [S1, unofficial].

## Background
StereoGaussians was introduced in 2026 (preprint) by authors associated with the SceneSplat-Stereo dataset. It builds on the principles of feed-forward 3DGS and leverages frozen stereo encoders to transfer geometric priors to Gaussian attribute prediction [S1, unofficial]. It is used on StereoNVS and ReplicaGS datasets [S1, unofficial].

## Sources
[S1] StereoGaussians: Feed-Forward 3D Gaussian Splatting from Stereo Images, https://arxiv.org/abs/2609.38592, fetched 2026-10-29 (unofficial)
