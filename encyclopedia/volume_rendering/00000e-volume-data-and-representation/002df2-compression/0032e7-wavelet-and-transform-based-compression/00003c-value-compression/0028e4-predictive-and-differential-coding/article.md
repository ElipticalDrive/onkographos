# Predictive and Differential Coding

Also called: Differential Coding, Predictive Coding, DPCM

## Overview
Predictive and Differential Coding reduces data redundancy by encoding the difference (residual) between a predicted value and the actual value, rather than encoding the absolute values [S1, unofficial]. This family sets itself apart from transform-based methods (like [Wavelet and Transform-based Compression](6879-wavelet-and-transform-based-compression.md)) by operating directly on the signal domain (spatial or temporal) to exploit local coherence, often resulting in residuals with high entropy concentration (e.g., many zeros) that compress efficiently with entropy coders [S1, unofficial]. Unlike [Vector Quantization](907-vector-quantization.md) which maps blocks to codebooks, predictive coding is typically sequential and causal, relying on previously decoded data to predict the current sample [S1, unofficial].

## Shared principles
The core mechanism involves three steps common to all variants:
1.  **Prediction:** Estimate the current voxel value $\hat{v}$ using a predictor function $P$ applied to previously processed data (spatial neighbors or temporal predecessors).
    $$ \hat{v} = P(\text{context}) $$
2.  **Residual Calculation:** Compute the difference between the actual value $v$ and the prediction $\hat{v}$.
    $$ d = v - \hat{v} $$
3.  **Encoding:** Encode the residual $d$ using entropy coding (e.g., Huffman, Arithmetic, or Range coding). The efficiency relies on the residual distribution being narrower or more sparse than the original data distribution.

**Key Constraint:** In lossy scenarios or when quantization is applied to residuals, the encoder must use the *reconstructed* (quantized) previous values for prediction to prevent encoder-decoder drift [S1, unofficial]. This is known as closed-loop prediction.

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding (or DPCM) predicts the current voxel based on the immediately preceding value in a scan order (spatial) or the previous time step (temporal). It is chosen when data exhibits strong local continuity or temporal coherence. It differs from transform methods by being inherently sequential and causal.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
This method performs temporal prediction in the frequency domain (specifically DCT) rather than the spatial domain. It is chosen for time-varying implicit surfaces (e.g., TSDF) where surface changes are localized in frequency. It differs from [Delta Encoding](1096-delta-encoding.md) by operating on coefficients, allowing energy concentration near surface changes.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
This technique encodes the difference between the original image and an unquantized reconstruction of the volume. It is chosen to improve PSNR at high bitrates without affecting view synthesis quality. It differs from standard predictive coding because residuals are derived from an unquantized reference, making them bitrate-independent.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
A lossless method using an optimal linear predictor derived from data covariance to minimize residual variance, combined with a truncated Huffman code. It is chosen for medical volumes with high spatial coherence. It differs from simple delta encoding by using a multi-neighbor linear model and from full Huffman coding by truncating the codebook for memory efficiency.

### Directional Linear Predictive Coding
- **Definition:** Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
- **Steps:**
    1. Segment interference patterns into blocks [S1, unofficial].
    2. Scan pixels in eight directions for each block [S1, unofficial].
    3. Calculate entropy in each direction [S1, unofficial].
    4. Determine the direction with minimum entropy [S1, unofficial].
    5. Encode the difference by DPCM in the determined direction [S1, unofficial].
    6. Calculate prediction coefficients for the determined direction [S1, unofficial].
    7. Use DPCM only to determine the prediction direction for the original pixel [S1, unofficial].
    8. Calculate the difference between the predicted and the original pixel [S1, unofficial].
- **Performance:** The first process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

### JPEG-LS
- **Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
- **Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

### Inter-frame Delta Encoding with Haar Wavelets
- **Definition:** Classifies each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

### Functional Representation (Spherical Ellipsoids)
- **Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids derived by [JBL∗ 06] [S3].

### Lorenz Predictor with Arithmetic Coding
- **Steps:** Compress the prediction error using arithmetic coding [S4, unofficial].

## Background
Predictive coding originates from Differential Pulse-Code Modulation (DPCM) used in telecommunications [S1, unofficial]. In volume rendering, it was adapted to exploit spatial redundancy in medical imaging (e.g., MRI, CT) and temporal redundancy in time-varying datasets [S3]. Early applications included lossless compression of medical data using linear predictors [S2] and holographic display data [S1, unofficial]. The method is widely used in conjunction with entropy coding to achieve high compression ratios for volumetric data with smooth gradients [S4, unofficial].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S4] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
