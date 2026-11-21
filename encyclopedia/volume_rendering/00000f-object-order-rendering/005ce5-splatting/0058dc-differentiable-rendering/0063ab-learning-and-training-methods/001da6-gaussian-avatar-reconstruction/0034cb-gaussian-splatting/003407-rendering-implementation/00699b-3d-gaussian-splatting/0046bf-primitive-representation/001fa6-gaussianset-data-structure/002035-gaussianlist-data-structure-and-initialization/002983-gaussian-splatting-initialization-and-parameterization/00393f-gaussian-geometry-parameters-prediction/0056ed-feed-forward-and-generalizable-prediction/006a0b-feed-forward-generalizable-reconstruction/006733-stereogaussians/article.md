# StereoGaussians


## Overview
StereoGaussians is a feed-forward method that predicts a metric 3D Gaussian Splatting (3DGS) representation from a single calibrated stereo pair, enabling nearby-view extrapolation without per-scene optimization [S1, unofficial]. It differs from its siblings by explicitly transferring frozen stereo features into Gaussian attribute prediction and using a two-layer Gaussian structure with an expanded image canvas to handle disoccluded content and outside-field-of-view regions [S1, unofficial]. Unlike methods that synthesize images directly (e.g., [LVSM](16425-lvsm.md)) or refine inverse depth through a composer (e.g., [SHARP](16422-sharp.md)), StereoGaussians fixes the front-layer stereo depth and bounds image-plane offsets to provide a structural prior for geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A calibrated rectified stereo pair (left image $I_L$, right image $I_R$), camera intrinsics $K$ (focal lengths $f_x, f_y$, principal point $c_x, c_y$), and baseline $B$ [S1, unofficial].
*   **Outputs:** A set of $N$ Gaussians $\mathcal{G} = \{(x_i, s_i, q_i, \alpha_i, c_i)\}_{i=1}^N$ in the left-camera frame, where $x_i$ is position, $s_i$ is scale, $q_i$ is rotation (quaternion), $\alpha_i$ is opacity, and $c_i$ is color (Spherical Harmonics coefficients) [S1, unofficial].

**Data Structures:**
The method predicts two layers of Gaussians (front and back) for every pixel on an expanded canvas.
*   **Front Layer:** Anchored to the stereo-derived depth.
*   **Back Layer:** Anchored to a local maximum depth in a neighborhood plus a learned offset.
*   **Gaussian Attributes per Pixel:**
    *   Opacity: 1 channel (logit space).
    *   Image-plane offset: 2 channels ($\delta_u, \delta_v$).
    *   Scale: 3 channels (isotropic base scale + multiplicative residual).
    *   Rotation: 4 channels (quaternion residual).
    *   Color: 27 channels (degree-2 Spherical Harmonics residuals).
    *   Total per layer: 37 channels [S1, unofficial].
    *   Back-layer depth offset: 1 scalar channel [S1, unofficial].

**Steps:**

1.  **Stereo Feature Extraction:**
    Pass the stereo pair through a frozen pretrained stereo encoder (e.g., FoundationStereo) to obtain a left-reference disparity map $d$, aligned multiscale features $\{F_s\}$, and other context features [S1, unofficial]. Compute metric depth $z(p)$ for each pixel $p=(u,v)$:
    $$ z(p) = \text{clip}\left( \frac{f_x B}{d(p)}, z_{\min}, z_{\max} \right) $$
    where $f_x$ is the horizontal focal length in pixels, $B$ is the baseline, and $z_{\min}, z_{\max}$ are clipping bounds [S1, unofficial].

