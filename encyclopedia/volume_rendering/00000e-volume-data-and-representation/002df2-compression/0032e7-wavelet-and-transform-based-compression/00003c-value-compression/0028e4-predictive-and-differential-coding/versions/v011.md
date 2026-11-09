# Predictive and Differential Coding

Also called: Predictive Coding, Differential Pulse Code Modulation (DPCM), Residual Coding

## Overview
Predictive and Differential Coding compresses volumetric data by encoding the difference (residual) between a predicted value and the actual value, rather than the absolute value itself [S1, unofficial]. This method solves the problem of redundant data in time-varying volumes or smoothly varying spatial fields by converting uniform regions into zero-valored or small residuals, which are highly compressible via entropy coding [S1, unofficial]. It sets itself apart from transform-based methods (like [Wavelet and Transform-based Compression](828-wavelet-and-transform-based-compression.md)) by operating directly on the signal domain (spatial or temporal) using local neighbor correlations, rather than decomposing the volume into global frequency coefficients.

## Shared principles
All methods in this family share the mechanism of estimating a current sample value $\hat{v}$ from previously processed neighbors or reference frames, computing the residual $d = v - \hat{v}$, and encoding $d$ [S1, unofficial].

1.  **Prediction Model:** The predictor $\hat{v}$ is derived from a set of reference values. These references can be:
    *   **Spatial:** Neighboring voxels in a fixed scan order (e.g., previous voxel in raster scan, or face-sharing neighbors in 3D) [S1, unofficial][S3].
    *   **Temporal:** Reconstructed coefficients or voxel values from the previous time step $t-1$ [S1, unofficial].
    *   **Transform-Domain:** Coefficients from a previous frame in a transform basis (e.g., DCT) [S1, unofficial].
2.  **Residual Generation:** The residual $d$ represents the error of the prediction. If the prediction is accurate (high coherence), $d$ will have low variance or be zero, leading to high compression ratios [S1, unofficial].
3.  **Closed-Loop Reconstruction:** To prevent encoder-decoder drift in lossy scenarios, the decoder must use the *reconstructed* (dequantized) values for prediction, not the raw original values [Parent]. The encoder must mirror this process exactly [S1, unofficial].
4.  **Entropy Coding:** The residuals are encoded using entropy coding (e.g., Huffman, Arithmetic, Run-Length) [S1, unofficial].

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding is the simplest form of predictive coding, using the immediately preceding value in the processing order as the predictor. It is optimal for data with strong local coherence (e.g., heightfields or time-varying volumes with small changes) but lacks random access capabilities.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
This method applies predictive coding in the frequency domain. Instead of predicting voxel values, it predicts the transform coefficients (e.g., DCT) of the current frame from the reconstructed coefficients of the previous frame. It sets itself apart by concentrating residual energy near regions of surface change in the transform domain, improving compressibility for subsequent entropy coding compared to spatial predictors.

### [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md)
This technique encodes the difference between the original view images and the direct reconstruction of the volume data (without quantization/coding artifacts). It is used to improve view synthesis quality at high bitrates but does not contribute to the base volume reconstruction quality and may degrade view synthesis performance if not carefully managed.

### [Optimal Linear Volumetric Predictor with Truncated Huffman Coding](7759-optimal-linear-volumetric-predictor-with-truncated-huffman-c.md)
A lossless method that derives optimal linear predictor coefficients for each volume using the Yule-Walker equations, compensating for non-isotropic voxel sampling. It uses a 3-coefficient predictor based on the 6-neighborhood (face-sharing) and encodes residuals with a truncated Huffman code.

### [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md)
An isotropic predictor for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid. It is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$.

### Directional Linear Predictive Coding
*   **Definition:** Compresses hologram interference patterns losslessly by exploiting directional correlation using DPCM and linear prediction [S1, unofficial].
*   **Steps:**
    1.  Segment interference patterns into blocks [S1, unofficial].
    2.  Scan the pixels in eight directions for each block [S1, unofficial].
    3.  Calculate entropy in each direction [S1, unofficial].
    4.  Determine the direction with minimum entropy [S1, unofficial].
    5.  Encode the difference by DPCM in the determined direction [S1, unofficial].
    6.  Calculate prediction coefficients for the determined direction [S1, unofficial].
    7.  Use DPCM only to determine the prediction direction for the original pixel [S1, unofficial].
    8.  Calculate the difference between the predicted and the original pixel [S1, unofficial].
*   **Performance:** The first process (directional DPCM) achieved a compression rate of 26.7% [S1, unofficial].

### JPEG-LS
*   **Definition:** An older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S2].
*   **Comparison:** Provides the best compression for ANGIO data but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S2].

### Inter-frame Delta Encoding with Haar Wavelets
*   **Definition:** Classifying each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S3].

### Functional Representation (Spherical Ellipsoids)
*   **Definition:** An approach similar to inter-frame delta encoding but encoding data with a functional representation based on spherical ellipsoids [S3].

## Background
Predictive coding has been a fundamental technique in signal compression since the advent of DPCM. In the context of volume rendering, it was adapted to handle the large data volumes of medical and scientific datasets. Early applications focused on spatial redundancy in static volumes, leading to methods like the optimal linear predictor [S1, unofficial]. With the rise of time-varying volumes (4D data), predictive coding extended to the temporal domain, using inter-frame deltas [S3]. Transform-domain predictive coding emerged as a hybrid approach, combining the decorrelation power of transforms (like DCT or Wavelets) with the efficiency of differential coding for temporal redundancy [S1, unofficial]. The method is widely used in medical imaging standards (e.g., JPEG-LS comparisons) and specialized applications like holographic display of volume data [S1, unofficial][S2].

## Sources
[S1] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
