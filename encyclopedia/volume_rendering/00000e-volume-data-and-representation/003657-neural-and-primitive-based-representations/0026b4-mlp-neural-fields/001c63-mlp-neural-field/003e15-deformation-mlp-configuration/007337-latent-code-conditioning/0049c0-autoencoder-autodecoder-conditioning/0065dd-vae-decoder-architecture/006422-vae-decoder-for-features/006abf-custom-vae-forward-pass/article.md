# Custom VAE Forward Pass

Also called: Modified VAE encoder and decoder with skip connections

## Overview
This method defines a modified forward pass for a Variational Autoencoder (VAE) decoder architecture, specifically adapted from the latent diffusion model (LDM) framework to support feature interpretation in neural fields [S1, unofficial]. It distinguishes itself from standard VAE implementations by introducing learnable gamma scaling factors on skip connections and explicitly managing intermediate encoder outputs to facilitate cross-block feature fusion in the decoder [S1, unofficial]. This approach addresses the need for high-fidelity reconstruction of complex feature maps by preserving spatial details that are typically lost in deep bottleneck architectures.

## How it works
**Inputs and Outputs:**
- **Encoder Input:** A latent sample tensor.
- **Encoder Output:** The processed latent tensor and a list of intermediate feature maps stored in `self.current_down_blocks`.
- **Decoder Input:** The latent tensor, a list of incoming skip activations `self.incoming_skip_acts` (matching encoder down-block outputs), and optional latent embeddings `latent_embeds`.
- **Decoder Output:** The reconstructed feature tensor.

**Data Structures:**
- `self.current_down_blocks`: A list of tensors containing the output of each down-block in the encoder.
- `self.incoming_skip_acts`: A list of tensors provided to the decoder, expected to match the spatial dimensions and channel counts of the encoder's down-block outputs.
- `self.skip_convs`: A list of convolutional layers, one for each up-block, used to align channel dimensions of skip connections.
- `gamma`: A learnable scalar parameter (float) applied to scale skip connection features.
- `ignore_skip`: A boolean flag to disable skip connections entirely.

**Numbered Steps:**

1.  **Encoder Forward Pass:**
    1.  Pass the input sample through the initial convolution `conv_in`.
    2.  Iterate through the `down_blocks`. For each block:
        -   Pass the current feature map through the block.
        -   Append the output tensor to a temporary list `l_blocks`.
    3.  Pass the final down-block output through the `mid_block`.
    4.  Apply normalization `conv_norm_out`, activation `conv_act`, and final convolution `conv_out`.
    5.  Store `l_blocks` into `self.current_down_blocks` for later retrieval by the decoder.
    6.  Return the processed sample.

2.  **Decoder Forward Pass:**
    1.  Pass the input latent sample through `conv_in`.
    2.  Pass the result through the `mid_block`, injecting `latent_embeds` if provided.
    3.  Cast the tensor to `upscale_dtype` (typically lower precision for memory efficiency during upsampling).
    4.  If `ignore_skip` is false, iterate through the `up_blocks` in order (index `idx` from 0 to $N-1$):
        -   Retrieve the corresponding skip activation from the reversed list: `skip_act = self.incoming_skip_acts[::-1][idx]`.
        -   Apply the skip convolution: `skip_aligned = self.skip_convs[idx](skip_act)`.
        -   Scale by the learnable parameter: `skip_scaled = skip_aligned * self.gamma`.
        -   Add the scaled skip feature to the current decoder feature map.
        -   Pass the combined feature map through the `up_block[idx]`.
    5.  If `ignore_skip` is true, simply pass the feature map through each `up_block` sequentially without adding skip connections.

