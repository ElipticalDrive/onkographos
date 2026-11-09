# Predictive and Differential Coding

Also called: Differential Pulse Code Modulation (DPCM), Linear Predictive Coding (LPC), Residual Coding

## Overview
Predictive and differential coding compresses volumetric data by encoding the difference (residual) between a current sample and a predicted value derived from previously encoded neighbors, rather than the absolute value [S1, unofficial]. This method exploits spatial or temporal coherence to produce residuals with lower entropy than the original data, which are then entropy-coded. It sets itself apart from transform-based methods (like [Wavelet and Transform-based Compression](828-wavelet-and-transform-based-compression.md)) by operating directly on sample values (or transform coefficients) without a global frequency decomposition, and from [Vector Quantization](907-vector-quantization.md) by relying on local causal prediction rather than codebook matching.

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of residuals $R$ using a prediction model $P$, followed by entropy encoding of $R$.

1.  **Prediction Model:** A predictor $\hat{v}$ estimates the value of a sample $v$ based on a set of previously encoded neighbors $N$. The prediction can be spatial (using neighbors in the same volume) or temporal (using values from previous time steps).
    $$ \hat{v} = P(N) $$
2.  **Residual Calculation:** The residual $d$ is the difference between the actual value and the prediction.
    $$ d = v - \hat{v} $$
    If the prediction is accurate, $d$ will be small, often zero for unchanged regions [S1, unofficial].
3.  **Entropy Encoding:** The residuals $d$ are encoded using entropy coding (e.g., Huffman, Arithmetic, or Variable-Length Coding) [S1, unofficial]. Because the distribution of $d$ is often non-uniform and concentrated around zero, this step achieves significant compression [S1, unofficial].
4.  **Reconstruction:** To reconstruct the original value, the decoder adds the decoded residual to the predicted value.
    $$ v = \hat{v} + d $$
    For this to work, the decoder must have access to the same previously decoded neighbors used for prediction, enforcing a specific scan order or causal dependency [S1, unofficial].

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding is a predictive compression method that exploits coherence in data by storing the difference (residual) between a current sample and a predicted value derived from previously encoded neighbors, rather than the absolute value. In volume rendering, it is primarily used to compress time-varying data by assuming temporal coherence (that voxel values do not change between time steps) or spatially coherent heightfields. It sets itself apart from sibling methods like [Optimal Linear Volumetric Predictor](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md) by using a simple identity or nearest-neighbor prediction model, which minimizes computational overhead during encoding/decoding but requires sequential access for reconstruction.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
Residual Updates in Transform Domain is a temporal compression method for volumetric data that operates in the frequency domain rather than the spatial domain. It solves the problem of encoding time-varying volumes by predicting current transform coefficients from the previously reconstructed frame's coefficients, rather than predicting voxel values directly. It sets itself apart from spatial predictors like [Delta Encoding](1096-delta-encoding.md) by concentrating residual energy near regions of surface change in the transform domain, which improves compressibility for subsequent entropy coding.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
Residual coding is a technique that improves the compression performance (reconstructed image quality) of volumetric codecs by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding. Unlike standard predictive coding which aims to reduce view synthesis artifacts, this method does not contribute to view synthesis quality and may deteriorate it. It sets itself apart from siblings like [Delta Encoding](1096-delta-encoding.md) by deriving residuals from the unquantized volume reconstruction rather than previous samples, ensuring residuals encoded at all bitrates are similar.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
This method is a lossless compression algorithm for 3D volume data that removes spatial redundancy by predicting voxel values from previously encoded neighbors and encoding the resulting residuals using a truncated Huffman code. It differs from generic compressors (e.g., gzip, zip) by explicitly modeling spatial coherence via a causal linear predictor, and from fixed-coefficient predictors by deriving optimal coefficients from the data's global statistics. It achieves approximately 50% compression ratios on biomedical volume datasets, outperforming standard general-purpose tools.

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
The Lorenz Predictor with Arithmetic Coding is a lossless compression method for $n$-dimensional scalar fields that estimates each sample value from its $2^n - 1$ previously visited neighbors in a regular grid to generate residuals, which are then encoded using arithmetic coding. It is distinguished from anisotropic scanline predictors (like simple delta encoding) by being isotropic, exploiting data coherence in all dimensions, and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$.

### [RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md)
RDLS-SS-DWT+Pred is a lossless volumetric compression method that constructs a hybrid transform by applying predictive coding to the final subbands of a Reversible Denoising Lifting Steps with Step Skipping Discrete Wavelet Transform (RDLS-SS-DWT). It solves the problem of high entropy in transformed volumetric data by reducing residual variance through spatial prediction, specifically leveraging the characteristics of subbands generated when Null filters are used in the RDLS stage. It sets itself apart from its parent [Predictive and Differential Coding](4393-predictive-and-differential-coding.md) by integrating prediction into a multi-level 3D wavelet hierarchy rather than applying it directly to raw voxels, and from its sibling [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) by optionally including reversible denoising lifting steps, though sources indicate the denoising steps may be omitted without significant bitrate loss.

### Directional Linear Predictive Coding
- **Definition:** Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
- **Steps:**
    1.  Segment interference patterns into blocks [S1, unofficial].
    2.  Scan the pixels in eight directions for each block [S1, unofficial].
    3.  Calculate entropy in each direction [S1, unofficial].
    4.  Determine the direction with minimum entropy [S1, unofficial].
    5.  Encode the difference by DPCM in the determined direction [S1, unofficial].
    6.  Calculate prediction coefficients for the determined direction [S1, unofficial].
    7.  Use DPCM only to determine the prediction direction for the original pixel [S1, unofficial].
    8.  Calculate the difference between the predicted and the original pixel [S1, unofficial].
- **Performance:** The first process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

### JPEG-LS
- **Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
- **Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

### Inter-frame Delta Encoding with Haar Wavelets
- **Definition:** Classifying each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

### Functional Representation (Spherical Ellipsoids)
- **Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding has been a fundamental technique in image and video compression for decades, extending naturally to volumetric data. In the context of volume rendering, it is often used to handle time-varying datasets where temporal coherence is high [S3]. The method is particularly effective for biomedical and geological data, which often exhibit smooth variations and large regions of uniformity [S1, unofficial]. While transform-based methods like wavelets are dominant in modern standards (e.g., JPEG2000), predictive coding remains relevant for specific applications such as holographic display generation [S1, unofficial] and scenarios where low-complexity decoding is required. The integration of prediction with transforms, as seen in RDLS-SS-DWT+Pred, represents a hybrid approach that leverages the strengths of both domains [S1, unofficial].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