2.  **Descriptor Generation:**
    Construct a dense full-resolution descriptor $\Phi$ by concatenating stereo features with resized reference RGB and depth at each scale $s$:
    $$ \Phi = D \left[ \{F_s, \downarrow_s I_L, \downarrow_s z\}_s \right] \in \mathbb{R}^{C \times H \times W} $$
    where $D$ is a DPT-style decoder, $[\cdot, \cdot]$ denotes channel concatenation, and $\downarrow_s$ resizes inputs to feature scale $s$ (e.g., $s=4$ implies $H/4 \times W/4$) [S1, unofficial].
    *   *Implementation Detail:* At each scale, concatenate encoder features with resized RGB (bilinear) and depth (nearest-neighbor). Project to 128 channels via $1\times1$ convolution, Group Normalization, and GELU. Use residual convolution units and coarse-to-fine fusion blocks to yield the 128-channel descriptor $\Phi$ at original resolution [S1, unofficial].

3.  **Canvas Expansion:**
    Extend the reference RGB $I_L$, processed depth $z$, and descriptor $\Phi$ by edge-replication padding $P$ (default 16 pixels) on each side to size $(H+2P) \times (W+2P)$ [S1, unofficial].
    *   Update canvas intrinsics: Focal lengths remain unchanged. Principal point shifts to $\tilde{c}_x = c_x + P$, $\tilde{c}_y = c_y + P$ [S1, unofficial].
    *   Let the extended inputs be denoted with tildes: $\tilde{I}_L, \tilde{z}, \tilde{\Phi}$ [S1, unofficial].

4.  **Gaussian Attribute Prediction:**
    Pass extended inputs through a convolutional predictor $H$ to obtain attribute residuals:
    $$ (\theta_{\text{front}}, \theta_{\text{back}}, \Delta z) = H([\tilde{I}_L, \tilde{z}, \tilde{\Phi}]) $$
    *   *Regressor:* Two $3\times3$ convolutions with intermediate GELU map inputs to a 128-channel tensor $h$ [S1, unofficial].
    *   *Output Branches:* Each branch processes $[h, \tilde{I}_L, \tilde{\Phi}]$ using a $3\times3$ convolution, Group Normalization, GELU, and a final $3\times3$ convolution [S1, unofficial].
    *   *Front/Back Branches:* Output 37 channels each (opacity, offset, scale, quaternion, SH) [S1, unofficial].
    *   *Depth Offset Branch:* Outputs a scalar $\Delta z$ [S1, unofficial].

5.  **Gaussian Center and Depth Calculation:**
    For each pixel $p$ on the expanded canvas:
    *   **Front Layer Depth:** $z_{\text{front}}(p) = \tilde{z}(p)$ [S1, unofficial].
    *   **Front Layer Center:** Compute image-plane offsets $(\delta_{u,\text{front}}, \delta_{v,\text{front}}) = \sigma(o_{\text{front}}) - 0.5$, bounded to half a canvas pixel. Center is:
        $$ x_{\text{front}}(p) = z_{\text{front}}(p) \tilde{K}^{-1} [u + \delta_{u,\text{front}}, v + \delta_{v,\text{front}}, 1]^\top $$
        where $\tilde{K}$ is the expanded canvas intrinsic matrix [S1, unofficial].
    *   **Back Layer Depth:** Compute local maximum depth in a $9\times9$ neighborhood $\mathcal{N}_9(p)$ plus a learned positive offset:
        $$ z_{\text{back}}(p) = \max_{q \in \mathcal{N}_9(p)} \tilde{z}(q) + \text{softplus}(\Delta z(p)) $$
        [S1, unofficial].
    *   **Back Layer Center:** Similar to front, using $z_{\text{back}}(p)$ and its specific offsets [S1, unofficial].

6.  **Gaussian Attribute Composition:**
    Attributes are predicted as residuals around a base defined by the reference image and geometry [S1, unofficial]:
    *   **Base Scale:** Isotropic base scale $s_{0,\ell} = \kappa \frac{z_\ell}{f_x f_y}$, following the metric pixel footprint [S1, unofficial].
    *   **Residuals:**
        *   Scale: Bounded multiplicative updates.
        *   Rotation: Additive quaternion updates.
        *   Opacity: Updates in logit space.
        *   Color: Degree-2 Spherical Harmonic (SH) residuals around a reference-image DC base [S1, unofficial].

