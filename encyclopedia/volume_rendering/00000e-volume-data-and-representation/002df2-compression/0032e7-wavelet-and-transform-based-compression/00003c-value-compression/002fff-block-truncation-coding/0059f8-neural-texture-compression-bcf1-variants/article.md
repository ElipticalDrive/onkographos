# Neural Texture Compression (BCF1 Variants)

Also called: HW NBTC, Neural Block Texture Compression, Neural Block Texture Compression with Cooperative Vectors

## Overview
Neural Texture Compression (BCF1) is a method for compressing physically-based rendering (PBR) texture sets by storing latent features in BC1 format and decoding them using a Multi-Layer Perceptron (MLP) accelerated by hardware cooperative vectors [S1]. It solves the storage and memory bandwidth bottleneck of high-resolution PBR assets by leveraging existing block compression hardware for random access while using neural inference to reconstruct high-dimensional material properties [S1]. Unlike its sibling [Block Truncation Coding](6187-block-truncation-coding.md) which supports general hardware accelerated random access, BCF1 specifically targets PBR texture sets and uses low dynamic range (LDR) BC1 compression instead of high dynamic range (HDR) BC6, allowing for higher compression ratios or improved quality at the same footprint [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of PBR texture maps (albedo, normal, roughness, metalness, etc.) at resolution $W \times H$.
*   Network architecture parameters: Number of latent textures $K$, MLP hidden dimension $D$, number of layers $L$.
*   Training hyperparameters: Learning rates for MLP ($10^{-3}$) and BC1 textures ($10^{-2}$) [S1].

**Outputs:**
*   A set of $K$ BC1 compressed latent textures $T_0, \dots, T_{K-1}$ with varying resolutions.
*   The weights of the Multi-Layer Perceptron (MLP).

### Data Structures

**BC1 Latent Texture Layout:**
Each latent texture $T_k$ is stored in standard BC1 format [S1].
*   **Block Size:** $4 \times 4$ texels.
*   **Shared Data:** Two RGB endpoints $e_0, e_1$ per block.
    *   $e_0$: 5 bits Red, 6 bits Green, 5 bits Blue.
    *   $e_1$: 5 bits Red, 6 bits Green, 5 bits Blue.
*   **Per-Texel Data:** Blending factor $\alpha$ per texel.
    *   $\alpha$: 2 bits.
*   **Decoding Formula:** For a texel at local coordinates $(u, v)$ within a block:
    $$ T(uv) = (1 - \alpha(uv)) e_0 + \alpha(uv) e_1 $$
    where $\alpha(uv)$ is interpreted as a normalized value derived from the 2-bit integer [S1].

**MLP Structure:**
*   Input vector $x$: Concatenation of latent colors from all $K$ textures evaluated at UV coordinate $p(uv)$.
*   Hidden layers: Linear transformations followed by activation functions (typically ReLU, though not explicitly specified, standard MLP practice applies [G]).
*   Output vector $f$: The reconstructed PBR feature vector (e.g., albedo, normal, roughness).

**Resolution Variants:**
Two variants are defined based on the resolution of the $K$ latent textures relative to the original $W \times H$ resolution [S1]:
*   **VAR A:** Latent textures at resolutions $2 \times 2$, $4 \times 4$, $8 \times 8$, and $W \times H$. Designed to have a compression ratio similar to BCF6.
*   **VAR B:** Latent textures at resolutions $2 \times 2$, $4 \times 4$, $8 \times 8$, and $W \times H$. Designed to double the compression ratio of VAR A (implied by "almost double this compression ratio" [S1]). Note: The source lists the same resolutions for both, but implies different scaling/counts or usage strategies to achieve the ratio difference [S1].

### Training Procedure

1.  **Replicate BC1 Unit:** Implement a differentiable BC1 texture sampling unit in the training framework (e.g., PyTorch) [S1].
2.  **Initialize Parameters:**
    *   Initialize MLP weights.
    *   Initialize BC1 texture endpoints and alphas as floating-point tensors [S1].
3.  **Forward Pass (Sampling):**
    *   For a given training sample (UV coordinate $p(uv)$ and LOD level):
        *   Sample each latent texture $T_k$ at $p(uv)$ to get latent color $c_k$.
        *   Apply sub-pixel shifts: Shift the sampling coordinate by half a texel for the second and fourth latent textures ($k=1, 3$) to avoid alignment artifacts [S1].
        *   Concatenate latent colors: $x = [c_0, \dots, c_{K-1}]$.
        *   Feed $x$ to the MLP to obtain feature vector $f = \text{MLP}(x)$ [S1].
4.  **Loss Calculation:**
    *   Compute L1 loss between the decoded feature vector $f$ and the reference PBR features sampled using trilinear filtering [S1].
5.  **Backpropagation and Quantization-Aware Training:**
    *   Gradients are passed through the MLP and the BC1 textures.
    *   To simulate hardware quantization during training, apply sigmoid activation followed by quantization-aware operations [S1]:
        $$ \bar{\alpha} = \text{quant}(\text{sigmoid}(\alpha), [2]) $$
        $$ \bar{e}_0 = \text{quant}(\text{sigmoid}(e_0), [5, 6, 5]) $$
        $$ \bar{e}_1 = \text{quant}(\text{sigmoid}(e_1), [5, 6, 5]) $$
    *   The `quant` function returns the quantized integer value but allows gradients to pass through the floating-point input [S1].
    *   Update BC1 texture parameters using Adam optimizer with learning rate $10^{-2}$ [S1].
    *   Update MLP weights using Adam optimizer with learning rate $10^{-3}$ [S1].
6.  **LOD Sampling:** During training, sample a LOD level for each texture and regress the associated mip level, similar to Vaidyanathan et al. [S1].

### Runtime Inference

1.  **Visibility Pass:** Determine which pixels are visible.
2.  **Classification Pass:** Sort pixels/tiles requiring neural inference. Tiles are classified into types (e.g., no neural, simple neural) to optimize processing [S1].
3.  **G-Buffer Pass:**
    *   For each pixel (or tile) requiring neural decoding:
        *   Fetch latent features from the BC1 textures using hardware texture filtering.
        *   Concatenate fetched latent colors $x$.
        *   Evaluate the MLP to produce feature vector $f$.
        *   Accelerate MLP evaluation using hardware cooperative vectors (matrix multiplication engines) [S1].
        *   Store $f$ in the G-Buffer.
4.  **Lighting Pass:** Use the G-Buffer data to shade the pixel using a shading model (e.g., diffuse + GGX + IBL) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class BC1Quantization(torch.autograd.Function):
    """
    Differentiable BC1 quantization simulation.
    Applies sigmoid to float values, quantizes to integer bits,
    but passes gradients through the float values.
    """
    @staticmethod
    def forward(ctx, x, bits):
        # Quantize x to 'bits' bits
        # In a real implementation, this would map to 0..2^bits-1
        # For gradient passing, we keep x as is or use straight-through estimator
        # The source specifies: quant(sigmoid(x), bits)
        # We assume 'quant' here returns the integer representation
        # but for backward, we return the float input.
        # Note: The source says "let the gradients of its floating point value pass through"
        # This implies a Straight-Through Estimator (STE) behavior.
        quantized = torch.round(x * (2**bits - 1)).clamp(0, 2**bits - 1)
        return quantized

    @staticmethod
    def backward(ctx, grad_output):
        # Pass gradient directly to the input (STE)
        return grad_output, None

def sigmoid_quantize(x, bits):
    return BC1Quantization.apply(torch.sigmoid(x), bits)

class BCF1Decoder(nn.Module):
    def __init__(self, num_latents, hidden_dim, num_layers):
        super(BCF1Decoder, self).__init__()
        self.num_latents = num_latents
        # MLP Architecture
        layers = []
        input_dim = num_latents * 3 # Assuming RGB latent colors
        for i in range(num_layers):
            layers.append(nn.Linear(input_dim if i == 0 else hidden_dim, hidden_dim))
            layers.append(nn.ReLU()) # Standard activation, not specified but implied by MLP
        layers.append(nn.Linear(hidden_dim, 9)) # Example: 9 PBR channels
        self.mlp = nn.Sequential(*layers)
        
        # BC1 Parameters (Initialized during training)
        # These are stored as BC1 textures in practice, but for simulation:
        self.endpoints_0 = nn.Parameter(torch.randn(num_latents, 3)) # Placeholder for texture data
        self.endpoints_1 = nn.Parameter(torch.randn(num_latents, 3))
        self.alphas = nn.Parameter(torch.randn(num_latents, 1)) # Placeholder for alpha data

    def decode_bc1(self, uv_coords, latent_idx):
        """
        Simulate BC1 decoding for a specific latent texture at UV coords.
        In reality, this is a hardware texture fetch.
        """
        # Retrieve endpoints and alpha for the specific latent and UV
        # e0, e1 are [5,6,5] quantized, alpha is [2] bit
        # For simulation, we assume these are already quantized integers
        # and we convert them to float for the linear interpolation.
        
        # Note: In the actual method, this is a hardware fetch.
        # Here we simulate the interpolation formula:
        # T(uv) = (1 - alpha) * e0 + alpha * e1
        
        # Get e0, e1, alpha for this latent at this UV
        # In a real implementation, these are fetched from BC1 texture memory
        e0 = self.get_endpoint(latent_idx, uv_coords, 0)
        e1 = self.get_endpoint(latent_idx, uv_coords, 1)
        alpha = self.get_alpha(latent_idx, uv_coords)
        
        # Interpolate
        color = (1 - alpha) * e0 + alpha * e1
        return color

    def get_endpoint(self, latent_idx, uv, endpoint_idx):
        # Placeholder for hardware fetch logic
        # Returns the [5,6,5] quantized endpoint as float
        pass

    def get_alpha(self, latent_idx, uv):
        # Placeholder for hardware fetch logic
        # Returns the [2] bit quantized alpha as float
        pass

    def forward(self, uv_coords):
        """
        Evaluate the neural texture at UV coordinates.
        """
        latent_colors = []
        for k in range(self.num_latents):
            # Apply sub-pixel shift for k=1 and k=3 (second and fourth textures)
            uv = uv_coords.clone()
            if k in [1, 3]:
                uv += 0.5 # Shift by half a texel
            
            # Fetch latent color from BC1 texture
            c_k = self.decode_bc1(uv, k)
            latent_colors.append(c_k)
        
        # Concatenate latent colors
        x = torch.cat(latent_colors, dim=-1)
        
        # Feed to MLP
        f = self.mlp(x)
        return f

def train_step(model, uv_coords, reference_features):
    """
    Single training step.
    """
    # Forward pass
    predicted_features = model(uv_coords)
    
    # L1 Loss
    loss = torch.nn.functional.l1_loss(predicted_features, reference_features)
    
    # Backward pass
    loss.backward()
    
    # Update parameters
    # Note: In practice, separate optimizers for MLP and BC1 textures
    # optimizer_mlp.step()
    # optimizer_bc1.step()
    
    return loss.item()
```

### Edge Cases and Pitfalls
*   **Block Artifacts:** Doubling the compression ratio (VAR B vs VAR A) increases block artifacts from BC1 compression, reducing quality [S1].
*   **Alignment Artifacts:** Latent blocks aligning with texel boundaries can cause artifacts. Mitigated by shifting the second and fourth latent textures by half a texel [S1].
*   **Precision:** BC1 uses low precision (2-bit alpha, 5-6-5 endpoints). Training with quantization-aware techniques is necessary to maintain quality [S1].
*   **Hardware Dependency:** Runtime performance relies on hardware cooperative vectors extension. Without it, performance degrades significantly (13 ms vs 0.55 ms on Intel B580) [S1].

## Operating characteristics

**Time and Memory Complexity:**
*   **Memory:** The method stores $K$ BC1 textures. VAR A has a compression ratio similar to BCF6. VAR B has approximately double the compression ratio of VAR A [S1].
*   **Runtime:** The bottleneck is MLP evaluation. Hardware acceleration via cooperative vectors reduces inference time from 13 ms (software FMA) to 0.55 ms on Intel B580 for 4k texture sets at 1080p [S1].

**Measured Performance:**
*   **Hardware:** Intel B580, Nvidia RTX 4090 [S1].
*   **Scene:** 4 assets (robot, bee, bee hive, rocks) with 4096x4096 PBR textures [S1].
*   **Resolution:** 1920x1080 screen resolution [S1].
*   **Performance:** 0.55 ms rendering time for 4k texture sets (9 channels) at 1080p with anisotropic filtering on Intel B580, using 28MB VRAM per texture set [S1].
*   **Speedup:** Up to 23x speedup compared to software FMA implementation [S1].

**Quality:**
*   **PSNR (D=64):**
    *   VAR A1: min 23.34, max 40.91, avg 50.02 [S1].
    *   VAR B1: min 22.91, max 39.02, avg 49.18 [S1].
*   **Comparison:** VAR A produces similar quality to Ubisoft's BCF6 with a small gain in compression [S1]. BCF1 variants are compared against Nvidia's NTC (1.0 and 0.5 bits per channel per pixel variants) and Ubisoft's BCF6 [S1].

**Strengths and Limitations:**
*   **Strengths:** High compression ratio, hardware-accelerated decoding, anisotropic filtering works without specific training, low VRAM usage (28MB for 4k sets) [S1].
*   **Limitations:** Block artifacts at high compression ratios, dependency on hardware cooperative vectors for optimal performance, LDR limitation (though deemed sufficient) [S1].

**When to Use:**
*   Use when VRAM is constrained and high-resolution PBR textures are needed.
*   Use when hardware supports cooperative vectors for accelerated MLP inference.
*   Use when anisotropic filtering is required without additional training overhead [S1].

**When Not to Use:**
*   Use when HDR latent values are strictly required (use BCF6 instead) [S1].
*   Use when hardware does not support cooperative vectors and software fallback performance is unacceptable [S1].

## Background

**Origin:**
The method was introduced by L. Belcour and A. Benyoub from Intel Labs, France, in the paper "Hardware Accelerated Neural Block Texture Compression with Cooperative Vectors" presented at High-Performance Graphics (HPG) 2025 [S1]. It is an extension of the neural texture compression method by Weinreich and colleagues (BCF6) [S1].

**History and Development:**
The work builds on previous neural texture compression methods that used BC6 for HDR storage. The authors demonstrated that LDR BC1 compression is viable for PBR textures, allowing for higher compression ratios or improved quality. They introduced hardware acceleration using cooperative vectors to overcome the performance bottleneck of MLP inference [S1].

**Usage:**
The method is designed for modern games and rendering applications that rely on high-resolution PBR textures. It is integrated into a tile-based rendering pipeline with visibility, classification, G-Buffer, and lighting passes [S1].

## Sources

[S1] Hardware Accelerated Neural Block Texture Compression with Cooperative Vectors, https://doi.org/10.2312/hpg20251173, fetched 2026-11-06
