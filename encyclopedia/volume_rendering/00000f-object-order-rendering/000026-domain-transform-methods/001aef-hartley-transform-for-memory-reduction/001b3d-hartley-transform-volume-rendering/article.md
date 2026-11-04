# Hartley Transform Volume Rendering

Also called: Fourier Volume Rendering (using Hartley)

## Overview
Hartley Transform Volume Rendering is a frequency-domain technique that replaces spatial domain projection, shading, and depth cueing computations with frequency domain operations using the Hartley transform to reduce memory cost and computational complexity [S1, unofficial]. It sets itself apart from standard Fourier-based frequency rendering by using real-valued kernels, which eliminates complex number arithmetic and simplifies slice extraction compared to handling Hermitian symmetry [S1, unofficial]. It builds on [Hartley Transform for Memory Reduction](895-hartley-transform-for-memory-reduction.md) by providing the complete rendering pipeline, including adaptive filtering and format conversion.

## How it works
**Inputs:** 3D volume data $V(x,y,z)$ (typically padded to powers of 2, e.g., $128^3$ or $256^3$).
**Outputs:** 2D projected image.

**Data Structures:**
*   **Volume Data:** 3D array of real values.
*   **Spectrum:** 3D array of real values representing the Hartley transform of the volume.
*   **Format:** 16-bit floating point format is used for the spectrum to reduce memory cost [S1, unofficial].

**Parameters:**
*   **Floating Point Format:** 16-bit float. 64-bit double precision is considered safe for $256^3$ volumes but consumes significantly more memory [S1, unofficial].
*   **Adaptive Filtering Threshold ($d_{max}$):** 0.3. This value controls the switch between filter sizes [S1, unofficial].

**Steps:**

1.  **Pre-multiplication:** Pre-multiply the volume data before transforming to handle the wide dynamic range of spectra [S1, unofficial].
2.  **Forward Hartley Transform:** Perform the Hartley transform on the volume data to obtain the 3D spectrum $H(s_x, s_y, s_z)$ [S1, unofficial]. The Hartley transform is defined as:
    $$ H\{f(x)\} = F_H(s) = \int_{-\infty}^{+\infty} f(x) \text{cas}(2\pi sx) dx $$
    where $\text{cas}(x) = \cos(x) + \sin(x)$ [S1, unofficial].
3.  **Frequency Domain Shading and Depth Cueing:** Apply shading and depth cueing in the frequency domain. Multiple light sources are handled by shading the volume data (e.g., using gradient vectors for orientation intuition) before or during the transform process [S1, unofficial]. For example, multiple lights can be applied to create color cues for gradient orientation [S1, unofficial].
4.  **Slice Extraction:** For each frame, extract the relevant slice from the 3D spectrum corresponding to the current view angle. This involves interpolation and resampling [S1, unofficial].
5.  **Adaptive Filtering:** Perform adaptive filtering on the extracted slice to reduce interpolation cost. Use a $3 \times 3 \times 3$ filter and a $1 \times 1 \times 1$ filter. The switch between filters is determined by the maximum difference in response ($d_{max}$) of 0.3 [S1, unofficial]. If the difference is below $d_{max}$, the smaller filter is used [S1, unofficial].
6.  **Inverse Hartley Transform:** Perform the inverse Hartley transform on the filtered slice to return to the spatial domain [S1, unofficial].
7.  **Format Conversion:** Convert the result from the 16-bit floating point format to the machine's native format for display [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hartley_transform_volume_rendering(volume_data, view_angle, d_max=0.3):
    """
    Render a volume using the Hartley transform method.
    
    Args:
        volume_data: 3D array of real values (e.g., numpy array).
        view_angle: Parameters defining the current view angle for slice extraction.
        d_max: Threshold for adaptive filtering (default 0.3).
        
    Returns:
        2D array representing the rendered image.
    """
    # Step 1: Pre-multiply volume data to handle dynamic range
    # [G] Specific pre-multiplication factor not given in sources. 
    # Assume a standard scaling or normalization if required by the specific implementation.
    # For this implementation, we assume the volume data is already scaled appropriately 
    # or the pre-multiplication is a placeholder for any necessary dynamic range adjustment.
    pre_multiplied_volume = volume_data 

    # Step 2: Forward Hartley Transform
    # [G] Implementation of 3D Hartley Transform. 
    # H(s) = sum over x of f(x) * cas(2*pi*s*x)
    spectrum = compute_3d_hartley_transform(pre_multiplied_volume)

    # Step 3: Frequency Domain Shading and Depth Cueing
    # [G] Shading in frequency domain is complex. The source mentions shading 
    # gradient vectors for orientation intuition. 
    # This step modifies the spectrum based on light sources.
    shaded_spectrum = apply_frequency_domain_shading(spectrum, view_angle)

    # Step 4: Slice Extraction
    # Extract the relevant slice from the 3D spectrum corresponding to the view angle.
    # This involves interpolation and resampling.
    extracted_slice = extract_slice_from_spectrum(shaded_spectrum, view_angle)

    # Step 5: Adaptive Filtering
    # Use 3x3x3 and 1x1x1 filters based on d_max.
    filtered_slice = adaptive_filter(extracted_slice, d_max)

    # Step 6: Inverse Hartley Transform
    # Return to spatial domain.
    # [G] Implementation of 3D Inverse Hartley Transform.
    # f(x) = sum over s of H(s) * cas(2*pi*s*x)
    spatial_result = compute_3d_inverse_hartley_transform(filtered_slice)

    # Step 7: Format Conversion
    # Convert from 16-bit float to machine native format.
    # [G] Assuming spatial_result is in 16-bit float format.
    final_image = convert_to_native_format(spatial_result)

    return final_image