7.  **Rendering and Training:**
    Concatenate front and back layers. Render target views using differentiable Gaussian rasterization. The back layer is unmasked, allowing learned attributes and visibility to determine contributions [S1, unofficial].
    *   **Loss:** Minimize $\mathcal{L} = \text{MSE}(I_T, \hat{I}_T) + 0.05 \text{LPIPS}_{\text{VGG}}(I_T, \hat{I}_T)$, where $I_T$ is the target image and $\hat{I}_T$ is the rendered prediction [S1, unofficial].
    *   **Optimization:** AdamW optimizer, LR $4 \times 10^{-4}$, weight decay 0.01, momentum $(0.9, 0.999)$. OneCycleLR schedule with 2,700 warm-up steps, initial/final division factors 25/100. Total 30,000 steps [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class StereoGaussiansPredictor(nn.Module):
    def __init__(self, encoder, decoder_channels=128, padding=16):
        super().__init__()
        self.encoder = encoder  # Frozen stereo encoder
        self.decoder = DPTDecoder(in_channels=encoder.out_channels, out_channels=decoder_channels)
        self.padding = padding
        
        # Predictor components
        self.regressor = nn.Sequential(
            nn.Conv2d(decoder_channels + 3 + 1, 128, 3, padding=1), # +3 for RGB, +1 for depth
            nn.GELU(),
            nn.Conv2d(128, 128, 3, padding=1),
            nn.GELU()
        )
        
        # Output branches
        self.front_branch = nn.Sequential(
            nn.Conv2d(128 + 3 + 1, 128, 3, padding=1), # +3 RGB, +1 depth
            nn.GroupNorm(8, 128),
            nn.GELU(),
            nn.Conv2d(128, 37, 3, padding=1) # 1 op, 2 off, 3 scale, 4 quat, 27 SH
        )
        
        self.back_branch = nn.Sequential(
            nn.Conv2d(128 + 3 + 1, 128, 3, padding=1),
            nn.GroupNorm(8, 128),
            nn.GELU(),
            nn.Conv2d(128, 37, 3, padding=1)
        )
        
        self.depth_offset_branch = nn.Sequential(
            nn.Conv2d(128 + 3 + 1, 128, 3, padding=1),
            nn.GroupNorm(8, 128),
            nn.GELU(),
            nn.Conv2d(128, 1, 3, padding=1)
        )

    def forward(self, I_L, I_R, K, B, z_min, z_max):
        # 1. Stereo Feature Extraction
        # encoder returns disparity d and multiscale features {F_s}
        d, F_s = self.encoder(I_L, I_R)
        
        # Compute metric depth
        fx = K[0, 0]
        z = torch.clamp((fx * B) / d, z_min, z_max)
        
        # 2. Descriptor Generation
        # Resize RGB and depth for each scale s
        # DPT decoder fuses features
        Phi = self.decoder(F_s, I_L, z) # Output: [B, C, H, W]
        
        # 3. Canvas Expansion
        P = self.padding
        I_L_ext = F.pad(I_L, (P, P, P, P), mode='replicate')
        z_ext = F.pad(z, (P, P, P, P), mode='replicate')
        Phi_ext = F.pad(Phi, (P, P, P, P), mode='replicate')
        
        # 4. Gaussian Attribute Prediction
        # Regressor input: Extended RGB, Depth, Descriptor
        reg_input = torch.cat([I_L_ext, z_ext, Phi_ext], dim=1)
        h = self.regressor(reg_input)
        
        # Branch inputs: h, Extended RGB, Extended Descriptor
        branch_input = torch.cat([h, I_L_ext, Phi_ext], dim=1)
        
        theta_front = self.front_branch(branch_input)
        theta_back = self.back_branch(branch_input)
        delta_z = self.depth_offset_branch(branch_input)
        
        # 5. Gaussian Center and Depth Calculation
        # Front Layer
        z_front = z_ext
        # Offsets: sigma(o) - 0.5, bounded to 0.5
        off_front = torch.sigmoid(theta_front[:, 1:3, :, :]) - 0.5
        off_front = torch.clamp(off_front, -0.5, 0.5)
        
        # Back Layer
        # Local max in 9x9 neighborhood
        # Note: In practice, use F.max_pool2d with kernel 9
        z_max_pool = F.max_pool2d(z_ext, kernel_size=9, stride=1, padding=4)
        z_back = z_max_pool + F.softplus(delta_z)
        
        off_back = torch.sigmoid(theta_back[:, 1:3, :, :]) - 0.5
        off_back = torch.clamp(off_back, -0.5, 0.5)
        
        # 6. Construct Gaussians
        # This part typically happens in the rendering step or a separate constructor
        # returning the parameters needed for the rasterizer
        return {
            'z_front': z_front,
            'z_back': z_back,
            'off_front': off_front,
            'off_back': off_back,
            'theta_front': theta_front,
            'theta_back': theta_back,
            'delta_z': delta_z,
            'K_ext': K # With shifted principal point
        }
```

## Operating characteristics
*   **Time Complexity:** Inference is $O(HW)$ per stereo pair, dominated by the convolutional predictor and encoder. It is significantly faster than per-scene optimization methods which require iterative gradient descent [S1, unofficial].
*   **Memory Complexity:** Memory usage scales with image resolution $H \times W$ due to the dense prediction of Gaussian attributes on the expanded canvas. The expanded canvas increases memory by a factor of $(1 + 2P/H)(1 + 2P/W)$ [S1, unofficial].
*   **Performance:** On StereoNVS, StereoGaussians achieves 21.261 PSNR, 0.9447 SSIM, and 0.0818 LPIPS. On ReplicaGS, it achieves 30.266 PSNR, 0.9447 SSIM, and 0.0818 LPIPS [S1, unofficial].
*   **Strengths:**
    *   Handles disoccluded content via the back layer (removing it reduces PSNR by 0.180 dB on StereoNVS and 3.826 dB on ReplicaGS) [S1, unofficial].
    *   Handles outside-field-of-view content via canvas padding (removing 16-pixel padding lowers PSNR by 3.168 dB on StereoNVS and 8.732 dB on ReplicaGS) [S1, unofficial].
    *   Generalizes to unseen scenes without per-scene tuning [S1, unofficial].
*   **Limitations:**
    *   Fixed front-layer depth limits stereo-error correction [S1, unofficial].
    *   Teacher artifacts from the training data (SceneSplat-Stereo) can persist [S1, unofficial].
    *   Evaluation is focused on nearby views; larger viewpoint changes are not fully addressed [S1, unofficial].
    *   Composition improves perceptual metrics (LPIPS/SSIM) but lowers PSNR compared to the composer-free variant (0.614 dB higher PSNR for composer-free) [S1, unofficial].

## Background
StereoGaussians was introduced by [S1, unofficial] to address the need for nearby-view extrapolation in stereo-camera applications, XR headsets, and spatial video. It builds on the principles of feed-forward 3DGS [Parent] and leverages frozen pretrained stereo networks (like FoundationStereo) to transfer geometric priors into Gaussian attribute prediction. The method was trained on a new dataset, SceneSplat-Stereo, constructed from quality-filtered 3DGS teachers, pairing stereo inputs with nearby target views across 803 training scenes [S1, unofficial]. It is used on StereoNVS and ReplicaGS datasets [S1, unofficial].

## Sources
[S1] StereoGaussians: Feed-Forward 3D Gaussian Splatting from Stereo Images, https://arxiv.org/abs/2609.38592, fetched 2026-10-29 (unofficial)
