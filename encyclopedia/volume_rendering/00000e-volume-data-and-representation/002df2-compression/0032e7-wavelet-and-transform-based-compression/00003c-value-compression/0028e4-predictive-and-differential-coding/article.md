# Predictive and Differential Coding

Also called: Differential Pulse Code Modulation (DPCM), Linear Predictive Coding (LPC), Delta Encoding

## Overview
Predictive and Differential Coding compresses volumetric data by exploiting spatial, temporal, or directional redundancy, encoding the difference (residual) between a predicted value and the actual value rather than the absolute value itself [S1, unofficial][S2, S3]. This family solves the problem of high entropy in correlated data by converting redundant absolute values into sparse, low-magnitude residuals that are more efficiently entropy-coded [S1, unofficial]. Unlike transform-based methods that decompose data into frequency subbands, predictive coding operates directly on the sample values (or transform coefficients) using local neighborhood information [S1, unofficial][S2]. It sets itself apart from siblings like [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) by relying on causal or spatial prediction models rather than hierarchical frequency decomposition, and from [Vector Quantization](907-vector-quantization.md) by encoding residuals rather than indexing codebooks.

## Shared principles
All methods in this family share the mechanism of estimating a current sample value $V_{pred}$ from previously processed neighbors or reference frames, computing a residual $R = V - V_{pred}$, and encoding $R$ [S1, unofficial][S2, S3].

1.  **Prediction Model:** The core of the method is the predictor function $V_{pred} = f(V_{neighbors})$. Models range from simple spatial deltas ($V_{prev}$) to complex linear combinations of neighbors or transform coefficients [S1, unofficial][S2, S3].
2.  **Residual Encoding:** The residual $R$ typically has a smaller dynamic range and higher probability of zero values than the original data, making it suitable for entropy coding (Huffman, Arithmetic, Run-Length) [S1, unofficial][S2, S3].
3.  **Causality and Order:** For lossless spatial compression, a strict scan order (e.g., raster-scan) ensures that only already-decoded neighbors are used for prediction, maintaining decoder consistency [S1, unofficial][S2, S3].
4.  **Closed-Loop vs. Open-Loop:** In lossy scenarios, predictors may use reconstructed (dequantized) values to prevent error accumulation (closed-loop) or original values for simplicity (open-loop) [S1, unofficial][S2, S3].

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding predicts the current value based on the immediately preceding value (spatial or temporal) and stores the difference. It is chosen for data with high coherence where changes are sparse. Unlike linear predictors, it uses a fixed model $V_{pred} = V_{prev}$, which prevents random access without storing reference frames.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
This method predicts current transform coefficients (e.g., DCT) from the previous frame's reconstructed coefficients, operating in the frequency domain. It sets itself apart from spatial delta encoding by concentrating residual energy near surface changes in the transform domain, improving compressibility for time-varying volumes.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
This technique encodes the difference between the original view and the direct reconstruction from unquantized volume data. It decouples image fidelity from view synthesis quality, allowing residuals to be truncated for bitrate scaling without affecting the base volume's view synthesis properties.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
A lossless method that derives optimal linear predictor coefficients for each volume to minimize mean-square error, using a causal 6-neighborhood predictor and truncated Huffman coding. It outperforms generic compressors by explicitly removing spatial redundancy and adapts to the specific volume statistics.

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
An isotropic lossless predictor for $n$-dimensional fields that estimates a sample from its $2^n - 1$ immediate neighbors using a specific algebraic formula. It is distinguished by its ability to exactly reconstruct polynomials of degree less than $n$ and its isotropic exploitation of coherence, unlike anisotropic scanline predictors.

### [RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md)
Combines a modified 3D DWT with spatial prediction on the final subbands. It uses Reversible Denoising Lifting Steps (RDLS) and Step Skipping (SS) to reduce noise and redundancy before applying a heuristic-selected spatial predictor to decorrelate the highest-frequency subbands.

### [Closed-Loop DPCM Decompression](11737-closed-loop-dpcm-decompression.md)
Reconstructs data (e.g., seismic traces) using inverse quantization and delta decoding within a closed-loop feedback structure. It ensures that quantization errors do not accumulate unbounded drift by using the reconstructed value as the prediction base for the next sample.

### Directional Linear Predictive Coding
- **Definition:** Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
- **Steps:** Segment interference patterns into blocks. Scan pixels in eight directions for each block. Calculate entropy in each direction. Determine the direction with minimum entropy. Encode the difference by DPCM in the determined direction. Calculate prediction coefficients for the determined direction. Use DPCM only to determine the prediction direction for the original pixel. Calculate the difference between the predicted and the original pixel [S1, unofficial].
- **Performance:** The first process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

### JPEG-LS
- **Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
- **Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

### Inter-frame Delta Encoding with Haar Wavelets
- **Definition:** Classifies each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

### Functional Representation (Spherical Ellipsoids)
- **Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding in volume rendering originated from the need to compress large medical and scientific datasets for efficient storage and transmission. Early work focused on spatial redundancy using linear predictors [S1, unofficial][S2]. The application to holographic displays highlighted the importance of directional correlation in interference patterns [S1, unofficial]. Temporal compression methods evolved from simple delta encoding to transform-domain residuals to handle time-varying volumes [S3]. The integration of predictive coding with wavelet transforms (e.g., JP3D) became a standard approach for lossless and near-lossless compression in medical imaging [S2, S3].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
