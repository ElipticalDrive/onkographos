# GPU-based Fast Hartley Transform (FHT)
Also called: Fast Hartley Transform, GPU-based FHT, Hartley Transform
## Overview
The GPU-based Fast Hartley Transform (FHT) is a frequency-domain volume rendering method that replaces the Fourier transform with the Hartley transform to map real-valued input data directly to real-valued spectra [S1, S2, unofficial]. This eliminates the need for complex number arithmetic, reducing memory requirements by half compared to the standard GPU-based Fourier transform [S1, S2, unofficial]. It builds on [Hartley Transform for Memory Reduction](895-hartley-transform-for-memory-reduction.md) by providing a specific GPU shader implementation using double butterfly passes and dual lookup textures.

## How it works
The FHT implementation on the GPU follows the structure of the Fast Fourier Transform (FFT) but modifies the butterfly operation and data storage to handle real-valued arithmetic [S1, S2, unofficial].

### Inputs and Outputs
- **Input**: A 3D volume dataset of real-valued samples (e.g., density values).
- **Output**: A 3D volume dataset of real-valued Hartley coefficients.

### Data Structures
The implementation requires two lookup textures to store the parameters for the butterfly operations, replacing the single complex-weight texture used in FFT [S1, S2, unofficial].

1.  **FHT Lookup Texture 1**:
    -   **Format**: Three-channel (RGB) texture.
    -   **Content**: Stores three integer addresses (indices) of the input samples that contribute to the final butterfly result [S1, S2, unofficial].
2.  **FHT Lookup Texture 2**:
    -   **Format**: Two-channel (Luminance/Alpha or RG) texture.
    -   **Content**: Stores precomputed weights for the cosine and sine terms required for the Hartley kernel [S1, S2, unofficial].

### Algorithm Steps
The transform is performed in passes. For multi-dimensional data, the separable approach is used: 1D transforms are performed along each axis, followed by a correction pass [S1, S2, unofficial].

1.  **Scrambling**: Perform the bit-reversal permutation (scrambling) exactly as in the standard FFT [S2, unofficial].
2.  **Butterfly Passes**: Perform "double butterfly" passes instead of standard FFT butterfly passes [S1, S2, unofficial].
    -   For each butterfly operation, the shader performs four texture fetches from the input data using the addresses stored in Lookup Texture 1 [S1, S2, unofficial].
    -   Let the four fetched values be $v_1, v_2, v_3, v_4$.
    -   Retrieve the precomputed cosine ($c$) and sine ($s$) weights from Lookup Texture 2 [S1, S2, unofficial].
    -   Compute the Hartley butterfly result by multiplying the input values with the trigonometric weights and summing them [S1, S2, unofficial]. The specific combination corresponds to the Hartley kernel $\text{cas}(\theta) = \cos(\theta) + \sin(\theta)$ [S1, S2, unofficial].
    -   *Note*: The sources reference "equations 4 and 5" for the correction and summation logic but do not provide the explicit algebraic formulas in the text [S1, S2, unofficial]. The core operation is the summation of weighted inputs [S1, S2, unofficial].
3.  **Multi-dimensional Correction**: In the multi-dimensional case, the product of the one-dimensional transforms for each direction is not yet the correct multi-dimensional Hartley transform [S1, S2, unofficial]. A final correction pass is applied to the product of the 1D transforms to yield the correct multi-dimensional result [S1, S2, unofficial].

