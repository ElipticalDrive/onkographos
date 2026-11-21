# CodecSplat

Also called: Ultra-compact latent coding framework for feed-forward 3D Gaussian splatting

## Overview
CodecSplat is a compression framework for feed-forward 3D Gaussian Splatting (FF3DGS) that encodes an intermediate 2D Gaussian-generation feature into an entropy-coded scene bitstream, avoiding the inefficient compression of irregular 3D Gaussian primitives [S1, unofficial]. Unlike existing 3DGS compression methods that compress Gaussian primitives after a structured-to-irregular transformation, CodecSplat places the coding bottleneck before 2D-to-3D Gaussian construction, allowing compression to exploit the structured intermediate feature representation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ posed context views $C = \{(I_i, K_i, T_i, n_i, f_i)\}_{i=1}^N$, where $I_i \in \mathbb{R}^{H \times W \times 3}$ is a context image, $K_i$ and $T_i$ denote camera intrinsics and extrinsics, and $n_i, f_i$ are near and far bounds [S1, unofficial].
*   **Output:** A self-contained scene bitstream $B$ [S1, unofficial].

**Data Structures:**
*   **Multi-view features ($F_{mv}$):** Image and cross-view matching features extracted by the backbone.
*   **Encoder-side depth ($D_{enc}$):** A tensor in $\mathbb{R}^{N \times H \times W}$ representing depth predictions at the encoder side [S1, unofficial].
*   **Depth matching-probability features ($P_{depth}$):** Features containing depth matching probabilities [S1, unofficial].
*   **Gaussian-generation feature ($F_g$):** A dense, regular tensor in $\mathbb{R}^{N \times C_F \times H \times W}$ specialized for Gaussian generation [S1, unofficial].
*   **Latent features:** The compressed representation of $F_g$ after analysis transform, quantization, and entropy coding [S1, unofficial].

**Parameters:**
*   **Rate control parameter ($\lambda$):** Used in the encoder and decoder to control the trade-off between bitstream size and reconstruction quality [S1, unofficial].

**Steps:**

1.  **Multi-view Feature Extraction:** The encoder performs multi-view matching and feature extraction on the context views $C$. This produces multi-view features $F_{mv}$, encoder-side depth $D_{enc}$, and depth matching-probability features $P_{depth}$:
    $$ (F_{mv}, D_{enc}, P_{depth}) = \text{MVEnc}(C) $$
    where $\text{MVEnc}$ denotes the multi-view encoder backbone [S1, unofficial].

2.  **Gaussian-Generation Feature Construction:** The extracted cues are converted into a dense Gaussian-generation feature $F_g$:
    $$ F_g = \text{FeatHead}(I_{1:N}, F_{mv}, P_{depth}) $$
    where $\text{FeatHead}$ is a feature head network [S1, unofficial].

3.  **Latent Encoding (Encoder Side):**
    *   **Analysis Transform:** The feature $F_g$ is transformed into a latent representation.
    *   **Quantization:** The latent representation is quantized using the rate control parameter $\lambda$ [S1, unofficial].
    *   **Entropy Coding:** The quantized latents are entropy-coded.
    *   **Bitstream Packing:** The encoded latent features and compressed context-view camera parameters are packed into the scene bitstream $B$ [S1, unofficial].

4.  **Latent Decoding (Decoder Side):**
    *   **Entropy Decoding:** The scene bitstream $B$ is entropy-decoded to recover the quantized latent representation [S1, unofficial].
    *   **Synthesis Transform:** The latent representation is reconstructed into a feature map using a synthesis transform [S1, unofficial].
    *   **Camera Recovery:** The compressed context-view camera metadata is recovered [S1, unofficial].

5.  **Gaussian Reconstruction:** The decoded feature is used for:
    *   Depth prediction.
    *   Feature refinement.
    *   Gaussian parameter prediction (opacity, scale, rotation, color/SH) [S1, unofficial].
    *   2D-to-3D mapping to produce the final 3D Gaussian primitives $\hat{\mathcal{G}}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def codec_splat_encode(context_views):
    """
    Encodes context views into a scene bitstream.
    context_views: List of tuples (I, K, T, n, f) for N views.
    """
    # Step 1: Multi-view feature extraction
    # MVEnc extracts features and depth from context views
    F_mv, D_enc, P_depth = MVEnc(context_views)
    
    # Step 2: Construct Gaussian-generation feature
    # FeatHead combines images, multi-view features, and depth cues
    F_g = FeatHead(context_views.images, F_mv, P_depth)
    
    # Step 3: Latent encoding
    # Analysis transform converts F_g to latent space
    latent = analysis_transform(F_g)
    
    # Quantization with rate control lambda
    # lambda controls the trade-off between size and quality
    quantized_latent = quantize(latent, lambda)
    
    # Entropy coding
    bitstream_latents = entropy_encode(quantized_latent)
    
    # Compress camera parameters
    compressed_cameras = compress_cameras(context_views.cameras)
    
    # Pack bitstream
    bitstream = pack(bitstream_latents, compressed_cameras)
    
    return bitstream

