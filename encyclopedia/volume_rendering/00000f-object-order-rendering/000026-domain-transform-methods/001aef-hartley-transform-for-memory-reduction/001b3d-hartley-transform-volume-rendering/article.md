# Hartley Transform Volume Rendering

Also called: Discrete Hartley Transform (DHT) Volume Rendering, Fourier Volume Rendering (using Hartley), Hartley-based Volume Rendering, Hartley-based frequency domain rendering, Real-valued frequency domain rendering

## Overview
Hartley Transform Volume Rendering is a frequency-domain technique that replaces spatial domain projection, shading, and depth cueing computations with frequency domain operations using the Hartley transform to reduce memory cost and computational complexity [S3, unofficial]. It sets itself apart from standard Fourier-based frequency rendering by using real-valued kernels, which eliminates complex number arithmetic and simplifies slice extraction compared to handling Hermitian symmetry [S2][S3, unofficial]. It builds on [Hartley Transform for Memory Reduction](895-hartley-transform-for-memory-reduction.md) by providing the complete rendering pipeline, including adaptive filtering and format conversion [S3, unofficial].

## How it works
**Inputs:** 3D volume data $V(x,y,z)$ (typically padded to powers of 2, e.g., $128^3$ or $256^3$) [S3, unofficial].
**Outputs:** 2D projected image [S3, unofficial].

**Data Structures:**
*   **Volume Data:** 3D array of real values [S3, unofficial].
*   **Spectrum:** 3D array of real values representing the Hartley transform of the volume [S3, unofficial].
*   **Format:** 16-bit floating point format is used for the spectrum to reduce memory cost [S3, unofficial].

**Parameters:**
*   **Floating Point Format:** 16-bit float; 64-bit double precision is considered safe for $256^3$ volumes but consumes significantly more memory [S3, unofficial].
*   **Adaptive Filtering Threshold ($d_{max}$):** 0.3, controlling the switch between filter sizes [S3, unofficial].

**Steps:**

1.  **Pre-multiplication:** Pre-multiply the volume data before transforming to handle the wide dynamic range of spectra [S3, unofficial].
2.  **Forward Hartley Transform:** Perform the Hartley transform on the volume data to obtain the 3D spectrum $H(s_x, s_y, s_z)$ [S3, unofficial]. The Hartley transform is defined as:
    $$ H\{f(\mathbf{x})\} = \int_{-\infty}^{+\infty} f(\mathbf{x}) \text{cas}(2\pi \mathbf{s} \cdot \mathbf{x}) d\mathbf{x} $$
    where $\text{cas}(x) = \cos(x) + \sin(x)$ [S3, unofficial].
3.  **Frequency Domain Shading and Depth Cueing:** Apply shading and depth cueing in the frequency domain [S3, unofficial]. Multiple light sources are handled by shading the volume data (e.g., using gradient vectors for orientation intuition) before or during the transform process [S3, unofficial]. For example, multiple lights can be applied to create color cues for gradient orientation [S3, unofficial].
4.  **Slice Extraction:** For each frame, extract the relevant slice from the 3D spectrum corresponding to the current view angle [S3, unofficial]. This involves interpolation and resampling [S3, unofficial].
5.  **Adaptive Filtering:** Perform adaptive filtering on the extracted slice to reduce interpolation cost [S3, unofficial]. Use a $3 \times 3 \times 3$ filter and a $1 \times 1 \times 1$ filter [S3, unofficial]. The switch between filters is determined by the maximum difference in response ($d_{max}$) of 0.3 [S3, unofficial]. If the difference is below $d_{max}$, the smaller filter is used [S3, unofficial].
6.  **Inverse Hartley Transform:** Perform the inverse Hartley transform on the filtered slice to return to the spatial domain [S3, unofficial].
7.  **Format Conversion:** Convert the result from the 16-bit floating point format to the machine's native format for display [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def hartley_transform_volume_rendering(volume_data, view_angle, d_max=0.3):
    # Step 1: Pre-multiply volume data to handle wide dynamic range of spectra [S3, unofficial]
    # [G] Specific pre-multiplication factor not given in sources.
    pre_multiplied_volume = volume_data 

    # Step 2: Forward Hartley Transform to obtain 3D spectrum [S3, unofficial]
    # [G] Standard relation: H(s) = Re(F(s)) - Im(F(s)) where F is DFT [G]
    fourier_spectrum = np.fft.fftn(pre_multiplied_volume)
    spectrum = np.real(fourier_spectrum) - np.imag(fourier_spectrum)

    # Step 3: Frequency Domain Shading and Depth Cueing [S3, unofficial]
    # [G] Shading modifies spectrum based on light sources and gradient vectors [S3, unofficial]
    shaded_spectrum = apply_frequency_domain_shading(spectrum, view_angle)

    # Step 4: Slice Extraction for current view angle [S3, unofficial]
    extracted_slice = extract_slice_from_spectrum(shaded_spectrum, view_angle)

    # Step 5: Adaptive Filtering to reduce interpolation cost [S3, unofficial]
    filtered_slice = adaptive_filter(extracted_slice, d_max)

    # Step 6: Inverse Hartley Transform to return to spatial domain [S3, unofficial]
    # [G] Inverse DHT of real spectrum is equivalent to Inverse DFT real part [G]
    spatial_result = np.real(np.fft.ifftn(filtered_slice))

    # Step 7: Format Conversion to machine native format [S3, unofficial]
    final_image = convert_to_native_format(spatial_result)
    return final_image

