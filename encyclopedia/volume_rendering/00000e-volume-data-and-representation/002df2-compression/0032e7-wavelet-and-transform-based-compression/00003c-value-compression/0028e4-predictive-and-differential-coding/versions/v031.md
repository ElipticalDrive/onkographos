# Predictive and Differential Coding


## Overview
Predictive and Differential Coding reduces data redundancy by estimating a sample's value from previously processed neighbors or reference frames, computing the difference (residual) between the actual and predicted values, and encoding only that residual [G]. Unlike transform-based methods that decorrelate data globally, this family operates locally or temporally to exploit spatial or temporal coherence directly [G], setting itself apart from its siblings by focusing on the prediction mechanism itself—whether spatial, temporal, or directional—rather than the transform domain or entropy coding scheme [G].

## Shared principles
All methods in this family share the mechanism of estimating a sample's value $V_{pred}$ from previously processed neighbors or reference frames, computing a residual $R = V_{actual} - V_{pred}$, and encoding $R$ [G].

1. **Prediction:** The core operation generates $V_{pred}$ from a causal neighborhood (spatial) or a reference frame (temporal) [G].
2. **Residual Calculation:** The residual $R = V_{actual} - V_{pred}$ is computed. In lossless contexts, $R$ is exact; in lossy contexts, $R$ may be quantized [G].
3. **Entropy Coding:** The residuals are encoded using entropy coding (e.g., Huffman, Arithmetic, or Range coding) [G].
4. **Closed-Loop vs. Open-Loop:** In temporal or iterative contexts, a closed-loop structure is often required where the decoder uses its reconstructed value (not the original) as the predictor for the next step to prevent error accumulation [G].

## Subtopics

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
Residual coding improves compression performance by encoding the difference between the original image and the direct reconstruction from the volume without quantization or coding [6518-residual-coding-for-volumetric-compression.md]. Unlike standard video codecs where residuals depend on the quantization level, here residuals are derived from the unquantized volume reconstruction, making them similar across all bitrates [6518-residual-coding-for-volumetric-compression.md]. It is chosen when view synthesis quality at high bitrates is critical, though it may deteriorate view synthesis quality at lower bitrates [6518-residual-coding-for-volumetric-compression.md].

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
The Lorenz Predictor is a lossless method for $n$-dimensional scalar fields that estimates a sample's value from its $2^n - 1$ previously visited neighbors in a regular grid [8070-lorenz-predictor-with-arithmetic-coding.md]. It is distinguished by being isotropic, exploiting data coherence in all dimensions, and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$ [8070-lorenz-predictor-with-arithmetic-coding.md]. It is chosen for high-dimensional data where anisotropic scanline predictors fail to capture full coherence [8070-lorenz-predictor-with-arithmetic-coding.md].

### [RDLS-SS-DWT+Pred](11251-rdls-ss-dwt-pred.md)
RDLS-SS-DWT+Pred is a lossless method that combines 3D-DWT with reversible denoising lifting steps (RDLS), step skipping (SS), and predictive coding [11251-rdls-ss-dwt-pred.md]. It solves the problem of high entropy in wavelet subbands by applying spatial prediction to the final subbands of the RDLS-SS-DWT transform [11251-rdls-ss-dwt-pred.md]. It sets itself apart by integrating prediction specifically into the post-processing stage of a modified DWT, and from its sibling SS-DWT+Pred by including the RDLS denoising stage, although sources note that RDLS offers negligible bitrate improvement over SS-DWT+Pred alone [11251-rdls-ss-dwt-pred.md].

### [Spatial Predictive Coding](12623-spatial-predictive-coding.md)
Spatial Predictive Coding reduces redundancy in voxel grids by exploiting statistical dependence between spatially adjacent samples, analogous to adjacent pixel correlation in images [12623-spatial-predictive-coding.md]. It sets itself apart by constructing a reference graph to select the optimal neighbor via Sum of Absolute Differences (SAD) and by performing quantization *before* prediction to improve parallelism [12623-spatial-predictive-coding.md]. Unlike methods that predict density, this approach targets feature grids (e.g., color) where storage costs are significantly higher [12623-spatial-predictive-coding.md].

### [Temporal Prediction](12644-temporal-prediction.md)
Temporal prediction compresses time-varying volume data by exploiting temporal coherence, estimating a voxel's value from previously processed reference frames rather than spatial neighbors [12644-temporal-prediction.md]. It sets itself apart from spatial predictive methods by operating across time steps, often requiring closed-loop reconstruction to prevent error accumulation between encoder and decoder [12644-temporal-prediction.md].

### [DPCM with Huffman Coding](16225-dpcm-with-huffman-coding.md)
DPCM with Huffman Coding applies Differential Pulse Code Modulation to generate residuals, which are then encoded using Huffman coding. This method is a classic approach for lossless compression of volumetric data where the residual distribution is highly skewed, allowing Huffman coding to achieve near-optimal compression for the predicted differences. [G]

#### Directional Linear Predictive Coding
- **Definition:** Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
- **Steps:** Segment interference patterns into blocks. Scan the pixels in eight directions for each block [S1, unofficial]. Calculate entropy in each direction. Determine the direction with minimum entropy [S1, unofficial]. Encode the difference by DPCM in the determined direction [S1, unofficial]. Calculate prediction coefficients for the determined direction [S1, unofficial]. Use DPCM only to determine the prediction direction for the original pixel [S1, unofficial]. Calculate the difference between the predicted and the original pixel [S1, unofficial].
- **Performance:** The first process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

#### JPEG-LS
- **Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
- **Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

#### Functional Representation (Spherical Ellipsoids)
- **Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding has its roots in early telecommunications and image compression, where Differential Pulse Code Modulation (DPCM) was used to reduce the dynamic range of signals before quantization [G]. In volume rendering, it was adapted to handle the large spatial redundancy in 3D scalar fields [G]. The specific application to holographic interference patterns was proposed by Hyun et al. in 2019, demonstrating the efficacy of directional linear prediction for such data [S1, unofficial]. JPEG-LS, a standard for lossless medical imaging, has been evaluated for 3D volumes, showing strong performance on specific datasets like ANGIO [S2]. More recent approaches integrate prediction into wavelet-based frameworks (e.g., RDLS-SS-DWT+Pred) to handle the high entropy of detail subbands [11251-rdls-ss-dwt-pred.md].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
