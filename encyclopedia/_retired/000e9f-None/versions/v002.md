# 16-bit Floating Point Format for Spectra

Also called: 16-bit Floating Point Spectrum Representation, 1G-bit floating point format

## Overview
This method defines a custom 16-bit floating point format to represent the frequency domain spectrum in volume rendering, reducing memory cost compared to 64-bit double precision while maintaining acceptable precision [S1, S2, unofficial]. It is specifically designed for frequency domain rendering pipelines (such as those using the Hartley transform) where the dynamic range of spectral coefficients necessitates floating point representation, distinguishing it from integer or fixed-point compression methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A complex-valued (or real-valued Hartley) frequency domain spectrum derived from a spatial volume.
*   **Output:** A compressed bitstream or memory buffer where each spectral coefficient is stored as a 16-bit value.

**Data Structures:**
The format is a custom 16-bit floating point representation. The exact bit-field layout (sign, exponent, mantissa) is not specified in the sources [G: Standard IEEE 754-like or custom half-precision layout is implied but not defined]. Each voxel in the frequency domain is represented by exactly 16 bits.

**Steps:**
1.  Compute the frequency domain representation of the volume data (e.g., using the Hartley transform or Fourier transform) [S1, S2, unofficial].
2.  For each coefficient in the spectrum, convert the 64-bit double precision value to the custom 16-bit floating point format [S1, S2, unofficial].
3.  Store the resulting 16-bit values in memory [S1, S2, unofficial].

**Parameters:**
*   **Format Width:** 16 bits [S1, S2, unofficial].
*   **Precision:** The relative difference between images rendered using this format and those rendered using 64-bit double precision is below 50 dB [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Range:** The wide dynamic range of spectra makes 8-bit or integer formats insufficient; floating point is necessary [S1, S2, unofficial].
*   **Hermitian Symmetry:** While the Fourier spectrum of a real signal is Hermitian, allowing storage of only positive coefficients to save memory, this approach is not used here because it unnecessarily complicates the slice extraction process [S1, unofficial]. The full spectrum is stored [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory Cost:** The 16-bit format occupies 8 times more memory than the original spatial volume (assuming 1 byte per voxel in the spatial domain) [S1, S2, unofficial]. In contrast, using 64-bit double precision would occupy 16 times the original volume size [S1, S2, unofficial]. Thus, the 16-bit format reduces the memory cost factor to two compared to double precision [S1, S2, unofficial].
*   **Computation Time:** The use of frequency domain methods (including this format) allows the rendering computation to grow at $O(n^2 \log n)$ or similar, much slower than the $O(n^3)$ growth of spatial domain methods [S1, S2, unofficial].

**Measured Performance:**
*   Tests on datasets including a human head (128³), a teapot (256³), and a turbine blade (256³) showed that the round-off error effect was very small [S1, S2, unofficial].
*   Relative difference from images generated using 64-bit double precision representation was below 50 dB [S1, S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant memory reduction compared to double precision (factor of 2) while maintaining high visual fidelity (50 dB difference) [S1, S2, unofficial].
*   **Limitations:** Still requires 8x the memory of the original spatial data [S1, S2, unofficial]. The custom format is not a standard hardware format, potentially requiring software conversion or specific hardware support [G].

## Background
This format was defined and used by Takashi Totsuka and Marc Levoy in their work on frequency domain volume rendering [S1, S2, unofficial]. It was developed to address the memory overhead of representing frequency domain data, which is significantly larger than the spatial domain data due to the transform and the need for high precision [S1, S2, unofficial]. The method was applied to render CT scans of human skulls and other datasets [S1, S2, unofficial].

## Sources
[S1] FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S2] Frequency Domain Volume Rendering - studylib.net, https://studylib.net/doc/18162906/frequency-domain-volume-rendering, fetched 2026-10-27 (unofficial)