def compute_3d_hartley_transform(volume):
    """
    Compute the 3D Hartley Transform of the volume.
    """
    # [G] Placeholder for actual FFT-based or direct Hartley Transform implementation.
    # The Hartley Transform can be computed using FFTs:
    # H(s) = Re(F(s)) - Im(F(s))
    # where F(s) is the Fourier Transform.
    import numpy as np
    # Assuming volume is a numpy array
    fourier_spectrum = np.fft.fftn(volume)
    hartley_spectrum = fourier_spectrum.real - fourier_spectrum.imag
    return hartley_spectrum

def compute_3d_inverse_hartley_transform(spectrum):
    """
    Compute the 3D Inverse Hartley Transform of the spectrum.
    """
    # [G] Placeholder for actual FFT-based or direct Inverse Hartley Transform implementation.
    # The Inverse Hartley Transform is similar to the forward transform for real signals.
    import numpy as np
    # Assuming spectrum is a numpy array
    # Reconstruct Fourier Spectrum from Hartley Spectrum
    # F(s) = (H(s) + i*H(-s)) / 2 ? 
    # Actually, for real signals, H(s) = Re(F(s)) - Im(F(s))
    # And H(-s) = Re(F(s)) + Im(F(s))
    # So Re(F(s)) = (H(s) + H(-s)) / 2
    # Im(F(s)) = (H(-s) - H(s)) / 2
    # However, the inverse Hartley transform of a real spectrum is:
    # f(x) = sum H(s) * cas(2*pi*s*x)
    # Which is the same as the forward transform for real signals.
    # So we can use the same logic as forward transform but with scaling.
    # f(x) = (1/N) * H{H(s)}
    hartley_result = compute_3d_hartley_transform(spectrum)
    # Scale by 1/N where N is the total number of voxels
    n_voxels = hartley_result.size
    return hartley_result / n_voxels

def apply_frequency_domain_shading(spectrum, view_angle):
    """
    Apply shading and depth cueing in the frequency domain.
    """
    # [G] The source mentions shading gradient vectors for orientation intuition.
    # This is a high-level operation. A precise mathematical formulation is not provided.
    # This function is a placeholder for the specific shading algorithm.
    return spectrum

def extract_slice_from_spectrum(spectrum, view_angle):
    """
    Extract a slice from the 3D spectrum corresponding to the view angle.
    """
    # [G] This involves interpolation and resampling.
    # The exact interpolation method is not specified, but adaptive filtering is used later.
    # This function extracts the relevant 2D slice from the 3D spectrum.
    return spectrum

def adaptive_filter(slice_data, d_max):
    """
    Apply adaptive filtering using 3x3x3 and 1x1x1 filters.
    """
    # [G] The source mentions using a 3x3x3 and 1x1x1 filter with d_max=0.3.
    # The logic is: if the maximum difference in response is below d_max, use the smaller filter.
    # This is a placeholder for the specific filtering logic.
    return slice_data

def convert_to_native_format(data):
    """
    Convert data from 16-bit float to machine native format.
    """
    # [G] Assuming data is in 16-bit float format.
    # Convert to standard float32 or float64 for display.
    return data.astype(np.float32)
```

## Operating characteristics
*   **Time Complexity:** Spatial domain projection complexity is $O(n^3)$. Frequency domain projection complexity is $O(n^2 \log n)$. Shading and depth cueing in the frequency domain are $O(n^2)$ [S1, unofficial].
*   **Memory Complexity:** A straightforward double precision 3D spectrum implementation consumes 16 times more memory than a spatial domain algorithm (assuming 1 byte per voxel). Using Hartley + 16-bit float reduces this factor to 2 [S1, unofficial].
*   **Measured Performance:** On an IRIS Crimson with a 50Mhz R4000 processor, a $128^3$ volume can be rendered in a fraction of a second [S1, unofficial].
*   **Interpolation Cost:** Adaptive filtering reduces interpolation operations to ~15% of the non-adaptive case (e.g., 17.1% for Head, 12.9% for Teapot, 16.2% for Turbine) [S1, unofficial].
*   **Relative Error:** Adaptive filtering relative error is always below 40dB, where image differences are not visible [S1, unofficial].
*   **Round-off Error:** Using 16-bit float causes relative difference from 64-bit double precision images below 50dB [S1, unofficial].
*   **Strengths:** Significant speed advantages over spatial domain methods; allows shading/depth cueing to be factored out; reduced memory usage with 16-bit float; real-valued arithmetic simplifies implementation [S1, unofficial].
*   **Limitations:** Non-occluding projection operator results in a loss of realism (no shadows/occlusion); does not allow non-linear attenuation due to the linear nature of the transform [S1, unofficial].
*   **When to use:** When speed is critical and occlusion/shadows are not required; when memory is a constraint and 16-bit precision is acceptable; when implementing on DSPs due to high data level parallelism [S1, unofficial].
*   **When not to use:** When realistic shadows and occlusion are required; when non-linear attenuation effects are necessary [S1, unofficial].

## Background
*   **Origin:** The method was introduced in the paper "Frequency Domain Volume Rendering" [S1, unofficial].
*   **History:** The paper demonstrates the use of the Hartley transform to replace spatial domain computations with frequency domain operations. It highlights the benefits of real-valued arithmetic and reduced memory usage.
*   **Usage:** Used in experiments with Human skull (CT data, $106^3$ padded to $128^3$), Utah teapot (3D scanconverted to $256^3$), and Turbine blade ($256^3$) [S1, unofficial].
*   **Hardware:** Suggested for implementation on Digital Signal Processors (DSPs) due to high data level parallelism and convolution/FFT core computations [S1, unofficial].

## Sources
[S1] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
