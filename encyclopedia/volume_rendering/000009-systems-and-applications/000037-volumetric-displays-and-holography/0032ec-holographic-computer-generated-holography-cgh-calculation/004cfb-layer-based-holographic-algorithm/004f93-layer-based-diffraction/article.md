# Layer-based diffraction

Also called: FFT-based holographic computation, Layer-based CGH Calculation, Layer-based hologram generation, Layer-based holographic rendering method, Layer-based method

## Overview
Layer-based diffraction computes left and right holograms by numerically propagating light waves from compensated depth layers towards the hologram plane and summing them [S1]. It generates high-quality holographic patterns for a Spatial Light Modulator (SLM) from RGB-D 3D scene data by computing optical field distributions on the eye lens plane using 2D Inverse Fast Fourier Transforms (IFFTs) [S2]. It is distinguished from sibling methods by its specific reliance on 2D IFFTs for both the optical field distribution and hologram pattern computations, enabling high-resolution binocular color holography on hardware accelerators [S2].

## How it works
**Inputs and Outputs**
*   **Input:** RGB-D 3D image data representing the 3D image projected onto the viewer's retina plane [S2]. The data format consists of Red, Green, Blue, and Depth channels [S2].
*   **Output:** A single combined hologram pattern for the SLM, derived from simultaneously processed left and right eye images [S2].

**Data Structures**
*   **Complex Numbers:** Stored in memory using exactly 32 bits [S2].
*   **Number Representation:** Fixed-point number representation is used for all data processing and IFFT multiplications [S2].
*   **Hologram Levels:** The amplitude-only hologram is quantized onto $L=256$ discrete levels [S1].

**Steps**
1.  **Receive Input:** Obtain the RGB-D 3D image data from rendering software or a 3D camera [S2].
2.  **Compute Optical Field:** Compute the optical field distribution on the eye lens plane from the retina plane image [S2].
3.  **Compute Hologram Patterns:** Compute the holographic patterns to be applied to the SLM from the eye lens plane distributions [S2].
4.  **Execute 2D IFFTs:** Perform 2D IFFT computations for the field distribution and pattern computations [S2]. The 2D IFFTs are computed using the row-column method, where 1-D IFFTs are performed on rows and then columns [S2].
5.  **Process Binocular Images:** Process right and left eye images simultaneously [S2].
6.  **Combine:** Combine the two eye images into a single hologram [S2].
7.  **Quantize:** Obtain the amplitude-only hologram $H$ by shifting, scaling, and quantizing the real part of the complex-valued object wave $O$ onto $L=256$ discrete levels [S1].

**Formulas**
The complex-valued object wave $O(x, y; c)$ is the sum of propagated waves from $N_z$ compensated depth layers:
$$
O(x, y; c) = \sum_{i=0}^{N_z-1} P_{\hat{z}_i} \{ U_i \} (x, y; c)
$$
where $c$ is the color channel [S1].

The propagation of a layer $U_i$ from depth $\hat{z}_i$ is given by the Fourier transform:
$$
P_{\hat{z}_i} \{ U_i \} = \frac{\exp(j \lambda_c \hat{z}_i)}{j \lambda_c \hat{z}_i} \mathcal{F} \left\{ U_i(\xi, \eta; c) \exp\left( \frac{j \pi (\xi^2 + \eta^2)}{\lambda_c \hat{z}_i} \right) \exp\left( -\frac{j \pi (x^2 + y^2)}{\lambda_c \hat{z}_i} \right) \right\}
$$
where:
*   $\lambda_c$ is the wavelength of color channel $c$ [S1].
*   $\mathcal{F}$ is the forward Fourier transform [S1].
*   $\hat{z}_i$ is the intermediate compensated layer depth [S1].
*   $U_i(\xi, \eta; c)$ is the field at the layer [S1].

The final amplitude-only hologram $H$ is:
$$
H(x, y; c) = Q(\Re \{ O(x, y; c) \}, L, \Delta)
$$
where $\Re \{ O \}$ is the real part of the object wave, $L=256$ is the number of discrete levels, and $Q$ represents the quantization operation [S1].

**Parameters**
*   **Output Resolution:** 3840 × 2160 pixels [S2].
*   **Refresh Rate:** 30 frames per second (fps) [S2].
*   **Quantization Levels:** $L = 256$ [S1].