def codec_splat_decode(bitstream):
    """
    Decodes a scene bitstream into 3D Gaussian primitives.
    bitstream: The encoded scene bitstream.
    """
    # Step 4: Latent decoding
    # Unpack bitstream
    bitstream_latents, compressed_cameras = unpack(bitstream)
    
    # Entropy decoding
    quantized_latent = entropy_decode(bitstream_latents)
    
    # Synthesis transform to reconstruct feature
    F_g_decoded = synthesis_transform(quantized_latent)
    
    # Recover camera metadata
    cameras = decompress_cameras(compressed_cameras)
    
    # Step 5: Gaussian reconstruction
    # Predict depth and refine features
    depth_pred, refined_features = depth_and_refine(F_g_decoded)
    
    # Predict Gaussian parameters
    # Heads predict opacity, scale, rotation, color/SH
    gaussians = predict_gaussians(refined_features, depth_pred, cameras)
    
    return gaussians
```

**Edge Cases and Pitfalls:**
*   **Rate-Distortion Trade-off:** The choice of $\lambda$ significantly impacts the bitstream size and reconstruction quality. A suboptimal $\lambda$ may lead to excessive compression artifacts or unnecessarily large bitstreams [S1, unofficial].
*   **Structured Feature Assumption:** The method relies on the intermediate feature $F_g$ being a regular, structured tensor. If the backbone fails to produce a coherent $F_g$, the entropy coding efficiency will degrade [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The method produces ultra-compact bitstreams. On the DL3DV dataset, scene sizes range from 20.00–107.77 KiB per scene. On the RealEstate10K dataset, sizes range from 3.37–12.51 KiB per scene [S1, unofficial].
*   **Measured Performance:**
    *   **DL3DV:** 23.56–26.36 dB PSNR [S1, unofficial].
    *   **RealEstate10K:** 24.76–27.05 dB PSNR [S1, unofficial].
*   **Scaling:** CodecSplat reduces scene size by roughly one order of magnitude compared with compressing feed-forward generated Gaussian primitives [S1, unofficial].
*   **Strengths:**
    *   Avoids inefficient compression of irregular 3D Gaussian primitives [S1, unofficial].
    *   Exploits the structured intermediate feature representation for better compression [S1, unofficial].
    *   Preserves controllable rate-distortion behavior [S1, unofficial].
*   **Limitations:**
    *   Relies on the quality of the feed-forward backbone (e.g., DepthSplat-style) to produce a compressible intermediate feature [S1, unofficial].
    *   Performance is dependent on the rate control parameter $\lambda$ [S1, unofficial].
*   **When to use:** When ultra-compact storage or transmission of feed-forward 3D Gaussian scenes is required, and the underlying feed-forward backbone is already trained.
*   **When not to use:** When per-scene optimization is acceptable and higher fidelity is prioritized over extreme compression ratios, or when the feed-forward backbone does not produce a structured intermediate feature suitable for entropy coding.

## Background
*   **Origin:** CodecSplat was introduced in the paper "CodecSplat: Ultra-Compact Latent Coding for Feed-Forward 3D Gaussian Splatting" [S1, unofficial].
*   **History:** It builds upon feed-forward 3D Gaussian Splatting methods, specifically those using depth-guided multi-view feature refinement (e.g., DepthSplat [S1, unofficial]). It addresses the limitation of existing 3DGS compression methods that operate on the final irregular 3D representation, which is decoupled from the internal feature-to-Gaussian generation process [S1, unofficial].
*   **Usage:** It is used for compressing scenes reconstructed by feed-forward 3D Gaussian splatting pipelines, enabling efficient storage and transmission of 3D Gaussian representations [S1, unofficial].

## Sources
[S1] CodecSplat: Ultra-Compact Latent Coding for Feed-Forward 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2605.25563, fetched 2026-10-29 (unofficial)
