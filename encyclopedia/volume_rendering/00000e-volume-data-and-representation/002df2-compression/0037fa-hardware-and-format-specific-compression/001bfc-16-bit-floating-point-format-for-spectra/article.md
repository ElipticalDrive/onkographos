# 16-bit Floating Point Format for Spectra

Also called: 16-bit Floating Point Spectrum Representation, 1G-bit floating point format

## Overview
This method defines a custom 16-bit floating point data type used to store the frequency domain spectrum of a volume during frequency-domain volume rendering [S1, S2, unofficial]. It solves the problem of excessive memory consumption associated with standard 64-bit double precision representations of spectral data, while maintaining sufficient numerical precision to avoid visible artifacts in the final rendered image [S1, S2, unofficial]. It is distinct from sibling methods like [Color Packing (RGBA)](5142-color-packing-rgba.md) which target spatial domain voxel storage, and [ELSCINT1/PMSCT_RLE1 Delta Decoding](11897-elscint1-pmsct-rle1-delta-decoding.md) which targets spatial domain compression via run-length encoding [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A complex-valued frequency domain spectrum (or real-valued Hartley spectrum) derived from a spatial volume.
*   **Output:** A memory buffer containing the spectrum values packed into the custom 16-bit format.

**Data Structures:**
The format is a 16-bit floating point representation. The exact bit-field layout (sign, exponent, mantissa) is not specified in the sources [G]. The data is stored as a contiguous array of 16-bit words.

**Parameters:**
*   **Format Width:** 16 bits [S1, S2, unofficial].
*   **Precision Requirement:** The format must support the wide dynamic range of spectral coefficients [S1, S2, unofficial].

**Steps:**
1.  Compute the frequency domain representation of the volume data (e.g., using the Hartley transform or Fourier transform) [S1, S2, unofficial].
2.  For each coefficient in the spectrum, convert the 64-bit double precision value to the custom 16-bit floating point format [S1, S2, unofficial].
3.  Store the resulting 16-bit values in the output buffer [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Range:** The spectrum exhibits a wide dynamic range, making fixed-point (8-bit or integer) formats insufficient [S1, S2, unofficial].
*   **Hermitian Symmetry:** While the Fourier spectrum of a real signal is Hermitian symmetric, allowing for storage of only positive coefficients to save memory, this implementation stores the full spectrum to avoid complicating the slice extraction process [S1, unofficial].
*   **Precision Loss:** The use of 16-bit format introduces round-off errors, but these are negligible for rendering purposes [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_spectrum_to_16bit(spectrum_64bit):
    """
    Converts a list of 64-bit double precision complex numbers
    to a list of 16-bit floating point values.
    
    Note: The exact bit-packing logic for the custom 16-bit format
    is not defined in the sources. This function assumes a hypothetical
    conversion function exists.
    """
    spectrum_16bit = []
    for value in spectrum_64bit:
        # Convert 64-bit double to custom 16-bit float
        # The specific rounding and bit-layout rules are not provided
        # in the source text [S1, S2, unofficial].
        value_16 = custom_float16_from_float64(value)
        spectrum_16bit.append(value_16)
    return spectrum_16bit

def custom_float16_from_float64(val):
    """
    Placeholder for the custom 16-bit floating point conversion.
    The sources do not specify the exponent/mantissa bit allocation.
    """
    # Implementation details not given in the sources.
    # [G] In a real implementation, this would involve extracting
    # sign, exponent, and mantissa bits from the 64-bit float
    # and re-pack them into the 16-bit structure defined by the authors.
    pass
```

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The 16-bit format reduces the memory cost factor to two compared to the original spatial volume size (assuming 1 byte per voxel in the spatial domain) [S1, unofficial]. In contrast, a 64-bit double precision format would occupy 8 times more memory than the original spatial volume [S1, S2, unofficial].
*   **Computation:** The frequency domain method's running time grows at $O(n^2 \log n)$, significantly slower growth than the spatial domain method's $O(n^3)$ [S1, S2, unofficial].

**Measured Performance:**
*   **Precision:** The relative difference between images rendered using the 16-bit format and those rendered using 64-bit double precision is below 50 dB [S1, S2, unofficial].
*   **Hardware/Resolution:** Tested on datasets including a human skull (padded to $128^3$) and a turbine blade ($256^3$) [S1, S2, unofficial].
*   **Rendering Time:** For a $256^3$ turbine blade dataset, the frequency domain method took 1.77 seconds (with adaptive filtering) compared to 24.29 seconds for the spatial domain method [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant memory reduction compared to double precision; negligible impact on visual quality (below 50 dB error); enables efficient frequency-domain rendering.
*   **Limitations:** Requires a custom floating point format implementation; does not exploit Hermitian symmetry to further reduce memory (to keep slice extraction simple) [S1, unofficial].

## Background
**Origin:**
The method was introduced by Takashi Totsuka and Marc Levoy in their work on Frequency Domain Volume Rendering [S1, S2, unofficial].

**History and Development:**
The technique was developed to address the memory overhead of storing frequency domain data for volume rendering. The authors noted that while 64-bit double precision is a "safe choice" for representing spectra, it is memory-intensive. The 16-bit format was defined and used to minimize this problem while maintaining acceptable precision [S1, S2, unofficial].

**Usage:**
It is used in frequency domain volume rendering pipelines, specifically in conjunction with the Fourier projection slice theorem, adaptive filtering, and shading techniques [S1, S2, unofficial].

## Sources
[S1] FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S2] Frequency Domain Volume Rendering - studylib.net, https://studylib.net/doc/18162906/frequency-domain-volume-rendering, fetched 2026-10-27 (unofficial)
