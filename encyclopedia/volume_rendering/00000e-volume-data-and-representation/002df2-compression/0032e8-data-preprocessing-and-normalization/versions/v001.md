# Data Preprocessing and Normalization

Also called: Volume Preprocessing, Volume Normalization, Volume Utility Functions

## Overview
Data Preprocessing and Normalization prepares volumetric data for compression and rendering by ensuring format compatibility, correcting metadata errors, and mapping value ranges to standardized intervals. It differs from sibling methods like [Sparse structures](5-sparse-structures.md) or [Value compression](6-value-compression.md) by operating on the data layout and physical units rather than reducing bit-depth or changing topology [S1, unofficial][2677-volume-utility-functions.md].

## Shared principles
This family shares no specific algorithmic mechanism beyond the general goal of preparing data for the [Compression](4-compression.md) pipeline. Each child method addresses a distinct aspect of data integrity or range mapping.

## Subtopics

### [Volume Normalization](1823-volume-normalization.md)
Volume Normalization scales a volume tensor to the range $[0, 1]$ while ensuring constant positive masks are preserved as full foreground ($1.0$). It differs from general value compression by focusing on range mapping rather than bit-depth reduction, and from sparse structures by operating on dense data layouts.

### [Volume Utility Functions](2677-volume-utility-functions.md)
Volume Utility Functions are a collection of low-level helper routines used to prepare, validate, and transform volumetric data for compression and rendering pipelines. They solve problems related to tensor dimensionality alignment, physical unit normalization (Hounsfield units), spatial resampling, and memory layout corrections. Unlike [Sparse structures](5-sparse-structures.md) which organize data topology, or [Value compression](6-value-compression.md) which reduces bit-depth, these utilities ensure data integrity and format compatibility before compression or rendering occurs. They serve as the foundational preprocessing and postprocessing steps for the [Compression](4-compression.md) pipeline.

### Density Range Restriction and Ramp Function
#### definition
- [S1, unofficial] A preprocessing function applied to volume data to restrict the range of density values, improving compression ratio by removing irrelevant data and avoiding discontinuities that generate high frequencies.
#### step
- [S1, unofficial] Apply function $T$ to each density value: if density $\le d_1$, set density $= 0$; if density $\ge d_4$, set density $= 0$; if $d_1 \le$ density $\le d_2$, set density $= (\text{density} - d_1) \times d_2 / (d_2 - d_1)$; if $d_3 \le$ density $\le d_4$, set density $= (d_3 - \text{density}) \times d_4 / (d_4 - d_3)$.
- [S1, unofficial] Normalize models with histograms outside the $[0, 255]$ range to achieve the same output value range after applying $T$.
#### parameter
- [S1, unofficial] $d_1, d_2, d_3, d_4$: Threshold parameters defining the density range $[d_2, d_3]$ to keep intact and the ramps $[d_1, d_2]$ and $[d_3, d_4]$ to avoid discontinuities.

## Background
The concept of restricting density ranges to improve compression ratios was introduced in the context of volumetric medical images visualization on mobile devices [S1, unofficial]. The specific utility functions for Hounsfield unit normalization and tensor manipulation are derived from modern deep learning and graphics pipelines [2677-volume-utility-functions.md].

## Sources
[S1] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
