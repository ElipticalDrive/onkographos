# Predictive and Differential Coding

Also called: Differential Pulse-Code Modulation (DPCM), Linear Predictive Coding (LPC), Delta Encoding

## Overview
Predictive and Differential Coding compresses volume data by storing the difference (residual) between a predicted value and the actual value, exploiting spatial continuity or temporal coherence to concentrate residuals around zero [S1, unofficial][S2]. It distinguishes itself from sibling methods by operating directly on the raw residual stream before entropy coding, often resulting in uniform bricks of zeros for static regions, whereas siblings like [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) operate in the transform domain or [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md) integrate prediction and coding more tightly with specific covariance models.

## Shared principles
The family relies on the fundamental relationship:
$$ d = v - \hat{v} $$
where $v$ is the actual voxel value, $\hat{v}$ is the predicted value, and $d$ is the delta (residual). The prediction $\hat{v}$ is derived from previously encoded or reconstructed data. The core mechanism involves:
1.  **Prediction:** Estimating $v$ based on a model (e.g., previous time step, spatial neighbors, or transform coefficients).
2.  **Residual Calculation:** Computing $d$. If the prediction is accurate, $d$ will be small or zero.
3.  **Encoding:** The residuals $d$ are entropy-coded. Because $d$ is concentrated around zero, it requires fewer bits than the original data $v$.
4.  **Reconstruction:** The decoder recovers $v$ by adding the decoded residual to the predicted value:
    $$ v = \hat{v} + d $$

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding (DPCM) compresses volume data by storing the difference between a predicted value and the actual value, exploiting temporal coherence in time-varying data or spatial continuity in heightfields. It distinguishes itself from siblings by focusing on the raw residual stream before entropy coding, often resulting in uniform bricks of zeros for static regions, which are highly compressible, whereas siblings like [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md) may integrate prediction and coding more tightly or operate in the transform domain.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
Residual Updates in Transform Domain is a temporal compression method for volumetric data (specifically TSDF) that performs motion compensation in the frequency domain rather than the spatial domain. It solves the problem of encoding dynamic volumetric scenes by exploiting temporal coherence between frames. Unlike spatial predictors that estimate voxel values from neighbors, this method subtracts the DCT coefficients of the previously reconstructed frame from the current frame's coefficients to form residuals. This approach avoids the need for explicit geometric correspondence tracking, relying instead on the regular, indexable structure of voxel blocks.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
Residual Coding is a technique that improves the compression performance (reconstructed image quality) of volumetric codecs by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding. It does not contribute to view synthesis quality and may deteriorate it. Unlike standard video codecs where residuals depend on the quantized output, these residuals are derived from the unquantized volume reconstruction, making the encoded residuals similar across all bitrates.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
This method is a lossless compression algorithm for 3D volume data that removes spatial redundancy by predicting voxel values from previously encoded neighbors and encoding the resulting residuals using a truncated Huffman code. It distinguishes itself from generic compressors (like gzip or zip) by explicitly modeling the spatial coherence of volumetric data through an optimal linear predictor derived from the data's covariance, rather than relying solely on statistical redundancy removal. It builds on [Predictive and Differential Coding](4393-predictive-and-differential-coding.md) by specifying the exact causal linear predictor and truncated Huffman implementation for volumes.

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

### Lorenz Predictor with Arithmetic Coding
- **Steps:** Compress the prediction error using arithmetic coding [S4, unofficial].

## Background
Predictive coding in volume rendering has been applied to various domains, including holographic displays and medical imaging. The lossless compression of hologram interference patterns using directional linear predictive coding was proposed by Kim, Tamagawa, and Sakamoto in 2019, achieving a 26.7% compression rate [S1, unofficial]. In medical imaging, JPEG-LS has been compared against 3-D/2-D MLZC, showing strong performance on ANGIO data [S2]. Time-varying volume compression strategies often combine wavelet transforms with delta encoding, as seen in methods by Ko et al. and Mensman et al., which classify frames as intra or predictive and encode deltas with respect to previous frames [S3]. Additionally, Lorenz predictors have been used in conjunction with arithmetic coding for volume data compression [S4, unofficial].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S4] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
