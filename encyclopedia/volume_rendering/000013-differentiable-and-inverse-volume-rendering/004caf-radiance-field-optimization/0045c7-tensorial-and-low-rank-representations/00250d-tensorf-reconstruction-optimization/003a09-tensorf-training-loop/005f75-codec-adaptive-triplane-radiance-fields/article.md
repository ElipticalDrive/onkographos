# Codec-Adaptive TriPlane Radiance Fields

Also called: CATRF, SCL, Standard-Codec-in-the-Loop

## Overview
Codec-Adaptive TriPlane Radiance Fields (CATRF) is a compression framework for plane-factorized radiance fields that optimizes neural features directly in the decoded space of a standard video or image codec [S1]. It solves the problem of severe rendering degradation at low bitrates by integrating a real codec round-trip (JPEG, VP9, HEVC, or AV1) into the training loop via a Straight-Through Estimator (STE) [S1]. Unlike Codec-Agnostic (CA) methods that compress features after training, or Learned-Codec-in-the-Loop (LCL) methods that require shipping scene-specific neural decoder parameters, CATRF adapts features to real codec distortions without introducing any learned codec parameters or entropy models [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained plane-factorized radiance field backbone (TensoRF for static scenes, DVGO for dynamic scenes) [S1].
*   **Output:** A compressed bitstream of the radiance field features decodable by standard codecs, and the ability to render the scene from this compressed representation [S1].

**Data Structures:**
The method operates on plane-factorized radiance fields [S1]. The scene is represented by three orthogonal feature planes $P = \{P_{xy}, P_{xz}, P_{yz}\}$ and a 3D density grid $D$ [S1].
*   Each feature plane $P_{ax}$ (where $ax \in \{xy, xz, yz\}$) has layout $P_{ax} \in \mathbb{R}^{C \times H \times W}$, where $C$ is the channel dimension, and $H, W$ are height and width [S1].
*   The density grid is $D \in \mathbb{R}^{D_y \times D_x \times D_z}$ [S1].
*   These tensors are quantized and packed into 2D canvases (images with integer pixel values) suitable for standard codecs [S1]. The default packing format is `FlattenGray`, which avoids cross-channel mixing to reduce codec artifact sensitivity [S1].
*   Quantization uses AbsMax quantization by default [S1]. Channelwise quantization is avoided as it increases bitrate by ~45x [S1].

**Parameters:**
*   **Codec:** Off-the-shelf codecs (JPEG, VP9, HEVC, AV1) [S1].
*   **Total-variation regularization weight ($\lambda_{tv}$):** Controls the trade-off between codec compatibility and high-frequency detail preservation [S1]. Default value: $5 \times 10^{-5}$ [S1].
*   **Cache refresh interval ($M$):** The number of training steps before the cached decoded planes are refreshed [S1]. Default value: 128 [S1].

**Algorithm Steps:**
The method employs a two-stage pipeline [S1]:
1.  **Backbone Training:** Train the vanilla backbone (TensoRF for static, DVGO for dynamic) for each scene or video segment to convergence [S1].
2.  **SCL Finetuning:** Finetune the radiance fields using the Standard-Codec-in-the-Loop (SCL) framework under a fixed codec configuration [S1].

During the SCL finetuning phase, each training step proceeds as follows:
1.  **Quantize and Pack:** Quantize the multi-channel appearance planes and density grid into 2D canvases using AbsMax quantization and `FlattenGray` packing [S1].
2.  **Codec Round-Trip:** Encode the canvases using the selected standard codec, then decode them to obtain reconstructed features [S1].
3.  **Unpack and Dequantize:** Unpack the decoded canvases back into feature planes and dequantize them [S1].
4.  **Render and Loss:** Render the radiance fields from the decoded features to compute the reconstruction loss [S1].
5.  **Regularization:** Apply spatial total-variation and temporal-consistency regularization terms to encourage smoother, codec-friendly patterns [S1]. The total loss includes the reconstruction loss and $\lambda_{tv} \times \text{TV}(P)$ [S1].
6.  **Backpropagation:** Pass gradients across the entire codec round-trip using a Straight-Through Estimator (STE) [S1]. The codec is treated as a black box with an identity gradient (i.e., $\frac{\partial L}{\partial \text{Input}} = \frac{\partial L}{\partial \text{Output}}$) [S1].
7.  **Caching:** Cache the decoded planes across training steps to avoid per-minibatch codec round trips [S1]. The cache is refreshed every $M$ steps [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class StraightThroughCodec(nn.Module):
    """Straight-Through Estimator for non-differentiable codec round-trip."""
    def forward(self, x):
        # Forward: encode then decode
        bitstream = codec_encode(x)
        return codec_decode(bitstream)

def codec_encode(x):
    """Simulate codec encoding: quantize to uint8."""
    return torch.clamp(x.round(), 0, 255).to(torch.uint8)

def codec_decode(bitstream):
    """Simulate codec decoding: convert back to float."""
    return bitstream.float()

def quantize_absmax(tensor):
    """AbsMax quantization: scale to [0, 255] uint8."""
    abs_max = tensor.abs().max()
    if abs_max == 0:
        abs_max = 1e-8
    scale = abs_max / 255.0
    quantized = (tensor / scale).round().clamp(0, 255).to(torch.uint8)
    return quantized, scale

def dequantize_absmax(quantized, scale):
    """Dequantize: scale back to original range."""
    return quantized.float() * scale

def pack_flatten_gray(planes):
    """Pack list of (C, H, W) planes into a single 2D canvas (H, W*C)."""
    return torch.cat([p.flatten(0, 1) for p in planes], dim=1)

def unpack_flatten_gray(canvas, num_planes, channels_per_plane):
    """Unpack canvas back to list of planes."""
    planes = []
    for i in range(num_planes):
        start = i * channels_per_plane
        end = start + channels_per_plane
        planes.append(canvas[:, start:end].reshape(-1, canvas.shape[0], canvas.shape[1]))
    return planes

def mse_loss(pred, target):
    """Mean Squared Error loss."""
    return torch.mean((pred - target) ** 2)

def tv_loss(tensor):
    """Spatial total variation regularization."""
    dx = tensor[:, :, 1:] - tensor[:, :, :-1]
    dy = tensor[:, 1:, :] - tensor[:, :-1, :]
    return dx.abs().mean() + dy.abs().mean()

class CATRF:
    def __init__(self, backbone, optimizer, lambda_tv=5e-5, cache_refresh_interval=128):
        self.backbone = backbone
        self.optimizer = optimizer
        self.codec = StraightThroughCodec()
        self.lambda_tv = lambda_tv
        self.cache_refresh_interval = cache_refresh_interval
        self.step_count = 0
        self.cached_canvases = None
        self.cached_scales = None

    def train_step(self, rays, gt_images):
        # 1. Get planes and density from backbone
        planes, density_grid = self.backbone.get_planes_and_density()

        # 2. Check cache
        if self.cached_canvases is None or self.step_count % self.cache_refresh_interval == 0:
            # Quantize and pack
            self.cached_scales = []
            canvases = []
            for p in planes:
                q, s = quantize_absmax(p)
                self.cached_scales.append(s)
                canvases.append(q)
            # Pack density grid similarly
            q_d, s_d = quantize_absmax(density_grid)
            self.cached_scales.append(s_d)
            canvases.append(q_d)

            # Codec round-trip
            bitstream = self.codec(canvases)
            self.cached_canvases = bitstream

        # Unpack and dequantize
        decoded_canvases = self.cached_canvases
        decoded_planes = []
        for i, canvas in enumerate(decoded_canvases[:-1]):
            decoded_planes.append(dequantize_absmax(canvas, self.cached_scales[i]))
        decoded_density = dequantize_absmax(decoded_canvases[-1], self.cached_scales[-1])

        # 3. Render
        rendered_images = self.backbone.render(decoded_planes, decoded_density, rays)

        # 4. Compute Loss
        recon_loss = mse_loss(rendered_images, gt_images)
        tv_reg = tv_loss(torch.cat(decoded_planes, dim=0))
        total_loss = recon_loss + self.lambda_tv * tv_reg

        # 5. Backpropagate
        total_loss.backward()
        self.optimizer.step()
        self.optimizer.zero_grad()
        self.step_count += 1
        return total_loss.item()
```

**Edge Cases and Pitfalls:**
*   **Codec Sensitivity:** Feature planes are highly sensitive to codec distortions (quantization, block transforms, in-loop filtering) [S1]. Training without codec exposure (Codec-Agnostic) causes severe rendering degradation at low bitrates [S1].
*   **Bitrate vs. Quality:** Standard codecs are sensitive to artifacts [S1]. CATRF adapts to various quality levels, operating at lower bitrates with modest detail loss or higher bitrates for sharper edges [S1].
*   **Quantization Choice:** Channelwise quantization increases bitrate by ~45x and is unsuitable for streaming [S1]. FlattenGray with AbsMax quantization is the default [S1].
*   **Cache Refresh:** The cache refresh interval $M$ balances training time and bitrate [S1]. $M=128$ is the empirical default [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The memory footprint is dominated by the encoded feature planes [S1]. On the NeRF Synthetic dataset, CATRF uses 70.04 MB total, with 69.95 MB for encoded feature planes, 0.05 MB for feature vectors, and 0.04 MB for hash embeddings [S1]. It requires 0 MB for neural codec parameters, entropy model parameters, anchor features, and MLPs [S1].
*   **Measured Performance:**
    *   **Static Scenes:** CATRF-JPEG achieves a 43% BD-rate reduction over state-of-the-art LCL compression methods [S1]. On NeRF Synthetic, it achieves a 23% bitrate reduction over CNC at matched PSNR [S1]. On Tanks and Temples, it achieves a 43% bitrate saving over CNC [S1].
    *   **Dynamic Scenes:** CATRF achieves BD-rate reductions of 40.5% (AV1), 63.3% (HEVC), and 51.8% (VP9) compared to TeTriRF on the N3D dataset [S1]. It achieves 17% bitrate savings over HAC++ and 39% BD-rate reduction over GIFStream [S1].
    *   **Decoding Speed:** CATRF-HEVC reconstructs radiance fields at 138 FPS, compared to 2.94 FPS for GIFStream and 0.35 FPS for HAC++ [S1]. This is because CATRF reconstruction is neural-free, relying only on standard-codec decoding [S1].
*   **Strengths:** No need to transmit scene-specific decoding parameters (unlike LCL) [S1]. Neural-free reconstruction (unlike 3DGS methods) [S1]. High decoding throughput [S1].
*   **Limitations:** Requires a two-stage training process [S1]. Relies on the quality of the underlying backbone (TensoRF/DVGO) [S1].
*   **When to Use:** Volumetric content delivery, free-viewpoint video streaming, 3D video playback, and telepresence [S1].

## Background
CATRF was introduced as the first standard-codec-in-the-loop (SCL) training framework for plane-factorized radiance fields [S1]. It overturns the prevailing assumption that the codec must be differentiable, instead treating the real codec round trip as a black box and using STE to pass gradients [S1]. It differs from Learned-Codec-in-the-Loop (LCL) methods like NeRFCodec and CNC, which require learned neural codecs and entropy models, and from Codec-Agnostic (CA) methods that compress features after training [S1].

## Sources
[S1] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
