# Direction-Adaptive DWT (DA-DWT)

Also called: SD-DA-DWT (Segmentation Driven)

## Overview
Direction-Adaptive DWT (DA-DWT) is a wavelet transform technique where the direction vector $d$ changes dynamically based on spatial location to optimize energy compaction for localized directional features in volumetric data [S1]. It solves the problem of inefficient energy compaction when using fixed-direction transforms on data with varying local orientations [S1]. Unlike standard 3D DWT which uses fixed axes, DA-DWT adapts to the data's geometry, but requires signaling the chosen direction in the code-stream [S1].

## How it works
**Inputs:** Volumetric dataset (e.g., medical images).
**Outputs:** Compressed bit-stream containing wavelet coefficients and direction metadata.

**Data Structures:**
*   **DA-blocks:** Fixed-sized $N_X \times N_Y \times N_Z$ blocks into which the input is segmented [S1].
*   **Direction Index:** An integer index representing the selected direction vector from a predefined set of modulus $N_D$ vectors [S1].
*   **Prediction Error:** The difference between the predicted and actual direction index for a block [S1].

**Steps:**
1.  **Segmentation:** Divide the volumetric data into fixed-sized DA-blocks ($N_X \times N_Y \times N_Z$) [S1].
2.  **Direction Selection:** For each DA-block, select a suited direction vector $d$ from a predefined set that best matches the local directional features [S1].
3.  **Transform Application:** Apply the directional 1D-DWTD steps using the selected direction vector $d$ for the block [S1].
4.  **Direction Coding:** Encode the selected direction vector information for each block [S1]:
    a. Map the selected direction vector to an index in the set of modulus $N_D$ possible vectors [S1].
    b. Order these indexes circularly to support predictive coding [S1].
    c. Predict the direction index for the current block from its causal neighborhood [S1].
    d. Calculate the prediction error (actual index minus predicted index) [S1].
    e. Encode the prediction error using an $N_D$-symbol arithmetic coder [S1].
5.  **Coefficient Coding:** Encode the resulting wavelet coefficients using standard entropy coding (e.g., EBCOT) [S1].

**Parameters:**
*   **DA-block dimensions ($N_X, N_Y, N_Z$):** Fixed size for segmentation [S1].
*   **$N_D$:** Number of possible direction vectors in the set [S1].
*   **Bit-rate for directionality:** Ranges from 0.001 to 0.05 bits per pixel (bpp) depending on DA-block dimensions [S1].

**Edge Cases and Pitfalls:**
*   **Per-pixel selection:** Not profitable because the rate cost for encoding directional information inhibits the rate-distortion gain [S1].
*   **Signaling overhead:** The encoder must signal the direction per spatial location (or per block) for the decoder to apply the inverse transform correctly [S1].

## Subtopics

### Direction vector coding
Encodes selected direction vectors as indexes to vectors in modulus $N_D$ for each 1D-DWTD step independently [S1]. Indexes are ordered circularly to support predictive coding, where each direction index is predicted from its causal neighborhood, and the resulting prediction error is sent to an $N_D$-symbol arithmetic coder [S1]. This method is independent of the segmentation technique [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Bit-rate spent on directionality coding is extremely low, ranging from 0.001 to 0.05 bits per pixel (bpp) depending on DA-block dimensions [S1].
*   **Strengths:** Optimizes energy compaction for localized directional features; low overhead for direction signaling compared to per-pixel selection [S1].
*   **Limitations:** Requires signaling direction information; per-pixel direction selection is not profitable due to high rate cost [S1].
*   **When to use:** Volumetric medical images with anatomical symmetry and directional features [S1].
*   **When not to use:** When computational resources are extremely limited and direction signaling overhead is prohibitive relative to gain [S1].

## Background
Originated from research into wavelet-based volumetric medical image compression [S1]. The method addresses the need for adaptive directional transforms in medical imaging, where anatomical structures exhibit distinct local directionality [S1]. Various segmentation strategies were proposed, ranging from simple block-based to tree-based and content-dependent segmentations [S1].

## Sources
[S1] Wavelet based volumetric medical image compression, https://doi.org/10.1016/j.image.2014.12.007, fetched 2026-10-26