3.  **Decoder Post-Processing:**
    1.  Apply normalization `conv_norm_out`. If `latent_embeds` is not None, pass it as a condition; otherwise, apply bias-only normalization.
    2.  Apply activation `conv_act`.
    3.  Apply final convolution `conv_out`.
    4.  Return the final sample.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class CustomVAEForwardPass:
    def __init__(self, encoder, decoder, gamma_init=1.0, ignore_skip=False):
        self.encoder = encoder
        self.decoder = decoder
        # Learnable gamma for skip connections
        self.gamma = nn.Parameter(torch.tensor(gamma_init))
        self.ignore_skip = ignore_skip
        # Placeholder for skip convs and blocks, assumed present in decoder structure
        # In the source, these are part of the decoder architecture itself.
        
    def encoder_forward(self, sample: torch.Tensor) -> torch.Tensor:
        """
        Encoder forward pass storing intermediate features.
        """
        # 1. Initial convolution
        x = self.encoder.conv_in(sample)
        
        # 2. Down blocks with storage
        l_blocks = []
        for down_block in self.encoder.down_blocks:
            x = down_block(x)
            l_blocks.append(x)
            
        # 3. Mid block
        x = self.encoder.mid_block(x)
        
        # 4. Output projection
        x = self.encoder.conv_norm_out(x)
        x = self.encoder.conv_act(x)
        x = self.encoder.conv_out(x)
        
        # 5. Store for decoder
        self.current_down_blocks = l_blocks
        
        return x

    def decoder_forward(self, sample: torch.Tensor, 
                        incoming_skip_acts: list, 
                        latent_embeds: torch.Tensor = None) -> torch.Tensor:
        """
        Decoder forward pass with learnable skip connections.
        """
        # 1. Initial convolution
        x = self.decoder.conv_in(sample)
        
        # 2. Mid block with embeddings
        x = self.decoder.mid_block(x, latent_embeds)
        
        # 3. Cast to upscale dtype
        upscale_dtype = self.decoder.upscale_dtype
        x = x.to(upscale_dtype)
        
        # 4. Up blocks with skip connections
        if not self.ignore_skip:
            # Reverse the skip acts to match upsampling order
            reversed_skips = incoming_skip_acts[::-1]
            
            for idx, up_block in enumerate(self.decoder.up_blocks):
                # Get skip connection
                skip_act = reversed_skips[idx]
                
                # Apply skip convolution and gamma scaling
                # Note: self.decoder.skip_convs[idx] is assumed to exist
                skip_aligned = self.decoder.skip_convs[idx](skip_act)
                skip_scaled = skip_aligned * self.gamma
                
                # Add skip connection
                x = x + skip_scaled
                
                # Pass through up block
                x = up_block(x)
        else:
            for up_block in self.decoder.up_blocks:
                x = up_block(x)
                
        # 5. Post-processing
        # Apply norm with or without embeddings
        if latent_embeds is not None:
            x = self.decoder.conv_norm_out(x, latent_embeds)
        else:
            x = self.decoder.conv_norm_out(x)
            
        x = self.decoder.conv_act(x)
        x = self.decoder.conv_out(x)
        
        return x
```

**Parameters:**
- `gamma`: Float, initialized to 1.0. It scales the magnitude of skip connection features. [S1, unofficial]
- `ignore_skip`: Boolean. If True, skip connections are bypassed. [S1, unofficial]
- `upscale_dtype`: Data type (e.g., `torch.float16`) used for intermediate computations in the decoder to save memory. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Ordering:** Skip connections must be reversed (`[::-1]`) because the encoder stores them in down-sampling order (coarse to fine), while the decoder processes them in up-sampling order (fine to coarse). Failure to reverse results in dimension mismatch or semantic misalignment. [S1, unofficial]
- **Embedding Scope:** Latent embeddings are only injected into the `mid_block` and the final `conv_norm_out`. They are not passed to the `up_blocks`. This design choice isolates global conditioning from local upsampling operations. [S1, unofficial]
- **Precision:** Casting to `upscale_dtype` occurs before upsampling. If the dtype is too low precision (e.g., FP16 without proper handling), gradient instability may occur during training. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of up/down blocks. The addition of skip connections and gamma scaling adds negligible computational overhead compared to the convolutional operations.
- **Memory Complexity:** The encoder must store all intermediate down-block outputs in `self.current_down_blocks`, increasing peak memory usage proportional to the number of blocks and feature map sizes. This is a trade-off for improved reconstruction quality.
- **Strengths:** The learnable `gamma` allows the model to adaptively weight the importance of skip connections, which is crucial for preserving high-frequency details in feature maps. The explicit separation of embedding injection points provides stable conditioning.
- **Limitations:** The requirement to store all encoder outputs increases VRAM consumption, potentially limiting batch sizes or input resolutions on hardware with limited memory.
- **When to use:** Use this method when high-fidelity feature reconstruction is required, such as in neural field representations where spatial detail is critical. It is particularly effective when combined with LoRA or other parameter-efficient fine-tuning methods, as indicated by the source's focus on saving LoRA states. [S1, unofficial]
- **When not to use:** Avoid if memory is strictly constrained and approximate reconstruction is acceptable. Standard VAE decoders without skip connections are more memory-efficient.

## Background
This method originates from the Difix3D project by NVIDIA-Tlabs, which adapts the VAE architecture from Latent Diffusion Models (LDM) for 3D generation tasks [S1, unofficial]. The modification introduces learnable skip connection scaling to enhance the decoder's ability to interpret features generated by neural fields. It builds upon the standard VAE Decoder for Features architecture, which uses a purely convolutional structure with residual blocks. The approach reflects a trend in neural rendering and diffusion models to refine reconstruction quality through explicit feature fusion mechanisms.

## Sources
[S1] nv-tlabs/Difix3D, https://github.com/nv-tlabs/Difix3D, fetched 2026-10-26 (unofficial)
