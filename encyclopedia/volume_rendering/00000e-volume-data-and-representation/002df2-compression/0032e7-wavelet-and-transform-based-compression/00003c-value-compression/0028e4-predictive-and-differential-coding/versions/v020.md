# Predictive and Differential Coding

Also called: Differential Pulse Code Modulation (DPCM), Linear Predictive Coding (LPC), Delta Encoding

## Overview
Predictive and Differential Coding compresses volumetric data by exploiting local spatial or temporal coherence to predict voxel values from neighbors, encoding only the residual difference (error) rather than the absolute values [S1, unofficial]. This method sets itself apart from transform-based siblings like [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) by operating directly on raw values in spatial or temporal order without a linear transform, making it particularly effective for data with high local continuity, such as time-varying volumes or heightfields [S1, unofficial][S2]. Unlike [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md) which corrects view-synthesis errors, predictive coding targets the intrinsic redundancy of the scalar field itself.

## Shared principles
The family shares a core mechanism of prediction and residual encoding:
1.  **Prediction:** A voxel value $v_i$ is estimated as $\hat{v}_i$ based on a set of previously encoded or spatially adjacent neighbors. The predictor can be simple (e.g., previous value) or complex (e.g., linear combination of multiple neighbors).
2.  **Residual Calculation:** The difference between the actual value and the prediction is computed: $d_i = v_i - \hat{v}_i$.
3.  **Encoding:** The residual $d_i$ is entropy-coded. Because residuals typically have a smaller dynamic range and higher probability of being zero than raw values, they compress more efficiently.
4.  **Reconstruction:** The decoder uses the same predictor logic and the decoded residuals to reconstruct the original values: $v_i = \hat{v}_i + d_i$.

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
[Delta Encoding](1096-delta-encoding.md) predicts the current value based on the immediately preceding element in a traversal order (spatial or temporal) and stores the difference. It is chosen for data with high local or temporal coherence where values change slowly. Unlike transform methods, it requires no frequency decomposition.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
[Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md) operates in the frequency domain, predicting current transform coefficients from the previously reconstructed frame's coefficients. It sets itself apart from spatial predictors by concentrating residual energy near regions of surface change in the transform domain, improving compressibility for subsequent entropy coding.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
[Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md) encodes the difference between the original view and the direct reconstruction from the volume without quantization. It is used to improve reconstructed image quality at high bitrates but does not contribute to view synthesis quality and may deteriorate it.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
[Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md) derives optimal linear predictor coefficients for each volume to minimize mean-square error, encoding residuals with a truncated Huffman code. It outperforms generic compressors by explicitly removing spatial redundancy.

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
[Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md) estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals, encoded via arithmetic coding. It is distinguished by exploiting data coherence in all dimensions isotropically.

### [RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md)
[RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md) combines a modified Discrete Wavelet Transform (DWT) with predictive coding on the final subbands. It sets itself apart by integrating denoising and step-skipping heuristics before prediction, leveraging the multi-resolution structure of the wavelet tree.

### [Closed-Loop DPCM Decompression](11737-closed-loop-dpcm-decompression.md)
[Closed-Loop DPCM Decompression](11737-closed-loop-dpcm-decompression.md) reconstructs data by reversing a predictive coding scheme that maintains state consistency between encoding and decoding steps. It prevents error accumulation by explicitly handling quantization parameters and bit-depth metadata.

### Directional Linear Predictive Coding
#### definition
- Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
#### step
- Segment interference patterns into blocks [S1, unofficial].
- Scan the pixels in eight directions for each block [S1, unofficial].
- Calculate entropy in each direction [S1, unofficial].
- Determine the direction with minimum entropy [S1, unofficial].
- Encode the difference by DPCM in the determined direction [S1, unofficial].
- Calculate prediction coefficients for the determined direction [S1, unofficial].
- Use DPCM only to determine the prediction direction for the original pixel [S1, unofficial].
- Calculate the difference between the predicted and the original pixel [S1, unofficial].
#### number
- First process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

### JPEG-LS
#### definition
- An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
#### comparison
- Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

### Inter-frame Delta Encoding with Haar Wavelets
#### definition
- Classifying each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

### Functional Representation (Spherical Ellipsoids)
#### definition
- An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding has been a fundamental technique in signal processing for decades. In the context of volume rendering and medical imaging, it has been applied to various data types, including holographic interference patterns [S1, unofficial], medical datasets like ANGIO and MR-MRI [S2], and time-varying volumetric data [S3]. The method is often compared against transform-based approaches like JPEG2000 and wavelet codecs, with performance varying based on the data's correlation structure [S2].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