def apply_frequency_domain_shading(spectrum, view_angle):
    # [G] Sources do not provide exact mathematical formulation for frequency domain shading.
    # Implementation applies lighting model to spectrum coefficients.
    return spectrum

def extract_slice_from_spectrum(spectrum, view_angle):
    # [G] Sources do not specify exact interpolation/resampling algorithm.
    # Implementation extracts 2D slice from 3D spectrum along view direction.
    return spectrum

def adaptive_filter(slice_data, d_max):
    # [G] Sources specify 3x3x3 and 1x1x1 filters with d_max=0.3 threshold [S3, unofficial]
    # Implementation switches filter size based on maximum response difference.
    return slice_data

def convert_to_native_format(data):
    # [G] Converts 16-bit float representation to standard float32/float64 [S3, unofficial]
    return data.astype(np.float32)
```

**Pitfalls:**
*   The algorithm does not allow non-linear attenuation due to the linear nature of the transform [S3, unofficial].
*   Using Fourier transform with Hermitian symmetry unnecessarily complicates slice extraction compared to the real-valued Hartley transform [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Spatial domain projection complexity is $O(n^3)$ [S3, unofficial]. Frequency domain projection complexity is $O(n^2 \log n)$ [S3, unofficial]. Shading and depth cueing in the frequency domain are $O(n^2)$ [S3, unofficial].
*   **Memory Complexity:** A straightforward double precision 3D spectrum implementation consumes 16 times more memory than a spatial domain algorithm (assuming 1 byte per voxel) [S3, unofficial]. Using Hartley + 16-bit float reduces this factor to 2 [S3, unofficial].
*   **Measured Performance:** On an IRIS Crimson with a 50Mhz R4000 processor, a $128^3$ volume can be rendered in a fraction of a second [S3, unofficial].
*   **Interpolation Cost:** Adaptive filtering reduces interpolation operations to ~15% of the non-adaptive case (e.g., 17.1% for Head, 12.9% for Teapot, 16.2% for Turbine) [S3, unofficial].
*   **Relative Error:** Adaptive filtering relative error is always below 40dB, where image differences are not visible [S3, unofficial].
*   **Round-off Error:** Using 16-bit float causes relative difference from 64-bit double precision images below 50dB [S3, unofficial].
*   **Strengths:** Significant speed advantages over spatial domain methods; allows shading/depth cueing to be factored out; reduced memory usage with 16-bit float; real-valued arithmetic simplifies implementation [S3, unofficial][S2].
*   **Limitations:** Non-occluding projection operator results in a loss of realism (no shadows/occlusion); does not allow non-linear attenuation due to the linear nature of the transform [S3, unofficial].
*   **When to use:** When speed is critical and occlusion/shadows are not required; when memory is a constraint and 16-bit precision is acceptable; when implementing on DSPs due to high data level parallelism [S3, unofficial].
*   **When not to use:** When realistic shadows and occlusion are required; when non-linear attenuation effects are necessary [S3, unofficial].

## Background
*   **Origin:** The method was introduced in the paper "Frequency Domain Volume Rendering" [S3, unofficial].
*   **History:** The paper demonstrates the use of the Hartley transform to replace spatial domain computations with frequency domain operations [S3, unofficial]. It highlights the benefits of real-valued arithmetic and reduced memory usage [S3, unofficial].
*   **Usage:** Experiments used Human skull (CT data, $106^3$ padded to $128^3$), Utah teapot (3D scanconverted to $256^3$), and Turbine blade ($256^3$) [S3, unofficial].
*   **Hardware:** Suggested for implementation on Digital Signal Processors (DSPs) due to high data level parallelism and convolution/FFT core computations [S3, unofficial].

## Sources
[S1] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
