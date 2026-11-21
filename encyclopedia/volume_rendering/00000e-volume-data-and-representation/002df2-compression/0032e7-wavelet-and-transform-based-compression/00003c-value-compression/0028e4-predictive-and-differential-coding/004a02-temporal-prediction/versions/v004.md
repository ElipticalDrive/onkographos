# Temporal Prediction


## Overview
Temporal prediction compresses time-varying volume data by exploiting temporal coherence, estimating a voxel's value from previously processed reference frames rather than spatial neighbors [S1]. It distinguishes itself from sibling methods like [Spatial Predictive Coding](12623-spatial-predictive-coding.md) by operating across time steps, and from [Residual Coding for Volumetric Compression](6518-residual-coding-for-volumetric-compression.md) by its specific focus on frame-to-frame dependencies in dynamic datasets.

## Shared principles
All methods in this family share the mechanism of estimating a voxel's value $V_{pred}$ from previously processed neighbors or reference frames, computing a residual $R = V_{actual} - V_{pred}$, and encoding $R$ [S1].

1.  **Prediction:** The core operation generates $V_{pred}$ from a causal neighborhood (spatial) or a reference frame (temporal). Predictors can be fixed (e.g., previous value, zero) or adaptive (e.g., linear combination of neighbors, optimal coefficients derived from covariance) [S1].
2.  **Residual Calculation:** The residual $R = V_{actual} - V_{pred}$ is computed. In lossless contexts, $R$ is exact; in lossy contexts, $R$ may be quantized. The goal is to minimize the variance or entropy of $R$ [S1].
3.  **Entropy Coding:** The residuals are encoded using entropy coding (e.g., Huffman, Arithmetic, or Range coding) [S1].
4.  **Closed-Loop vs. Open-Loop:** In temporal or iterative contexts, a closed-loop structure is often required where the decoder uses its reconstructed value (not the original) as the predictor for the next step to prevent error accumulation [S1].

## Subtopics

### [Delta Encoding](1096-delta-encoding.md)
Operates directly in the spatial-temporal domain without prior transform steps, relying on unchanged regions producing zero-valued residuals.

### [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md)
Operates in the frequency domain (e.g., DCT), estimating current frame's transform coefficients from the previously reconstructed frame's coefficients.

### [MPEG-like Time-Variant Encoding](13596-mpeg-like-time-variant-encoding.md)
Applies lossy MPEG encoding principles to time-varying data, allowing high compression ratios and fast rendering via texture mapping hardware.

### [Difference encoding for time-variant volumes](13616-difference-encoding-for-time-variant-volumes.md)
Transforms data in the temporal domain by computing first- and second-order differences for each voxel's time series.

### Inter-frame Delta Encoding with Haar Wavelets
Classifies each time step as an intra-coded or predictive frame. All frames are encoded with Haar wavelets, but predictive frames are additionally expressed as deltas with respect to previous frames before encoding [S1].

## Background
Temporal prediction techniques have been applied to compressed GPU-based direct volume rendering to handle dynamic datasets. Early approaches included packing three time steps into RGB channels for 3D-RGB compression [NIH08] and classifying frames as intra-coded or predictive, encoding predictive frames as deltas with respect to previous frames using Haar wavelets [KLW∗ 08]. Subsequent methods performed delta encoding on block data with respect to the previous time step, quantizing blocks based on data range and encoding with LZO [MRH10], or performing delta encoding followed by hierarchical vector quantization [SBN11].

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
