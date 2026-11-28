# Data Preprocessing and Normalization


## Overview
Data Preprocessing and Normalization prepares volumetric data for compression and rendering by restricting density ranges to improve compression ratios and converting data formats [S1, unofficial][S2, unofficial]. It operates on raw data values and geometry before entropy coding or spatial subdivision occurs [G].

## Shared principles
Not given in the sources.

## Subtopics

### [Volume Normalization](1823-volume-normalization.md)
Scales a volume tensor to $[0, 1]$ while preserving constant positive masks as full foreground ($1.0$).

### [Volume Utility Functions](2677-volume-utility-functions.md)
Provides low-level operations for structural adjustments (dimensionality, voxel scale), format conversions (half-precision, normal encoding), and data integrity checks.

### [Preprocessing Pipeline](6733-preprocessing-pipeline.md)
Prepares raw volumetric data for efficient rendering by applying label banding, cropping to the body bounding box, downsampling, adjusting spacing, and optionally converting to 16-bit float before wrapping in VTK `imageData`.

### [TetSplat Pre-processing and Data Layout](7221-tetsplat-pre-processing-and-data-layout.md)
Quantizes scalar field values, packs them into a binary bitstring, and serializes hierarchical data structures (Shell and Solid) into a single binary stream for efficient storage and transmission.

### [Dataset Preprocessing (DeepLesion)](7228-dataset-preprocessing-deeplesion.md)
Applies a specific, fixed thresholding and cropping pipeline tailored to the wide dynamic range of CT data, converting raw pixel intensities to Hounsfield Units (HU) and segmenting the body from background air.

### [Geometric Normalization](12018-geometric-normalization.md)
Translates and scales the input grid to prevent screen clipping during volume rendering by centering the object at the origin and scaling it to fit within a specific display resolution.

### Density Range Restriction and Ramp Function
- **Definition**: A preprocessing function applied to volume data to restrict the range of density values, improving compression ratio by removing irrelevant data and avoiding discontinuities that generate high frequencies [S1, unofficial].
- **Step**: Apply function $T$ to each density value: if density $\le d_1$, set density $= 0$; if density $\ge d_3$, set density $= 0$; if $d_1 \le \text{density} \le d_2$, set density $= (\text{density} - d_1) \times d_2 / (d_2 - d_1)$; if $d_3 \le \text{density} \le d_4$, set density $= (d_3 - \text{density}) \times d_4 / (d_4 - d_3)$ [S1, unofficial]. Normalize models with histograms outside the $[0, 255]$ range to achieve the same output value range after applying $T$ [S1, unofficial].
- **Parameter**: $d_1, d_2, d_3, d_4$: Threshold parameters defining the density range $[d_2, d_3]$ to keep intact and the ramps $[d_1, d_2]$ and $[d_3, d_4]$ to avoid discontinuities [S1, unofficial].

### Byte/Short Conversion
- **Step**: `shortsToBytes`: For each short $v$, write $(v \gg 8)$ as high byte, then $(v)$ as low byte to output array [S2, unofficial]. `bytesToShorts`: Check `data.length` is even. Check $n = \text{data.length}/2$ equals `expectedSamples` if `expectedSamples > 0` [S2, unofficial]. For each pair of bytes, combine as `(hi << 8) | lo` to form short [S2, unofficial].
- **Pitfall**: `bytesToShorts` throws `IllegalArgumentException` if `data.length` is odd. Throws `IllegalStateException` if $n \neq \text{expectedSamples}$ and `expectedSamples > 0` [S2, unofficial].

## Background
The concept of restricting density ranges to improve compression is described in the context of volumetric medical images visualization on mobile devices, where a function $T$ is used to filter density values before compression [S1, unofficial]. Byte and short conversion routines for seismic data compression are documented in open-source repositories [S2, unofficial].

## Sources
[S1] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
[S2] https://github.com/rubensrudio/SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
