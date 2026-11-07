# GPU-based Fast Hartley Transform (FHT)
Also called: Fast Hartley Transform, GPU-based FHT, Hartley Transform Implementation
## Overview
The GPU-based Fast Hartley Transform (FHT) is a frequency-domain volume rendering method that replaces the Fourier transform with the Hartley transform to map real-valued input data directly to real-valued spectra [S1, S2, S3, unofficial]. This reduces memory requirements by half compared to the standard GPU-based Fourier transform [S1, S2, unofficial]. It builds on [Hartley Transform for Memory Reduction](895-hartley-transform-for-memory-reduction.md) by providing a specific GPU shader implementation using double butterfly passes and dual lookup textures [S2, unofficial].

## How it works
The FHT implementation on the GPU follows the structure of the Fast Fourier Transform (FFT) but modifies the butterfly operation and data storage to handle real-valued arithmetic [S1, S2, unofficial].

### Inputs and Outputs
- **Input**: A 3D volume dataset of real-valued samples (e.g., density values) [S1, S2, unofficial].
- **Output**: A 3D volume dataset of real-valued Hartley coefficients [S1, S2, unofficial].

### Data Structures
The implementation requires two lookup textures to store the parameters for the butterfly operations, replacing the single complex-weight texture used in FFT [S1, S2, unofficial].

1.  **FHT Lookup Texture 1**:
    -   **Format**: Three-channel (RGB) texture [S1, S2, unofficial].
    -   **Content**: Stores three integer addresses (indices) of the input samples that contribute to the final butterfly result [S1, S2, unofficial].
2.  **FHT Lookup Texture 2**:
    -   **Format**: Two-channel (LUMINANCE ALPHA) texture [S1, S2, unofficial].
    -   **Content**: Stores precomputed weights for the cosine and sine terms required for the Hartley kernel [S1, S2, unofficial].

### Algorithm Steps
The transform is performed in passes. For multi-dimensional data, the separable approach is used: 1D transforms are performed along each axis, followed by a correction pass [S1, S2, unofficial].

1.  **Scrambling**: Perform the bit-reversal permutation (scrambling) exactly as in the standard FFT [S2, unofficial].
2.  **Butterfly Passes**: Perform "double butterfly" passes instead of standard FFT butterfly passes [S1, S2, unofficial].
    -   For each butterfly operation, the shader performs four texture fetches from the input data using the addresses stored in Lookup Texture 1 [S1, S2, unofficial].
    -   Let the four fetched values be $v_1, v_2, v_3, v_4$ [S1, S2, unofficial].
    -   Retrieve the precomputed cosine ($c$) and sine ($s$) weights from Lookup Texture 2 [S1, S2, unofficial].
    -   Compute the Hartley butterfly result by multiplying the input values with the trigonometric weights and summing them [S1, S2, unofficial].
    -   The shader performs four texture fetches, sums them together, and performs correction according to equations 4 and 5 in the source literature [S1, S2, unofficial].
3.  **Multi-dimensional Correction**: In the multi-dimensional case, the product of the one-dimensional transforms for each direction is not yet the correct multi-dimensional Hartley transform [S1, S2, unofficial]. A final correction pass is applied to the product of the 1D transforms to yield the correct multi-dimensional result [S1, S2, unofficial].

### Pseudocode
The following pseudocode describes the GPU fragment shader logic for a single 1D butterfly pass [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fht_butterfly_pass(input_volume, lookup_tex1, lookup_tex2, output_volume, coords):
    # Fetch three sample addresses from the 3-channel lookup texture
    idx1, idx2, idx3 = lookup_tex1[coords]
    # The source specifies four texture fetches total; the fourth index logic is not detailed in the sources [S1, S2, unofficial]
    idx4 = coords  # Fourth address derivation not provided in sources [G]
    # Fetch precomputed cosine and sine weights from the 2-channel lookup texture
    cos_w, sin_w = lookup_tex2[coords]
    # Perform four texture fetches as specified in the source
    v1 = input_volume[idx1]
    v2 = input_volume[idx2]
    v3 = input_volume[idx3]
    v4 = input_volume[idx4]
    # Multiply cos and sin terms with corresponding input values and sum them together
    # Exact coefficients and signs follow equations 4 and 5 in the source literature [S1, S2, unofficial]
    result = (v1 * cos_w) + (v2 * sin_w) + (v3 * cos_w) + (v4 * sin_w)
    output_volume[coords] = result

def fht_correction_pass(input_1d_transforms, output_3d_transform, coords):
    # Fetch 1D transform results for each dimension
    val_x = input_1d_transforms[coords[0]]
    val_y = input_1d_transforms[coords[1]]
    val_z = input_1d_transforms[coords[2]]
    # Apply correction to the product of 1D transforms
    # Exact correction formula not provided in sources; standard separable transform correction applies [G]
    result = val_x * val_y * val_z
    output_3d_transform[coords] = result
```

### Parameters and Edge Cases
-   **Lookup Textures**: Must be precomputed and uploaded to GPU memory before rendering [S1, S2, unofficial].
-   **Multi-dimensional Case**: Requires a separate correction pass after the 1D transforms [S1, S2, unofficial].
-   **Precision**: The method relies on texture fetches and floating-point arithmetic. Precision errors may accumulate in deep butterfly hierarchies [G].
-   **Boundary Conditions**: The scrambling and butterfly operations assume power-of-two dimensions for efficient bit-reversal and indexing [G].

## Operating characteristics
-   **Memory Usage**: Reduces memory requirements by half compared to FFT-based methods [S1, unofficial]. The FHT lookup structure stores five values (three addresses and two weights) across two textures, whereas the FFT stores a complex weight, addresses, and a sign in a single texture [S1, S2, unofficial].
-   **Performance**: Tested on an ATI Radeon 9800 XT with 256 MB memory [S1, S2, unofficial].
-   **Test Data**: X-mas Tree volume of size $256^3$ [S1, S2, unofficial].
-   **Resolutions**: Tested at projection slice resolutions of $256^2$ and $512^2$ pixels [S1, S2, unofficial].
-   **Interpolation**: Performance was measured with nearest neighbor, trilinear, tri-cubic, and windowed sinc (Blackman window, width 4) interpolation [S1, S2, unofficial].
-   **Strengths**: Lower memory footprint than FFT; similar computational structure to FFT allows for optimized GPU shader implementation [S1, S2, unofficial].
-   **Limitations**: Requires two lookup textures instead of one, increasing texture memory bandwidth usage [S1, S2, unofficial]. The multi-dimensional correction pass adds an extra stage [S1, S2, unofficial].

## Background
The Fast Hartley Transform (FHT) was introduced as an alternative to the Fourier transform to reduce memory requirements by half [S1, S2, unofficial]. The theoretical basis is attributed to Bracewell [Bracewell 1986] [S1, S2, unofficial]. The GPU-based implementation for volume rendering was developed by Viola et al. [S1, S2, unofficial]. The work builds on Hart's implementation of FFT on the GPU [Engel 2003] [S1, S2, unofficial].

## Sources
[S1] GPU-based Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S2] Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S3] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
