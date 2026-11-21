# Codec-Adaptive TriPlane Radiance Fields

Also called: CATRF, SCL, Standard-Codec-in-the-Loop

## Overview
Codec-Adaptive TriPlane Radiance Fields (CATRF) is a compression framework for plane-factorized radiance fields that optimizes neural features directly in the decoded space of a standard video or image codec [S1]. It solves the problem of severe rendering degradation at low bitrates by integrating a real codec round-trip (JPEG, VP9, HEVC, or AV1) into the training loop via a Straight-Through Estimator (STE) [S1]. Unlike Codec-Agnostic (CA) methods that compress features after training, or Learned-Codec-in-the-Loop (LCL) methods that require shipping scene-specific neural decoder parameters, CATRF adapts features to real codec distortions without introducing any learned codec parameters or entropy models [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained plane-factorized radiance field backbone (TensoRF for static scenes, DVGO for dynamic scenes) [S1].
*   **Output:** A compressed bitstream of the radiance field features decodable by standard codecs, and the ability to render the scene from this compressed representation [S1].

**Data Structures:**
The method operates on plane-factorized radiance fields. The scene is represented by three orthogonal feature planes $P = \{P_{xy}, P_{xz}, P_{yz}\}$ and a 3D density grid $D$ [S1].
*   Each feature plane $P_{ax}$ (where $ax \in \{xy, xz, yz\}$) has layout $P_{ax} \in \mathbb{R}^{C \times H \times W}$, where $C$ is the channel dimension, and $H, W$ are height and width [S1].
*   The density grid is $D \in \mathbb{R}^{D_y \times D_x \times D_z}$ [S1].
*   These tensors are quantized and packed into 2D canvases (images with integer pixel values) suitable for standard codecs [S1]. The default packing format is `FlattenGray`, which avoids cross-channel mixing to reduce codec artifact sensitivity [S1].
*   Quantization uses AbsMax quantization by default [S1]. Channelwise quantization is avoided as it increases bitrate by ~45x [S1].

**Parameters:**
*   **Codec:** Off-the-shelf codecs (JPEG, VP9, HEVC, AV1) [S1].
*   **Total-variation regularization weight ($\lambda_{tv}$):** Controls the trade-off between codec compatibility and high-frequency detail preservation. Default value: $5 \times 10^{-5}$ [S1].
*   **Cache refresh interval ($M$):** The number of training steps before the cached decoded planes are refreshed. Default value: 128 [S1].

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
6.  **Backpropagation:** Pass gradients across the entire codec round-trip using a Straight-Through Estimator (STE). The codec is treated as a black box with an identity gradient (i.e., $\frac{\partial L}{\partial \text{Input}} = \frac{\partial L}{\partial \text{Output}}$) [S1].
7.  **Caching:** Cache the decoded planes across training steps to avoid per-minibatch codec round trips. The cache is refreshed every $M$ steps [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

# Assume standard codec functions are available as black boxes
# codec_encode(image_tensor) -> bitstream
# codec_decode(bitstream) -> image_tensor
# These functions are non-differentiable.

class StraightThroughCodec(nn.Module):
    """
    A module that performs a codec round-trip and passes gradients
    through it using a Straight-Through Estimator (STE).
    """
    def forward(self, input_tensor):
        # Forward pass: encode then decode
        bitstream = codec_encode(input_tensor)
        output_tensor = codec_decode(bitstream)
        # STE: Gradient flows as if output == input
        return output_tensor

class CATRF:
    def __init__(self, backbone, codec_config, lambda_tv=5e-5, cache_refresh_interval=128):
        self.backbone = backbone # TensoRF or DVGO
        self.codec = StraightThroughCodec()
        self.lambda_tv = lambda_tv
        self.cache_refresh_interval = cache_refresh_interval
        self.step_count = 0
        self.cached_decoded_planes = None
        self.cached_bitstream = None
        
        # Packing and Quantization helpers
        self.packing_format = "FlattenGray"
        self.quantization = "AbsMax"

    def quantize_and_pack(self, planes, density_grid):
        """
        Quantize feature planes and density grid into 2D canvases.
        Uses AbsMax quantization and FlattenGray packing.
        """
        # FlattenGray: Flatten channels and spatial dims into 2D image-like structure
        # AbsMax: Scale each plane by its absolute maximum value to [0, 255] range (uint8)
        canvases = []
        for plane in planes:
            # plane shape: (C, H, W)
            abs_max = plane.abs().max()
            if abs_max == 0:
                abs_max = 1e-8
            normalized = plane / abs_max
            # Pack into 2D canvas (implementation detail of FlattenGray)
            canvas = pack_flatten_gray(normalized)
            canvases.append(canvas)
        
        # Pack density grid similarly if needed, or handle separately
        # For simplicity, assuming density is packed into a canvas too
        density_canvas = pack_density_grid(density_grid)
        canvases.append(density_canvas)
        
        return canvases

    def unpack_and_dequantize(self, decoded_canvases):
        """
        Unpack decoded canvases back to feature planes and dequantize.
        """
        planes = []
        for i, canvas in enumerate(decoded_canvases[:-1]): # Last one is density
            # Unpack FlattenGray
            normalized = unpack_flatten_gray(canvas)
            # Dequantize: multiply back by abs_max (stored or recomputed)
            # Note: In practice, abs_max needs to be transmitted or stored.
            # For the purpose of the loop, we assume the codec preserves the structure
            # and we recover the scale factor.
            # [G] In a real implementation, the quantization scale factors must be
            # stored in the bitstream or derived from the decoded data.
            # Here we assume a simplified recovery.
            plane = normalized * self.abs_max_cache[i] 
            planes.append(plane)
        
        density_grid = unpack_density_grid(decoded_canvases[-1])
        return planes, density_grid

    def compute_tv_regularization(self, planes):
        """
        Compute spatial total variation regularization.
        """
        tv_loss = 0.0
        for plane in planes:
            # plane shape: (C, H, W)
            # TV = sum(|dx| + |dy|)
            dx = plane[:, :, 1:] - plane[:, :, :-1]
            dy = plane[:, 1:, :] - plane[:, :-1, :]
            tv_loss += dx.abs().mean() + dy.abs().mean()
        return tv_loss

    def train_step(self, rays, gt_images):
        """
        One training step.
        """
        # 1. Get current planes from backbone
        planes, density_grid = self.backbone.get_planes_and_density()
        
        # 2. Check cache
        refresh_cache = False
        if self.cached_decoded_planes is None or self.step_count % self.cache_refresh_interval == 0:
            refresh_cache = True
        
        if refresh_cache:
            # Quantize and Pack
            canvases = self.quantize_and_pack(planes, density_grid)
            
            # Codec Round-Trip
            bitstream = codec_encode(canvases)
            decoded_canvases = codec_decode(bitstream)
            
            # Unpack and Dequantize
            decoded_planes, decoded_density = self.unpack_and_dequantize(decoded_canvases)
            
            # Cache results
            self.cached_decoded_planes = decoded_planes
            self.cached_density = decoded_density
            self.cached_bitstream = bitstream
            
            # Store abs_max for dequantization recovery [G]
            # In practice, this requires storing metadata.
            # For the loop, we assume the backbone can re-quantize or we store scales.
            # Here we assume the decoded planes are already in the correct scale
            # relative to the backbone's expected input range, or we adjust.
            # [G] The paper states "dequantize the reconstructed features", implying
            # the codec operates on quantized values. We must recover the scale.
            # Let's assume we stored the scales during packing.
            self.abs_max_cache = [p.abs().max().item() for p in planes]

        # Use cached decoded planes for rendering
        # Note: The backbone's internal planes are not used for rendering in this step,
        # only for the next packing step.
        planes_to_render = self.cached_decoded_planes
        density_to_render = self.cached_density
        
        # 3. Render
        rendered_images = self.backbone.render(planes_to_render, density_to_render, rays)
        
        # 4. Compute Loss
        reconstruction_loss = mse_loss(rendered_images, gt_images)
        tv_loss = self.compute_tv_regularization(planes_to_render)
        total_loss = reconstruction_loss + self.lambda_tv * tv_loss
        
        # 5. Backpropagate
        total_loss.backward()
        
        self.step_count += 1
        return total_loss.item()

    def optimize(self, dataloader, epochs):
        """
        Two-stage optimization.
        Stage 1: Train backbone to convergence.
        Stage 2: Finetune with SCL.
        """
        # Stage 1
        self.backbone.train_to_convergence(dataloader)
        
        # Stage 2
        for epoch in range(epochs):
            for rays, gt_images in dataloader:
                self.train_step(rays, gt_images)
                self.optimizer.step()
                self.optimizer.zero_grad()
```

**Edge Cases and Pitfalls:**
*   **Codec Sensitivity:** Feature planes are highly sensitive to codec distortions (quantization, block transforms, in-loop filtering). Training without codec exposure (Codec-Agnostic) causes severe rendering degradation at low bitrates [S1].
*   **Bitrate vs. Quality:** Standard codecs are sensitive to artifacts. CATRF adapts to various quality levels, operating at lower bitrates with modest detail loss or higher bitrates for sharper edges [S1].
*   **Quantization Choice:** Channelwise quantization increases bitrate by ~45x and is unsuitable for streaming. FlattenGray with AbsMax quantization is the default [S1].
*   **Cache Refresh:** The cache refresh interval $M$ balances training time and bitrate. $M=128$ is the empirical default [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The memory footprint is dominated by the encoded feature planes. On the NeRF Synthetic dataset, CATRF uses 70.04 MB total, with 69.95 MB for encoded feature planes, 0.05 MB for feature vectors, and 0.04 MB for hash embeddings. It requires 0 MB for neural codec parameters, entropy model parameters, anchor features, and MLPs [S1].
*   **Measured Performance:**
    *   **Static Scenes:** CATRF-JPEG achieves a 43% BD-rate reduction over state-of-the-art LCL compression methods. On NeRF Synthetic, it achieves a 23% bitrate reduction over CNC at matched PSNR. On Tanks and Temples, it achieves a 43% bitrate saving over CNC [S1].
    *   **Dynamic Scenes:** CATRF achieves BD-rate reductions of 40.5% (AV1), 63.3% (HEVC), and 51.8% (VP9) compared to TeTriRF on the N3D dataset. It achieves 17% bitrate savings over HAC++ and 39% BD-rate reduction over GIFStream [S1].
    *   **Decoding Speed:** CATRF-HEVC reconstructs radiance fields at 138 FPS, compared to 2.94 FPS for GIFStream and 0.35 FPS for HAC++. This is because CATRF reconstruction is neural-free, relying only on standard-codec decoding [S1].
*   **Strengths:** No need to transmit scene-specific decoding parameters (unlike LCL). Neural-free reconstruction (unlike 3DGS methods). High decoding throughput.
*   **Limitations:** Requires a two-stage training process. Relies on the quality of the underlying backbone (TensoRF/DVGO).
*   **When to Use:** Volumetric content delivery, free-viewpoint video streaming, 3D video playback, and telepresence [S1].

## Background
CATRF was introduced as the first standard-codec-in-the-loop (SCL) training framework for plane-factorized radiance fields [S1]. It overturns the prevailing assumption that the codec must be differentiable, instead treating the real codec round trip as a black box and using STE to pass gradients [S1]. It differs from Learned-Codec-in-the-Loop (LCL) methods like NeRFCodec and CNC, which require learned neural codecs and entropy models, and from Codec-Agnostic (CA) methods that compress features after training [S1].

## Sources
[S1] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
