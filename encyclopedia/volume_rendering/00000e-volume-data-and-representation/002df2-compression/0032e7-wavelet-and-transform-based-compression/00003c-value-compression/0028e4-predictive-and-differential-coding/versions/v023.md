# Predictive and Differential Coding

Also called: Differential Coding, Predictive Coding, DPCM in Volume Rendering

## Overview
Predictive and Differential Coding reduces data redundancy by encoding the difference (residual) between a predicted value and the actual value, exploiting local spatial, temporal, or directional coherence in volumetric data [S1, unofficial]. Unlike transform-based methods that operate on frequency coefficients, this family often operates directly in the spatial or temporal domain, or applies prediction to the final subbands of a transform, relying on the statistical concentration of small differences to enable efficient entropy coding [S1, unofficial]. It sets itself apart from sibling methods like [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) by focusing on value correlation rather than hierarchical frequency decomposition, and from [Vector Quantization](907-vector-quantization.md) by preserving exact values (in lossless modes) through residual encoding rather than codebook approximation.

## Shared principles
All methods in this family share the mechanism of estimating a voxel's value $V_{pred}$ from previously processed neighbors or reference frames, computing a residual $R = V_{actual} - V_{pred}$, and encoding $R$ [S1, unofficial].

1.  **Prediction:** The core operation is generating $V_{pred}$ from a causal neighborhood (spatial) or a reference frame (temporal). The predictor can be fixed (e.g., previous value, zero) or adaptive (e.g., linear combination of neighbors, optimal coefficients derived from covariance) [S1, unofficial][S2].
2.  **Residual Calculation:** The residual $R$ is computed. In lossless contexts, $R$ is exact. In lossy contexts, $R$ may be quantized. The goal is to minimize the variance of $R$, concentrating its probability mass near zero [S1, unofficial].
3.  **Entropy Coding:** The residuals are encoded using entropy coding (Huffman, Arithmetic, or Range coding). Because residuals are typically small integers with a Laplacian-like distribution, they compress significantly better than raw values [S1, unofficial].
4.  **Closed-Loop vs. Open-Loop:** In temporal or iterative contexts, a closed-loop structure is often required where the decoder uses its *reconstructed* value (not the original) as the predictor for the next step to prevent error accumulation [S1, unofficial].

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding (DPCM) stores the difference between a predicted value and the actual value. It is primarily used for time-varying volumes (predicting $V_t$ from $V_{t-1}$) or heightfields. It sets itself apart by operating directly on raw spatial/temporal data without a transform domain, relying on the high probability of zero differences in static regions.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
This method predicts current transform coefficients from the previously reconstructed frame's coefficients. It sets itself apart from spatial predictors by concentrating residual energy near regions of surface change in the frequency domain, improving compressibility for subsequent entropy coding.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
This technique encodes the difference between the original view and the direct reconstruction from the volume (without quantization). It sets itself apart by operating in the pixel domain to correct high-frequency loss from imperfect volume construction, rather than optimizing the transform domain directly.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
A lossless algorithm that derives optimal linear predictor coefficients via the Yule-Walker equations to minimize mean-square error, adapting to non-isotropic voxel sampling. It sets itself apart from fixed predictors by explicitly modeling spatial coherence to achieve ~50% compression ratios on medical data.

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
An isotropic lossless predictor for $n$-dimensional fields that estimates a sample using its $2^n - 1$ previously visited neighbors. It sets itself apart by being the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$, exploiting coherence in all dimensions simultaneously.

### [RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md)
Combines Reversible Denoising Lifting Steps with Step Skipping (RDLS-SS-DWT) with predictive coding applied to the final subbands. It sets itself apart by further decorrelating wavelet coefficients using spatial prediction, achieving significant bitrate reductions compared to standard DWT-based methods.

### [Closed-Loop DPCM Decompression](11737-closed-loop-dpcm-decompression.md)
Reconstructs data (e.g., seismic traces) by applying inverse quantization and delta decoding within a closed-loop feedback structure. It sets itself apart by using the reconstructed value as the predictor for the next sample, ensuring quantization errors do not accumulate unboundedly.

### [Spatial Predictive Coding](12623-spatial-predictive-coding.md)
Reduces information redundancy in a voxel grid by exploiting statistical dependence between spatially adjacent samples, analogous to adjacent pixel correlation in images.

#### Directional Linear Predictive Coding
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

#### JPEG-LS
- **Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
- **Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

#### Inter-frame Delta Encoding with Haar Wavelets
- **Definition:** Classifies each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

#### Functional Representation (Spherical Ellipsoids)
- **Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding originates from Differential Pulse-Code Modulation (DPCM) in telecommunications [S1, unofficial]. In volume rendering, it has been adapted for time-varying data (e.g., Ko et al. [KLW*08] using Haar wavelets with delta encoding [S3]) and spatial compression (e.g., holographic displays [S1, unofficial], medical imaging [S2]). The method is widely used in scenarios where spatial or temporal coherence is high, such as biomedical science and geology [S1, unofficial][S3].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
