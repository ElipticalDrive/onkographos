# Spatiotemporal Kernel Prediction Network for VPT

Also called: Dual-stream CNN denoiser, MSKPL denoiser

## Overview
A deep learning denoiser for Volume Path Tracing (VPT) image sequences that uses a dual-stream encoder to process G-buffers and radiance separately, and a Multi-Head Spatiotemporal Kernel Prediction Layer (MSKPL) to predict per-pixel temporal and spatial filter kernels for adaptive denoising [S1, unofficial]. It builds on [Denoising via Neural Auxiliary Features](10536-denoising-via-neural-auxiliary-features.md) by explicitly decoupling the encoding of low-noise auxiliary features from high-noise radiance to reduce color bias and improve learning efficiency [S1, unofficial]. Unlike unified encoders, this separation leverages the fact that G-buffers are color-independent and contain clearer edge/detail information, while the network predicts adaptive kernels rather than direct pixel values [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tuple $X = (\text{G-buffers}, \text{Radiance})$.
    *   **G-buffers:** 4-channel buffer containing Texture, Illumination, Alpha, and World Position [S1, unofficial].
    *   **Radiance:** The noisy current frame radiance image [S1, unofficial].
    *   **Historical Data:** The denoised output of the previous frame $x_{n-1}'$ and its associated motion vector $v$ [S1, unofficial].
*   **Output:** The denoised volume data $x_n''$ blended with the background [S1, unofficial].

**Data Structures:**
*   **Encoder Block:** Consists of two $3 \times 3$ dilated convolution layers, LeakyReLU activation layers, and max-pooling layers [S1, unofficial].
*   **Cascade Fusion Module (CFM):** Consists of convolutional layers, batch normalization layers, and pixel-wise convolutional layers [S1, unofficial].
*   **Decoder Block:** Uses upsampling and concatenation with encoder outputs [S1, unofficial].
*   **Kernel Prediction:** Predicts two kernels per pixel: temporal kernel $K_h$ and spatial kernel $K_c$ [S1, unofficial].

**Parameters:**
*   $L$: Number of blocks in encoder and decoder. Set to $L=3$ [S1, unofficial].
*   Opacity threshold for reprojection: $0.8$. If no sample meets this, the sample with maximum opacity is selected [S1, unofficial].

**Algorithm Steps:**

1.  **Training Optimization:**
    The network parameters $\theta$ are optimized by minimizing the loss function over $N$ data pairs:
    $$ \hat{\theta} = \min \frac{1}{N} \sum_{i=1}^{N} l(X_i; \theta) $$
    where $l$ is the loss function [S1, unofficial].

2.  **Temporal Reprojection:**
    Compute the reprojection of the historical denoised frame $x_{n-1}'$ onto the current frame $n$ using the motion vector $v$. Let $x_{np}$ be the reprojected historical frame:
    $$ x_{np}(i,j) = x_{n-1}'(i+v, j+v) $$
    *Note: The source formula $x_{n-1}'(i+v, j+v, j)$ contains a typo; the third index is interpreted as a spatial coordinate error, corrected to standard 2D spatial indexing $(i,j)$ [S1, unofficial].*
    The reprojection uses the nearest sample point along each viewing ray satisfying the opacity threshold $\alpha > 0.8$. If no sample meets this, the sample with maximum opacity is selected [S1, unofficial].

3.  **Dual-Stream Parallel Encoding:**
    The network processes G-buffers and Radiance through two parallel encoders, $E_b$ (for G-buffers) and $E_a$ (for Radiance). Let $h_i$ be the output of the $i$-th layer in $E_b$ and $x_i$ be the output of the $i$-th layer in $E_a$.
    *   **G-buffer Stream ($E_b$):**
        $$ h_i = \text{LeakyReLU}(E_b(h_{i-1})) $$
        For $i=1$, $h_0$ is the input G-buffer [S1, unofficial].
    *   **Radiance Stream ($E_a$):**
        $$ x_i = \text{LeakyReLU}(x_{i-1}, \text{maxpool}(h_i)) $$
        Here, $x_{i-1}$ is the previous layer's radiance output, and $\text{maxpool}(h_i)$ is the max-pooled output from the G-buffer stream. For $i=1$, $x_0$ is the input Radiance [S1, unofficial].
    *   **Structure:** Each encoder block uses two $3 \times 3$ dilated convolutions [S1, unofficial].

4.  **Cascade Fusion:**
    Fuse the feature maps from the dual-stream encoder using the Cascade Fusion Module (CFM). The CFM standardizes the distribution of the dual-channel feature maps and integrates information using convolutional layers, batch normalization, and pixel-wise convolutions [S1, unofficial].

5.  **Kernel Prediction (MSKPL):**
    Input the reprojected historical features $x_{n-1}'$, current frame radiance $x_n$, and G-buffers into the network $S$ to generate the temporal kernel $K_h$ and spatial kernel $K_c$:
    $$ K_h, K_c = S(x_{n-1}', x_n, \text{gbuffers}) $$
    The output layer uses a Softmax activation function. To prevent kernel polarization (where values approach 0 or 1, preventing effective weight learning), Dropout regularization is applied after the Softmax layer [S1, unofficial].

6.  **Spatial-Temporal Filtering:**
    Apply the predicted kernels to the reprojected historical frame and the current frame to obtain the denoised volume data $x_n'$:
    $$ x_n'(i,j) = K_h(i,j) * x_{np}(i,j) + K_c(i,j) * x_n(i,j) $$
    where $*$ denotes convolution [S1, unofficial].

7.  **Background Blending:**
    Blend the denoised volume data $x_n'$ with the background $B$ using the alpha map $A$:
    $$ x_n''(i,j) = A(i,j) \times x_n'(i,j) + (1 - A(i,j)) \times B(i,j) $$
    This step is performed last because volume data contains numerous semi-transparent structures [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatiotemporal_kernel_prediction_denoiser(
    current_radiance: Tensor,
    current_gbuffers: Tensor,
    prev_denoised: Tensor,
    motion_vectors: Tensor,
    alpha_map: Tensor,
    background: Tensor,
    opacity_threshold: float = 0.8
) -> Tensor:
    """
    Denoises a VPT frame using spatiotemporal kernel prediction.
    
    Args:
        current_radiance: Noisy radiance image of current frame n.
        current_gbuffers: G-buffers (Texture, Illumination, Alpha, World Pos) for frame n.
        prev_denoised: Denoised radiance image of previous frame n-1.
        motion_vectors: Motion vectors v for reprojecting prev_denoised.
        alpha_map: Alpha map for current frame.
        background: Background image B.
        
    Returns:
        Denoised and background-blended image x_n''.
    """
    
    # Step 1: Temporal Reprojection
    # Reproject previous denoised frame using motion vectors
    # Handle opacity threshold: select nearest sample with alpha > 0.8, 
    # else max opacity sample.
    x_np = reproject_historical(prev_denoised, motion_vectors, opacity_threshold)
    
    # Step 2: Dual-Stream Encoding
    # Encoder Eb for G-buffers
    # Encoder Ea for Radiance
    # L = 3 blocks
    h = current_gbuffers
    x = current_radiance
    
    for i in range(1, 4): # L=3
        # G-buffer stream
        h = leaky_relu(conv_dilated_3x3(h))
        
        # Radiance stream: uses previous radiance output and max-pooled G-buffer output
        h_pooled = max_pool(h)
        x = leaky_relu(concat([x, h_pooled]))
        x = conv_dilated_3x3(x)
        
    # Step 3: Cascade Fusion
    # Fuse h and x using CFM (Conv, BatchNorm, Pixel-wise Conv)
    fused_features = cascade_fusion_module(h, x)
    
    # Step 4: Kernel Prediction
    # Input: reprojected historical features, current radiance, gbuffers
    # Output: Temporal kernel Kh, Spatial kernel Kc
    # Note: The source implies the network S takes x_np, x_n, and gbuffers.
    # However, the encoding step above processed x_n and gbuffers. 
    # Assuming S is the full network or a head on top of fused features + history.
    # Based on formula (6): Kh, Kc = S(x_{n-1}', x_n, gbuffers)
    # We pass the necessary inputs to the kernel prediction head.
    Kh, Kc = predict_kernels(x_np, current_radiance, current_gbuffers, fused_features)
    
    # Apply Dropout after Softmax to prevent polarization
    Kh = apply_dropout(Kh)
    Kc = apply_dropout(Kc)
    
    # Step 5: Spatial-Temporal Filtering
    # Convolve kernels with respective frames
    # Kh and Kc are per-pixel kernels, so * denotes convolution operation
    # In implementation, this is likely a weighted sum or small kernel convolution
    x_n_prime = convolve(Kh, x_np) + convolve(Kc, current_radiance)
    
    # Step 6: Background Blending
    x_n_double_prime = alpha_map * x_n_prime + (1 - alpha_map) * background
    
    return x_n_double_prime

def reproject_historical(prev_frame, motion_vectors, threshold):
    """
    Reprojects previous frame using motion vectors.
    Uses opacity threshold to select valid samples.
    """
    # Implementation detail: For each pixel (i,j), use motion vector v to sample
    # from prev_frame. If the corresponding ray sample has alpha > threshold, use it.
    # Else, find sample with max alpha.
    # This requires access to the ray-marching data or pre-computed reprojection buffers.
    # For the purpose of this algorithm description, we assume x_np is computed.
    pass

def predict_kernels(x_np, x_n, gbuffers, fused_features):
    """
    Predicts temporal and spatial kernels.
    Uses Softmax activation.
    """
    # Network S takes x_np, x_n, gbuffers.
    # In the context of the dual-stream encoder, fused_features represents the 
    # processed radiance/gbuffer info.
    # The source formula (6) lists x_{n-1}', x_n, gbuffers as inputs to S.
    # We assume S is a network that processes these inputs to output Kh, Kc.
    
    # Placeholder for the actual network forward pass defined in the paper
    # Kh, Kc = S(x_np, x_n, gbuffers)
    
    # Since we don't have the exact S architecture beyond "MSKPL", 
    # we describe the operation:
    # S outputs two tensors Kh and Kc of same spatial dimensions as input.
    # Values are normalized via Softmax.
    
    # Return dummy tensors for structure compliance
    Kh = torch.zeros_like(x_n)
    Kc = torch.zeros_like(x_n)
    return Kh, Kc
```

## Subtopics

### [Weighted Recursive Least Squares (wRLS)](13229-weighted-recursive-least-squares-wrls.md)
A non-learning, linear-algebraic alternative that uses a temporally stable feature from an exponentially weighted history buffer and variance-based weighting to reduce spike noise while maintaining temporal stability.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method is designed for real-time cinematic rendering in AR/VR [S1, unofficial].
*   **Strengths:**
    *   Separates G-buffer and radiance encoding to reduce color bias and improve learning efficiency [S1, unofficial].
    *   G-buffers provide low-noise guidance for edge and detail preservation [S1, unofficial].
    *   Adaptive kernel prediction allows for handling heterogeneous noise [S1, unofficial].
*   **Limitations:**
    *   Kernel polarization can occur with Softmax, requiring Dropout regularization [S1, unofficial].
    *   Historical frames become unusable during rapid camera motion, abrupt illumination changes, or quick transfer function variations [S1, unofficial].
    *   In volume rendering, rays passing entirely through transparent regions may cause historical frame failure [S1, unofficial].
*   **When to use:** Real-time VPT denoising where temporal stability and detail preservation are critical, and hardware supports real-time inference [S1, unofficial].
*   **When not to use:** Scenes with rapid, unpredictable changes in illumination or transfer functions, or where rays frequently pass through fully transparent regions [S1, unofficial].

## Background
The method was proposed by Xu et al. [S1, unofficial] to achieve real-time ray-tracing volume-rendering denoising for VPT image sequences, targeting applications in AR/VR [S1, unofficial]. It addresses the limitations of unified encoders by decoupling the processing of G-buffers and radiance [S1, unofficial].

## Sources
[S1] Real-Time Volume-Rendering Image Denoising Based on ... - NIH, https://pmc.ncbi.nlm.nih.gov/articles/PMC12028196/, fetched 2026-10-26 (unofficial)
