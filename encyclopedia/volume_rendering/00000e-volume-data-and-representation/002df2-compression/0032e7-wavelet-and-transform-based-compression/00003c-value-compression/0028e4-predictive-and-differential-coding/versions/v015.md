# Predictive and Differential Coding

Also called: Differential Pulse-Code Modulation (DPCM), Predictive Coding, Residual Coding

## Overview
Predictive and Differential Coding compresses volumetric data by exploiting spatial or temporal coherence to encode the difference (residual) between a predicted value and the actual value, rather than encoding the absolute values [S1, unofficial]. This family sets itself apart from transform-based methods (like [Wavelet and Transform-based Compression](52-wavelet-and-transform-based-compression.md)) by operating directly on the signal domain (spatial or temporal) to reduce redundancy before entropy coding, whereas transform methods first convert the data into a frequency domain. It differs from [Vector Quantization](907-vector-quantization.md) by using explicit mathematical predictors rather than codebook-based approximation.

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of coefficients $C$ using a linear transform, followed by encoding and optional decoding for rendering.

1.  **Transform:** The volume is decomposed using a separable 1D filter bank applied along each axis. For a 3D volume, this results in subbands representing different frequency components. In wavelet transforms, these are typically approximation (low-pass) and detail (high-pass) coefficients. In Laplacian pyramids, they are detail residuals between levels of a Gaussian pyramid.
2.  **Hierarchy:** The decomposition is recursive. The approximation subband from level $j$ is further decomposed at level $j+1$, creating a tree-like structure of coefficients. This allows for multi-resolution access, where coarse data is available at the root and fine details are deeper in the tree.
3.  **Quantization and Entropy Encoding:** Coefficients are quantized (lossy) or kept precise (lossless). Entropy coding (e.g., Huffman, Arithmetic, Run-Length) is applied to the quantized or raw coefficients to produce the compressed bitstream.
4.  **Reconstruction:** To render, the necessary coefficients are decoded and the inverse transform is applied. This can be done globally (full reconstruction) or locally (block-wise or point-wise) to support adaptive resolution rendering.

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding (also known as Differential Pulse-Code Modulation or DPCM) is a predictive compression method that exploits coherence in data by storing the difference between a predicted value and the actual value. In volume rendering, it is primarily used to compress time-varying data by predicting that a voxel's value remains unchanged between time steps, or heightfields where values vary progressively. It sets itself apart from sibling methods like [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md) by using a constant predictor (identity or previous time step) rather than a complex linear combination of spatial neighbors, resulting in lower computational overhead but potentially higher residual entropy if the data changes rapidly.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
Residual Updates in Transform Domain is a temporal compression method for volumetric data that operates in the frequency domain rather than the spatial domain. It solves the problem of encoding time-varying volumes by predicting current transform coefficients from the previously reconstructed frame's coefficients, rather than predicting voxel values directly. It sets itself apart from spatial predictors like [Delta Encoding](1096-delta-encoding.md) by concentrating residual energy near regions of surface change in the transform domain, which improves compressibility for subsequent entropy coding.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
Residual Coding is a technique that improves the compression performance (reconstructed image quality) of a volumetric codec by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding. It does not contribute to view synthesis quality and may deteriorate it. Unlike standard video codecs where residuals depend on the quantized output, these residuals are derived from the unquantized volume reconstruction, making them consistent across all bitrates.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
This method is a lossless compression algorithm for 3D volume data that exploits spatial coherence by predicting voxel values from previously encoded neighbors and encoding the difference (residual) using a truncated Huffman code. It achieves approximately 50% compression ratios on medical volume datasets, outperforming generic compressors like gzip and zip by explicitly removing spatial redundancy rather than just statistical redundancy. Unlike simple delta encoding, it uses an optimal linear predictor derived from the data's covariance to minimize prediction error variance.

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
The Lorenz Predictor with Arithmetic Coding is a lossless compression method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals, which are then entropy-encoded using arithmetic coding. It is distinguished from anisotropic scanline predictors by being isotropic, exploiting data coherence in all dimensions, and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$.

### [RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md)
RDLS-SS-DWT+Pred is a lossless volumetric compression transform that combines the 3D Discrete Wavelet Transform (DWT) with Reversible Denoising Lifting Steps (RDLS) and step skipping, followed by predictive coding on the final subbands. It solves the problem of high entropy in wavelet coefficients by applying spatial prediction to the transformed data, similar to techniques used in 2D image compression. It differs from its sibling [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) by incorporating denoising filters in the lifting steps, though sources indicate that using actual denoising filters may be avoided without sacrificing bitrate improvements, making it functionally similar to [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) in practice.

### [Closed-Loop DPCM Decompression](11737-closed-loop-dpcm-decompression.md)
Closed-loop DPCM decompression reconstructs seismic trace blocks from compressed payloads using a feedback mechanism where the decoder's reconstructed values drive the prediction for subsequent samples. Unlike open-loop methods that rely on stored quantization parameters or legacy delta streams, this method dynamically tracks the reconstruction state to ensure bit-exact recovery. It is distinguished by its explicit check for a "closed-loop" flag in the payload, falling back to a legacy decoder if the format is incompatible.

### Directional Linear Predictive Coding
**Definition:** Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
**Steps:**
1.  Segment interference patterns into blocks [S1, unofficial].
2.  Scan the pixels in eight directions for each block [S1, unofficial].
3.  Calculate entropy in each direction [S1, unofficial].
4.  Determine the direction with minimum entropy [S1, unofficial].
5.  Encode the difference by DPCM in the determined direction [S1, unofficial].
6.  Calculate prediction coefficients for the determined direction [S1, unofficial].
7.  Use DPCM only to determine the prediction direction for the original pixel [S1, unofficial].
8.  Calculate the difference between the predicted and the original pixel [S1, unofficial].
**Performance:** The first process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

### JPEG-LS
**Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
**Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

### Inter-frame Delta Encoding with Haar Wavelets
**Definition:** Classifying each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

### Functional Representation (Spherical Ellipsoids)
**Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding has been a fundamental technique in signal processing for decades, adapted for volume rendering to handle the massive data sizes inherent in medical imaging and scientific visualization. The specific application to holographic display methods for volume data was explored by Hyun, Tamagawa, and Sakamoto, who demonstrated the efficacy of directional linear predictive coding for hologram interference patterns [S1, unofficial]. The broader context of compressed GPU-based direct volume rendering highlights the necessity of such techniques to allow streaming and rendering components to perform efficiently [S3]. Early comparisons of 3-D encoding/2-D decoding approaches, such as those involving JPEG-LS and MLZC, established the trade-offs between compression efficiency, quality scalability, and computational complexity in medical data compression [S2].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
