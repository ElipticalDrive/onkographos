# 16-bit Floating Point Format for Spectra

Also called: 16-bit Floating Point Spectrum Representation, 1G-bit floating point format

## Overview
This method defines a custom 16-bit floating-point encoding scheme for storing frequency-domain volume spectra, specifically within the context of frequency domain volume rendering. It solves the problem of excessive memory consumption associated with representing spectral data in standard 64-bit double precision, which is traditionally required to handle the wide dynamic range of transformed volume data without precision loss. It sets itself apart from sibling compression methods like [Sparse structures](5-sparse-structures.md) or [Wavelet and Transform-based Compression](6879-wavelet-and-transform-based-compression.md) by focusing on a fixed-width, lossy quantization of the entire spectrum rather than structural sparsity or hierarchical tree representations.

## How it works
**Inputs and Outputs:**
*   **Input:** A volume spectrum (real or complex coefficients) derived from a spatial volume via a Hartley or Fourier transform.
*   **Output:** A compressed bitstream or memory buffer where each spectral coefficient is stored as a 16-bit value.

**Data Structures:**
The format is a custom 16-bit floating-point layout. While the exact bit-field allocation (sign, exponent, mantissa) is not explicitly detailed in the sources, it is defined as a "shorter floating point format" distinct from standard IEEE 754 half-precision, tailored for the specific dynamic range of spectral data [S1, S2, unofficial].

**Numbered Steps:**
1.  **Transform:** Compute the frequency domain representation of the input volume. The sources indicate the use of the Hartley transform or Fourier transform [S1, S2, unofficial].
2.  **Quantize:** Map each spectral coefficient from its original high-precision representation (e.g., 64-bit double) to the custom 16-bit floating-point format.
3.  **Store:** Write the 16-bit values to memory. The full spectrum is stored, including both positive and negative frequency components, to simplify subsequent slice extraction and rendering operations [S1, unofficial].

**Formulas and Parameters:**
*   **Memory Cost Reduction:**
    Let $V$ be the original spatial volume size in bytes (assuming 1 byte per voxel).
    Standard 64-bit double precision spectrum memory $M_{64} = 8 \times V$ [S1, S2, unofficial].
    Custom 16-bit format memory $M_{16} = 2 \times V$ [S1, S2, unofficial].
    The format reduces the memory cost factor from 8 to 2 relative to the original spatial volume size [S1, S2, unofficial].

*   **Precision Error:**
    The relative difference between images rendered using the 16-bit format and those rendered using 64-bit double precision is below 50 dB [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Range:** The wide dynamic range of spectra necessitates a floating-point format; 8-bit or integer formats are insufficient [S1, unofficial].
*   **Hermitian Symmetry:** Although the Fourier spectrum of a real signal is Hermitian (allowing storage of only positive coefficients to save memory), this implementation stores the full spectrum. Storing only positive coefficients would unnecessarily complicate the slice extraction process [S1, unofficial].
*   **Precision Loss:** While the error is small (<50 dB), it is a lossy compression method. The sources note the effect of round-off error is "very small" but do not provide a threshold for visual artifacts [S1, S2, unofficial].

## Operating characteristics
*   **Memory Complexity:** The format occupies 2 times the memory of the original spatial volume (assuming 1 byte/voxel). This is a 4x reduction compared to 64-bit double precision storage of the spectrum [S1, S2, unofficial].
*   **Performance:** The sources do not provide specific decompression times for this format alone, but note it is used in conjunction with frequency domain rendering which has $O(N^2 \log N)$ complexity compared to spatial domain $O(N^3)$ [S1, S2, unofficial].
*   **Strengths:** Significantly reduces memory footprint for frequency-domain rendering pipelines; maintains high visual fidelity (error < 50 dB).
*   **Limitations:** Lossy compression; requires custom hardware or software support for the specific 16-bit format; does not exploit sparsity.
*   **When to use:** When using frequency domain volume rendering (e.g., projection slice theorem) and memory bandwidth is a bottleneck, but full double-precision precision is not strictly required for visual correctness.
*   **When not to use:** When the rendering pipeline is spatial-domain based, or when strict numerical precision is required for scientific analysis beyond visual rendering.

## Background
The 16-bit floating point format for spectra was introduced by Takashi Totsuka and Marc Levoy in their work on Frequency Domain Volume Rendering [S1, S2, unofficial]. It was developed to address the memory overhead of storing spectral data for large volumes (e.g., $256^3$) where 64-bit double precision was considered the "safe choice" due to the need for premultiplication before transformation. The method was demonstrated on datasets such as a human skull (CT scan) and a turbine blade [S1, S2, unofficial].

## Sources
[S1] PDF FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S2] Frequency Domain Volume Rendering - studylib.net, https://studylib.net/doc/18162906/frequency-domain-volume-rendering, fetched 2026-10-27 (unofficial)
