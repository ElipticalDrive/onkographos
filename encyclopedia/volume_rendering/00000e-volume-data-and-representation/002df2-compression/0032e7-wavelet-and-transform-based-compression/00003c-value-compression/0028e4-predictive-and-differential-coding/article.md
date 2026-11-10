# Predictive and Differential Coding
Also called: Differential Pulse-Code Modulation (DPCM), Residual Coding, Spatial/Temporal Prediction

## Overview
Predictive and Differential Coding compresses volumetric data by encoding the difference (residual) between a predicted value and the actual value, exploiting local spatial or temporal coherence [S1, unofficial]. Unlike transform-based methods that decorrelate data in the frequency domain, this family operates directly in the spatial or coefficient domain using causal or temporal predictors, relying on the statistical concentration of small residuals to enable efficient entropy coding [S1, unofficial]. It sets itself apart from siblings like [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) by avoiding global transform overhead in favor of local prediction, and from [Vector Quantization](907-vector-quantization.md) by preserving exact values in lossless modes without codebook mapping.

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of coefficients $C$ using a linear transform, followed by encoding and optional decoding for rendering.

1.  **Transform:** The volume is decomposed using a separable 1D filter bank applied along each axis. For a 3D volume, this results in subbands representing different frequency components. In wavelet transforms, these are typically approximation (low-pass) and detail (high-pass) coefficients. In Laplacian pyramids, they are detail residuals between levels of a Gaussian pyramid.
2.  **Hierarchy:** The decomposition is recursive. The approximation subband from level $j$ is further decomposed at level $j+1$, creating a tree-like structure of coefficients. This allows for multi-resolution access, where coarse data is available at the root and fine details are deeper in the tree.
3.  **Quantization and Entropy Encoding:** Coefficients are quantized (lossy) or kept precise (lossless). Entropy coding (e.g., Huffman, Arithmetic, Run-Length) is applied to the quantized or raw coefficients to produce the compressed bitstream.
4.  **Reconstruction:** To render, the necessary coefficients are decoded and the inverse transform is applied. This can be done globally (full reconstruction) or locally (block-wise or point-wise) to support adaptive resolution rendering.

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding (DPCM) stores the difference between a predicted value and the actual value, primarily used for time-varying datasets or heightfields. It sets itself apart by operating directly on the raw spatial or temporal domain, relying on the statistical concentration of small differences to enable efficient entropy coding. Choose it when data exhibits strong temporal coherence or monotonic spatial variation.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
This method predicts current transform coefficients from the previously reconstructed frame's coefficients, operating in the frequency domain. It sets itself apart from spatial predictors by concentrating residual energy near regions of surface change in the transform domain, improving compressibility for subsequent entropy coding. Choose it for time-varying volumetric data where the transform domain offers better decorrelation than the spatial domain.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
This technique encodes the difference between the original image and the direct reconstruction from the volume without quantization or coding. It sets itself apart by operating in the pixel domain to correct errors from imperfect volume construction or high-frequency loss, rather than optimizing the transform domain directly. Choose it to improve reconstructed image quality at high bitrates, though it may deteriorate view synthesis quality.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
A lossless algorithm that removes spatial redundancy by predicting voxel values from previously encoded neighbors and encoding the difference using a truncated Huffman code. It sets itself apart by deriving optimal linear predictor coefficients via the Yule-Walker equations to minimize mean-square error, adapting to non-isotropic voxel sampling. Choose it for medical volume datasets requiring high compression ratios with lossless fidelity.

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
This method estimates sample values using all $2^n - 1$ previously visited neighbors in a regular grid to generate residuals, encoded via arithmetic coding. It sets itself apart by being isotropic, exploiting data coherence in all dimensions simultaneously, and exactly reconstructing all polynomials of degree less than $n$. Choose it for $n$-dimensional scalar fields where isotropic prediction is required.

### [RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md)
This transform combines Reversible Denoising Lifting Steps with Step Skipping (RDLS-SS-DWT) with predictive coding applied to the final subbands. It sets itself apart by incorporating denoising filters within the lifting steps to further decorrelate data and reduce entropy. Choose it when standard DWT-based methods exhibit high entropy in detail subbands.

### [Closed-Loop DPCM Decompression](11737-closed-loop-dpcm-decompression.md)
This method reconstructs data by applying inverse quantization and delta decoding within a closed-loop feedback structure. It sets itself apart from open-loop delta encoding by using the reconstructed value as the predictor for the next sample, ensuring quantization errors do not accumulate unboundedly. Choose it for seismic trace data or legacy formats requiring backward compatibility and error stability.

#### Directional Linear Predictive Coding
**Definition:** Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
**Steps:** Segment interference patterns into blocks. Scan the pixels in eight directions for each block. Calculate entropy in each direction. Determine the direction with minimum entropy. Encode the difference by DPCM in the determined direction. Calculate prediction coefficients for the determined direction. Use DPCM only to determine the prediction direction for the original pixel. Calculate the difference between the predicted and the original pixel [S1, unofficial].
**Performance:** The first process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

#### JPEG-LS
**Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
**Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

#### Inter-frame Delta Encoding with Haar Wavelets
**Definition:** Classifying each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

#### Functional Representation (Spherical Ellipsoids)
**Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding in volume rendering originated from the need to handle large medical and scientific datasets efficiently. Early work focused on spatial redundancy removal using linear predictors optimized via Yule-Walker equations for medical volumes [S1, unofficial]. The application of DPCM to time-varying data was formalized in systems like those by Ko et al., who classified frames as intra or predictive and applied delta encoding on top of wavelet transforms [S3]. Directional predictive coding was later applied to specialized data like holographic interference patterns to exploit anisotropic correlations [S1, unofficial]. The family has evolved to include closed-loop mechanisms for seismic data to prevent error accumulation [S1, unofficial] and hybrid approaches combining lifting wavelets with spatial prediction for final subbands [S1, unofficial].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