**Pitfalls**
*   **Hardware Complexity:** FFT-based layer methods are traditionally unsuitable for CGH hardware accelerators because large-scale FFT processors are difficult to build [S2]. This implementation overcomes that using a highly parallel architecture and hardware-reducing techniques [S2].
*   **Precision:** Fixed-point arithmetic is used to minimize bit depth, requiring careful management of precision to avoid artifacts in the IFFT multiplications [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def layer_based_diffraction(rgb_d_data, retina_plane_image, eye_lens_params, slm_params):
    """
    Computes hologram from RGB-D data using layer-based diffraction.
    """
    # Step 1: Receive Input (RGB-D format) [S2]
    # Step 2 & 3: Compute optical field on eye lens plane and SLM patterns [S2]
    # Step 5: Process right and left eyes simultaneously [S2]
    right_field = compute_eye_lens_field(retina_plane_image, eye_lens_params, eye='right')
    right_pattern = compute_slm_pattern(right_field, slm_params)

    left_field = compute_eye_lens_field(retina_plane_image, eye_lens_params, eye='left')
    left_pattern = compute_slm_pattern(left_field, slm_params)

    # Step 6: Combine into single hologram [S2]
    combined_hologram = combine_holograms(right_pattern, left_pattern)

    # Step 7: Quantize to L=256 levels [S1]
    final_hologram = quantize_hologram(combined_hologram, levels=256)
    return final_hologram

def compute_eye_lens_field(retina_image, params, eye):
    # Initialize complex field sum [S1]
    field_sum = 0.0 + 0.0j
    N_z = params.num_layers

    for i in range(N_z):
        # Extract layer data Ui(xi, eta) for depth z_hat_i [S1]
        U_i = get_layer_data(retina_image, params.z_hat[i])
        
        # Propagate layer using diffraction formula [S1]
        propagated = propagate_layer(U_i, params.lambda_c, params.z_hat[i])
        field_sum += propagated

    return field_sum

def propagate_layer(U_i, lambda_c, z_hat_i):
    # Pre-computation phase factor [S1]
    phase_factor = math.exp(1j * lambda_c * z_hat_i) / (1j * lambda_c * z_hat_i)
    
    # Apply quadratic phase modulation to source coordinates (xi, eta) [S1]
    # The evidence formula includes a term with (x^2+y^2) inside the transform argument [S1]
    modulated = U_i * math.exp(1j * math.pi * (U_i.coords_x**2 + U_i.coords_y**2) / (lambda_c * z_hat_i))
    modulated *= math.exp(-1j * math.pi * (U_i.coords_x**2 + U_i.coords_y**2) / (lambda_c * z_hat_i))
    
    # Execute 2D IFFT/FFT as per row-column method [S2]
    fft_result = forward_fourier_transform(modulated)
    
    return phase_factor * fft_result

def forward_fourier_transform(data):
    # Standard 2D Discrete Fourier Transform (row-column method) [S2]
    row_transformed = [fft1d(row) for row in data]
    col_transformed = [fft1d(col) for col in zip(*row_transformed)]
    return col_transformed

def fft1d(signal):
    N = len(signal)
    result = [0.0 + 0.0j] * N
    for k in range(N):
        for n in range(N):
            result[k] += signal[n] * math.exp(-2j * math.pi * k * n / N)
    return result

def get_layer_data(retina_image, depth):
    # Extract pixels matching the compensated layer depth z_hat_i [S1]
    return retina_image.filter_by_depth(depth)

def compute_slm_pattern(lens_field, slm_params):
    # FSU performs filtering, scaling, and pixel encoding [S2]
    # Pixel encoding converts complex number to integer using fixed-point representation [S2]
    encoded = pixel_encode_fixed_point(lens_field)
    return encoded

def pixel_encode_fixed_point(complex_val):
    # Fixed-point representation uses 32 bits for complex numbers [S2]
    # Convert real and imaginary parts to fixed-point integers
    real_fixed = int(complex_val.real * (2**15)) & 0xFFFF
    imag_fixed = int(complex_val.imag * (2**15)) & 0xFFFF
    return (real_fixed << 16) | imag_fixed

def combine_holograms(right_pattern, left_pattern):
    # Combine right and left eye images into a single hologram [S2]
    return right_pattern + left_pattern

def quantize_hologram(hologram, levels=256):
    # Shift, scale, and quantize the real part of O onto L discrete levels [S1]
    real_part = [[row.real for row in col] for col in hologram]
    min_val = min(min(row) for row in real_part)
    max_val = max(max(row) for row in real_part)
    scale = (levels - 1) / (max_val - min_val) if max_val != min_val else 1.0
    quantized = [[int((val - min_val) * scale) for val in row] for row in real_part]
    return quantized
```

## Subtopics

### Fresnel Holography / Wavefront Propagation
CGH algorithms that process the light field using Fresnel or other wavefront propagation methods [S3]. Unlike holographic stereograms, these can be applied globally or within hogels [S3]. Requires different propagation kernels for different depths, resulting in a different calculation per pixel, which may make parallelization more difficult [S3]. True depth-varying Fresnel method takes roughly $O(m^4)$ operations [S3]. A single-plane approximation takes only $O(m^2 \log m)$ operations [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Complex numbers are stored using exactly 32 bits [S2]. Fixed-point representation is used to minimize bit depth [S2].
*   **Measured Performance:** A single-chip FPGA implementation generates binocular holographic colour images with 3840 × 2160 pixels at 30 fps [S2]. Scenes with 8 million voxels are rendered at 30 fps [S2].
*   **Scaling:** The method is implemented on a single-chip FPGA, suggesting scalability through parallel architecture [S2].
*   **Strengths:** Generates high-quality holograms for real scenes [S2]. Enables real-time embedded rendering [S1].
*   **Limitations:** Historically, FFT-based layer methods were considered unsuitable for CGH hardware accelerators due to the difficulty of building large-scale FFT processors [S2].
*   **When to use:** When high-quality binocular color holography is required for real scenes, and hardware acceleration via FPGA is available [S2].
*   **When not:** When computational resources are limited to CPUs without specialized FFT hardware, as large-scale FFT processors are difficult to build [S2].

## Background
*   **Origin:** The method is used in holographic near-eye displays and slim-panel holographic video displays [S1, S2].
*   **History:** Layer-based methods using FFTs were traditionally considered unsuitable for CGH hardware accelerators because FFT processors were difficult to build in large scale [S2]. Recent implementations have overcome this using highly parallel architectures [S2].
*   **Usage:** Implemented in a single-chip FPGA holographic video processor [S2]. Used to calculate CGH from Unity-rendered 3D scenes [S2].

## Sources
[S1] Holographic Near-eye Display with Real-time Embedded Rendering, https://doi.org/10.1145/3610548.3618179, fetched 2026-10-28
[S2] Slim-panel holographic video display, https://doi.org/10.1038/s41467-020-19298-4, fetched 2026-10-26
[S3] Holographic near-eye displays based on overlap-add stereograms, https://doi.org/10.1145/3355089.3356517, fetched 2026-11-01
