# Temporal Prediction


## Overview
Temporal prediction compresses time-varying volume data by exploiting temporal coherence, assuming that voxel values or transform coefficients remain similar between consecutive time steps. It sets itself apart from spatial prediction methods (like [Spatial Predictive Coding](12623-spatial-predictive-coding.md)) by using the previous frame as the reference rather than spatial neighbors, and from transform-domain methods (like [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)) by operating directly in the spatial domain unless explicitly combined with a transform.

## Shared principles
All methods in this family share the mechanism of estimating a voxel's value $V_{pred}$ from previously processed neighbors or reference frames, computing a residual $R = V_{actual} - V_{pred}$, and encoding $R$ [S1].

1.  **Prediction:** The core operation is generating $V_{pred}$ from a causal neighborhood (spatial) or a reference frame (temporal). The predictor can be fixed (e.g., previous value, zero) or adaptive (e.g., linear combination of neighbors, optimal coefficients derived from covariance) [S1][S2, unofficial].
2.  **Residual Calculation:** The residual $R$ is computed. In lossless contexts, $R$ is exact. In lossy contexts, $R$ may be quantized. The goal is to minimize the variance of $R$, concentrating its probability mass near zero [S1].
3.  **Entropy Coding:** The residuals are encoded using entropy coding (Huffman, Arithmetic, or Range coding). Because residuals are typically small integers with a Laplacian-like distribution, they compress significantly better than raw values [S1].
4.  **Closed-Loop vs. Open-Loop:** In temporal or iterative contexts, a closed-loop structure is often required where the decoder uses its *reconstructed* value (not the original) as the predictor for the next step to prevent error accumulation [S1].

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Delta encoding is a predictive compression method for time-varying volume data that exploits temporal coherence by assuming voxel values remain constant between time steps, storing only the difference (residual) between the actual value and the predicted previous value. It distinguishes itself from sibling methods like [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md) by operating directly in the spatial-temporal domain without prior transform steps, relying on the fact that unchanged regions produce zero-valued residuals that compress efficiently into uniform bricks.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
Residual Updates in Transform Domain is a temporal compression method for volumetric data that operates in the frequency domain rather than the spatial domain. It solves the problem of compressing sequences of volume datasets (such as TSDF volumes) by exploiting temporal coherence between frames. Unlike spatial prediction methods that estimate voxel values from neighbors, this method estimates the current frame's transform coefficients from the previously reconstructed frame's coefficients. It sets itself apart from siblings like [Delta Encoding](1096-delta-encoding.md) by performing subtraction and quantization in the DCT space, which concentrates residual energy more effectively for smooth volumetric changes, thereby improving compressibility for subsequent entropy coding.

### Inter-frame Delta Encoding with Haar Wavelets
Classifying each time step as intra-coded or predictive frame. All frames are encoded with Haar wavelet, but predictive frames are also expressed as delta with respect to previous frames and then encoded [S1].

### MPEG-like Time-Variant Encoding
A compression-based algorithm by Guthe and Straßer using a lossy MPEG-like approach to encode time-variant data [S2, unofficial]. The data were then decompressed on-the-fly for display with texture mapping hardware [S2, unofficial].

### Difference encoding for time-variant volumes
A method by Shen and Johnson using difference encoding of time-variant volumes to reduce storage and rendering time [S2, unofficial].

## Background
Temporal prediction techniques have been developed to facilitate interactive rendering speeds by addressing the large data volumes of time-variant 3D datasets. Early work by Shen and Johnson [226] used difference encoding to reduce storage and rendering time [S2, unofficial]. Guthe and Straßer [74] proposed a lossy MPEG-like approach for time-variant data [S2, unofficial]. More recent approaches include Nagayasu et al. [NIH08] who pack three time steps into RGB channels for VTC compression, and Ko et al. [KLW∗ 08] who classify time steps as intra-coded or predictive frames using Haar wavelets [S1]. Mensman et al. [MRH10] performed delta encoding of block data with LZO compression [S1], while She et al. [SBN11] combined delta encoding with hierarchical vector quantization [S1].

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
