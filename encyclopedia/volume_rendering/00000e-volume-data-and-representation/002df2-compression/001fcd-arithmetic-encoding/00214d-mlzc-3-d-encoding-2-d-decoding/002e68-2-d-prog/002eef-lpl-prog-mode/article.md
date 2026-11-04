# LPL-PROG Mode
Also called: Layer-Per-Layer Progression Mode

## Overview
LPL-PROG Mode is a bitstream organization scheme for the MLZC 3-D encoding/2-D decoding system that enables quality scalability for both the entire volume and individual 2-D slices [S1]. It is derived from G-PROG mode by inserting markers after every quantization layer of every subband image [S1]. It is chosen when 2-D local-scale conditioning is used and slice-level progressiveness is required, at the cost of increased bitstream overhead due to markers [S1].

## How it works
**Inputs:** Quantized subband coefficients organized by subband and image position $z$.
**Outputs:** A compressed bitstream with embedded markers.

**Data Structures:**
The bitstream consists of encoded coefficient layers interspersed with markers.
- **Marker:** A specific bit sequence inserted after the encoding of a quantization layer for a specific subband image. The exact bit pattern is not specified in the sources [S1].
- **Quantization Layer:** The set of bits representing coefficients at a specific resolution level for a specific subband and image.

**Steps:**
1. **Subband Iteration:** Iterate through all subbands in the wavelet decomposition.
2. **Image Iteration:** For each subband, iterate through all 2-D images (slices) along the axis $z$.
3. **Layer Encoding:** For each subband image, apply quantizers successively to generate quantization layers.
4. **Marker Insertion:** After encoding each quantization layer for a subband image, insert a marker into the bitstream [S1].
5. **Repeat:** Continue until all layers for all subbands and all images are encoded.

**Parameters:**
- **2-D Local-Scale Conditioning:** Required for progressiveness on individual 2-D images [S1].
- **Marker Frequency:** One marker per quantization layer per subband image [S1].

**Edge Cases and Pitfalls:**
- **Overloading:** The primary drawback is the overloading of the encoded information due to the frequent insertion of markers [S1].
- **Conditioning Constraint:** To enable 3-D encoding/2-D decoding functionalities, the system must use bi-dimensional contexts without interband conditioning [S1].

## Subtopics

### LPL Mode
LPL Mode reduces the overhead of LPL-PROG by applying the whole set of quantizers to each subband image of position $z$ before switching to the next one ($z+1$) [S1]. This eliminates the frequent markers but results in suboptimal progressiveness by quality for both single images and the whole volume, degrading lossy performance compared to G-PROG mode [S1]. Quality scalability could be improved with ad-hoc rate allocation, but this is left for future investigation [S1].

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Measured Performance:** Not given in the sources.
**Scaling:** Not given in the sources.
**Strengths:** Enables scalability on the whole volume; decoding can be stopped at any point in the bitstream [S1]. Allows progressiveness by quality on both the whole volume and any 2-D image [S1].
**Limitations:** The compression ratio is maximized, but the 3-D encoding/2-D decoding functionalities are not enabled in the base LPL mode; LPL-PROG suffers from bitstream overloading due to markers [S1].
**When to Use:** When 2-D local-scale conditioning is used and slice-level progressiveness is required.
**When Not to Use:** When minimizing bitstream overhead is critical or when 3-D encoding/2-D decoding capabilities are required without the overhead of markers (use LPL mode or other variants).

## Background
LPL-PROG Mode is part of the MLZC 3-D encoding/2-D decoding system for medical data, described in a paper by the authors of [S1] published in 2003. It is derived from G-PROG mode [S1].

## Sources
[S1] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