### Pseudocode
The following pseudocode describes the GPU fragment shader logic for a single 1D butterfly pass.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fht_butterfly_pass(input_volume, lookup_tex1, lookup_tex2, output_volume):
    """
    Performs one stage of the FHT butterfly operation.
    input_volume: 3D texture containing current volume data.
    lookup_tex1: 3-channel texture storing 3 sample addresses (indices).
    lookup_tex2: 2-channel texture storing cos/sin weights.
    output_volume: 3D texture to store results.
    """
    # For each fragment (voxel) in the output volume
    for each voxel in output_volume:
        # 1. Fetch addresses from Lookup Texture 1
        # The source states 3 addresses are stored in a 3-channel texture.
        # However, the text also states "The shader performs four texture fetches".
        # We assume the fourth address is derived or the third channel holds a flag/offset.
        # Based on "three addresses of samples... in one three-channel texture",
        # we fetch three indices. The fourth value is likely the current voxel or
        # a specific butterfly partner not explicitly indexed in the lookup texture
        # description, or the "four fetches" refers to the two pairs in the double butterfly.
        # Given the ambiguity of "four fetches" vs "three addresses", we assume
        # the standard butterfly involves two pairs.
        
        # Fetch the three indices from the RGB texture
        idx1 = lookup_tex1.read(voxel_coords).r
        idx2 = lookup_tex1.read(voxel_coords).g
        idx3 = lookup_tex1.read(voxel_coords).b
        
        # Fetch the weights from the 2-channel texture
        weight = lookup_tex2.read(voxel_coords)
        cos_val = weight.r  # or .x
        sin_val = weight.g  # or .y
        
        # Fetch the input values
        # Note: The source says "three addresses... contributing to the final butterfly result"
        # but also "four texture fetches". A standard butterfly combines two values.
        # A "double butterfly" might combine four.
        # We fetch the values corresponding to the indices.
        val1 = input_volume.read(idx1)
        val2 = input_volume.read(idx2)
        val3 = input_volume.read(idx3)
        
        # The fourth fetch is not explicitly mapped to an index in the lookup texture description.
        # In many FHT implementations, the butterfly involves x[i] and x[i+N/2].
        # If the lookup texture stores 3 addresses, the 4th might be implicit or
        # the "four fetches" includes the current output location or a specific partner.
        # Without the explicit formula from the source, we assume the operation sums
        # the weighted contributions of the fetched values.
        
        # Perform the Hartley butterfly calculation
        # The source states: "cos and sin terms are then multiplied with the corresponding 
        # input values and summed together."
        
        # Assuming a linear combination of the fetched values with the weights:
        # This is a placeholder for the exact "equations 4 and 5" which are not provided.
        # [G] Standard Hartley butterfly for real inputs often involves:
        # H[k] = sum(x[n] * cas(2*pi*k*n/N))
        # The GPU implementation approximates this via texture lookups.
        
        result = (val1 * cos_val) + (val2 * sin_val) + (val3 * cos_val) + (val4 * sin_val)
        
        # Write result
        output_volume.write(voxel_coords, result)

def fht_correction_pass(input_1d_transforms, output_3d_transform):
    """
    Corrects the product of 1D transforms to the multi-dimensional transform.
    """
    # The source states: "the last pass corrects the product of one-dimensional 
    # transforms for each direction to the multi-dimensional transform."
    # [G] This typically involves a sign correction or phase adjustment specific 
    # to the separable Hartley transform property.
    for each voxel in output_3d_transform:
        # Fetch 1D results from each dimension
        val_x = input_1d_transforms.read_x(voxel)
        val_y = input_1d_transforms.read_y(voxel)
        val_z = input_1d_transforms.read_z(voxel)
        
        # Apply correction
        # Exact formula not given in sources.
        result = val_x * val_y * val_z # Placeholder for correction logic
        
        output_3d_transform.write(voxel, result)
```

### Parameters and Edge Cases
-   **Lookup Textures**: Must be precomputed and uploaded to GPU memory before rendering [S1, S2, unofficial].
-   **Multi-dimensional Case**: Requires a separate correction pass after the 1D transforms [S1, S2, unofficial].
-   **Precision**: The method relies on texture fetches and floating-point arithmetic. Precision errors may accumulate in deep butterfly hierarchies [G].
-   **Boundary Conditions**: The scrambling and butterfly operations assume power-of-two dimensions for efficient bit-reversal and indexing [G].

## Operating characteristics
-   **Memory Usage**: Reduces memory requirements by half compared to FFT-based methods because it operates on real-valued data instead of complex-valued data [S1, S2, unofficial].
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
