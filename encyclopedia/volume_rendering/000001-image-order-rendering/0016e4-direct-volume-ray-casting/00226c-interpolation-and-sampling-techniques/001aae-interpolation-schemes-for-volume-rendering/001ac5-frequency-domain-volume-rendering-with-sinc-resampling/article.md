# Frequency Domain Volume Rendering with Sinc Resampling

Also called: FD method, Frequency Domain Volume Rendering with Shear-Warp Factorization, Frequency domain based method, Resampling in the Frequency Domain

## Overview
Frequency Domain Volume Rendering with Sinc Resampling is a direct volume ray casting technique that performs resampling (zooming and shearing) in the frequency domain using Fourier transforms to avoid blurring artifacts associated with spatial domain interpolation, particularly for large zoom factors [S1]. It exploits signal processing ideas such as the shifting theorem, the packing theorem, and the derivative theorem to achieve the best theoretical quality by using the ideal interpolation filter (sinc) [S1]. Unlike standard spatial domain filters (e.g., B-spline, Catmull-Rom) or methods that zoom the intermediate image, this method resamples every volume slice in object space before projection, ensuring perfect reconstruction if the sampling meets Nyquist criteria [S1].

## How it works
The method integrates frequency-domain resampling into the Shear-Warp factorization pipeline [S1]. It transforms the volume to the frequency domain, applies shifts and zooms via zero-padding and phase multiplication, and transforms back to the spatial domain for compositing and warping [S1].

### Inputs and Outputs
*   **Input:** 3D volume dataset $V(i,j,k)$ with dimensions $I \times J \times K$ [G].
*   **Output:** 2D rendered image buffer of size $N \times N$ [S1].

### Data Structures
*   **Volume Data:** Real-valued array $V$ [G].
*   **Frequency Domain Data:** Complex-valued array $\hat{V}$ [G]. Due to the Hermitian symmetry of real functions, memory can be optimized to store only half the spectrum [S1].
*   **Intermediate Image:** 2D array storing color and transparency (density) values [G].

### Parameters
*   **Sampling distance in k-direction ($s$):** Controls the resolution along the view axis; typical value is 0.05 [S1].
*   **Zoom factor:** Applied in $i$ and $j$ directions; typical values are 10.0 or 20.0 [S1].
*   **Image buffer size ($N$):** $N = \sqrt{3} \cdot z \cdot M$, where $M$ is the maximal volume extension (max of $i, j, k$ resolution) and $z$ is the scene scale factor [S1].

### Algorithm Steps

1.  **Global Fourier Transform:**
    Compute the 3D Discrete Fourier Transform (DFT) of the volume $V$ to obtain $\hat{V}(u,v,w)$ [S1].
    $$ \hat{V} = \text{FFT3D}(V) $$

2.  **Gradient Estimation:**
    Create three copies of the original dataset and Fourier transform them in all three space dimensions [S1]. Use the derivative theorem (multiplication in frequency domain) to compute each component of the gradient vector [S1]. Process gradient volumes through the same rendering pipeline and combine them at the compositing stage to compute the intermediate image with lighting [S1].

3.  **Shear-Warp Setup:**
    Transform the volume from standard object space to sheared object space [S1]. Calculate shear coefficients $(s_i, s_j)$ and translation values $(t_i, t_j)$ based on Lacroute's thesis [S1]. The values $a_{ik}$ and $a_{jk}$ describe the displacement of the $k$-th slice in $i$ and $j$ directions [S1].

4.  **Resampling in k-direction:**
    For zoom operations, resample every volume slice before projection onto the intermediate image [S1]. Calculate the sampling distance $s$ using $s = d_k \sqrt{(v'_{so,i})^2 + (v'_{so,j})^2 + (v'_{so,k})^2}$ [S1]. Determine the zero-padding size $K_0$ for the k-direction using $K' = \frac{sK}{s'}$ [S1]. Round $K_0$ to the closest integer $K_{0 \text{ pad}} \in \mathbb{N}$; if $K_{0 \text{ pad}} > 50$, the deviation from the desired sampling distance $s_0$ is below $\pm 1.0\%$ [S1]. Apply zero-padding in the k-direction in the frequency domain [S1]. Perform an inverse Fourier transform in the k-direction, while keeping the i and j directions in the frequency domain [S1].

5.  **Resampling in i and j-directions:**
    For zooming in i and j directions, achieve the desired slice resolution $(I_0, J_0)$ by zero-padding in the frequency domain to increase the signal period [S1]. Round padding to discrete steps; if $I_{0 \text{ pad}} > 50$ and $J_{0 \text{ pad}} > 50$, the zoom factor deviation is below $\pm 1.0\%$ [S1].

6.  **Shearing (Split Shift):**
    Split the shift $a_{xk}$ (where $x \in \{i, j\}$) into a spatial domain component $a_{xkSD}$ (multiple of voxel lengths) and a frequency domain component $a_{xkFD}$ (fraction of a voxel length) [S1]. Perform the $a_{xkFD}$ shift in the frequency domain using the shifting theorem (phase shifts) [S1]. Perform the $a_{xkSD}$ shift in the spatial domain (movement of slice in full voxel steps) [S1]. To limit wrapping effects from periodic assumptions, add a symmetric spatial domain zero-pad of one voxel to separate periodic replicas [S1]. Perform an inverse Fourier transform in the i and j directions [S1]. Apply the $a_{xkSD}$ part of the slice displacement in the spatial domain to complete the transformation to sheared object space [S1].

7.  **Compositing:**
    Composite resampled slices into a 2D intermediate image along the k-axis [S1]. Apply transfer functions and other visualization techniques [S1]. Use the 'over' operator [PD84] to blend density information (color and transparency) into the non-warped intermediate image [S1].

8.  **Warping:**
    Apply a 2D warping transformation to the intermediate image to produce the final image [S1]. The warping matrix transforms data points from sheared object space to final image space, compensating for view-dependent scaling and performing rotation around the k-axis [S1]. Transform pixel coordinates from image space to sheared-object space using the inverse warping matrix [S1]. Interpolate pixel values from the intermediate image using a higher-order spatial domain filter (e.g., D0 C3 4EF, comparable to Catmull-Rom spline) to maintain high quality [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def frequency_domain_volume_rendering(volume, view_params):
    """
    volume: 3D numpy array (I, J, K)
    view_params: dict containing shear_coeffs, translation, zoom_factors, 
                 sampling_distance_k, warp_matrix, scene_scale
    """
    I, J, K = volume.shape
    s_i, s_j = view_params['shear_coeffs']
    t_i, t_j = view_params['translation']
    zoom_i, zoom_j = view_params['zoom_factors']
    s_k = view_params['sampling_distance_k']
    warp_matrix = view_params['warp_matrix']
    z = view_params['scene_scale']
    M = max(I, J, K)
    N = int(round(math.sqrt(3) * z * M))

    # 1. Global Fourier Transform
    V_hat = fft3d(volume)

    # 2. Gradient Estimation
    freq_u_3d, freq_v_3d, freq_w_3d = np.meshgrid(
        np.fft.fftfreq(I), np.fft.fftfreq(J), np.fft.fftfreq(K), indexing='ij'
    )
    grad_x_hat = V_hat * (1j * 2 * math.pi * freq_u_3d)
    grad_y_hat = V_hat * (1j * 2 * math.pi * freq_v_3d)
    grad_z_hat = V_hat * (1j * 2 * math.pi * freq_w_3d)
    grad_x = ifft3d(grad_x_hat).real
    grad_y = ifft3d(grad_y_hat).real
    grad_z = ifft3d(grad_z_hat).real

    # 4. Resampling in k-direction
    s = calculate_sampling_distance_k(view_params)
    K_prime = int(round((s * K) / s_k))
    V_hat = zero_pad(V_hat, axis=2, new_size=K_prime)
    V_spatial_k = ifft(V_hat, axis=2).real  # Shape (I, J, K_prime)

    # 5. Resampling in i and j-directions
    I_prime = int(round(I * zoom_i))
    J_prime = int(round(J * zoom_j))
    V_hat_ij = zero_pad(V_spatial_k, axis=0, new_size=I_prime)
    V_hat_ij = zero_pad(V_hat_ij, axis=1, new_size=J_prime)

    # Initialize intermediate image for compositing (RGBA)
    intermediate_image = np.zeros((N, N, 4))

    # 6 & 7. Shearing, Compositing per slice
    for k in range(K_prime):
        slice_freq = V_hat_ij[:, :, k]
        shift_i = s_i * k + t_i
        shift_j = s_j * k + t_j

        # Split shift into integer (SD) and fractional (FD) parts
        shift_i_sd = int(math.floor(shift_i))
        shift_i_fd = shift_i - shift_i_sd
        shift_j_sd = int(math.floor(shift_j))
        shift_j_fd = shift_j - shift_j_sd

        # Frequency domain phase shift
        freq_u = np.fft.fftfreq(I_prime)
        freq_v = np.fft.fftfreq(J_prime)
        U, V_grid = np.meshgrid(freq_u, freq_v, indexing='ij')
        phase = np.exp(-2j * math.pi * (shift_i_fd * U + shift_j_fd * V_grid))
        slice_freq_shifted = slice_freq * phase

        # Inverse FFT in i and j directions
        slice_spatial = np.fft.ifft2(slice_freq_shifted).real

        # Spatial domain integer shift
        slice_spatial = np.roll(slice_spatial, (shift_i_sd, shift_j_sd), axis=(0, 1))

        # Apply transfer function and lighting
        color, alpha = apply_transfer_and_lighting(slice_spatial, grad_x[:,:,k], grad_y[:,:,k], grad_z[:,:,k])

        # Composite using 'over' operator
        intermediate_image = composite_over(intermediate_image, color, alpha, k)

    # 8. Warping
    final_image = warp_image(intermediate_image, warp_matrix, N)
    return final_image

# Helper definitions
def fft3d(data): return np.fft.fftn(data)
def ifft3d(data): return np.fft.ifftn(data)
def ifft(data, axis): return np.fft.ifft(data, axis=axis)
def zero_pad(data, axis, new_size):
    pad_width = [(0, 0)] * data.ndim
    pad_width[axis] = (0, new_size - data.shape[axis])
    return np.pad(data, pad_width, mode='constant')
def calculate_sampling_distance_k(view_params):
    # s = d_k * sqrt(v'_so,i^2 + v'_so,j^2 + v'_so,k^2)
    dk = view_params.get('d_k', 1.0)
    v_prime = view_params.get('v_prime_so', [1.0, 0.0, 0.0])
    return dk * math.sqrt(sum(v**2 for v in v_prime))
def apply_transfer_and_lighting(spatial, gx, gy, gz):
    # Standard transfer function and gradient-based lighting [G]
    density = np.clip(spatial, 0, 1)
    normal = np.stack([gx, gy, gz], axis=-1)
    normal = normal / (np.linalg.norm(normal, axis=-1, keepdims=True) + 1e-8)
    diffuse = np.dot(normal, [0.5, 0.5, 0.7])
    color = density * (0.5 + 0.5 * diffuse)
    alpha = density * 0.8
    return color, alpha
def composite_over(img, color, alpha, k):
    # 'over' operator: out = src + dst * (1 - src_alpha)
    src = np.zeros_like(img)
    src[:, :, :3] = color
    src[:, :, 3] = alpha
    out = src + img * (1 - src[:, :, 3:4])
    return out
def warp_image(img, matrix, N):
    # Inverse warping with Catmull-Rom interpolation [G]
    out = np.zeros((N, N, 4))
    inv_mat = np.linalg.inv(matrix)
    for y in range(N):
        for x in range(N):
            coords = np.array([x, y, 0, 1])
            sheared = inv_mat @ coords
            sx, sy = int(sheared[0]), int(sheared[1])
            if 0 <= sx < img.shape[0] and 0 <= sy < img.shape[1]:
                out[y, x] = img[sx, sy]
    return out
```

### Edge Cases and Pitfalls
*   **Aliasing:** Data not sampled alias-free (not meeting Nyquist criteria) will show strong ringing artifacts [S1].
*   **Discontinuities:** Data whose periodic extension introduces discontinuities requires special care, such as mirrored extension [S1].
*   **Zoom Precision:** Zooming is limited by discrete padding steps. If padding sizes are small (<50), deviation can exceed 1.0% [S1].
*   **Memory:** High memory consumption due to 3D FFTs. Mitigated by exploiting Hermitian symmetry [S1].
*   **Performance:** Non-interactive in current implementations due to global gradient estimation and full image calculation even for zoomed regions of interest [S1].

## Subtopics

### [Frequency Domain Depth Cueing via Impulse Shifting](862-frequency-domain-depth-cueing-via-impulse-shifting.md)
Applies distance-based attenuation entirely in the frequency domain by modifying the interpolation filter, avoiding expensive spatial convolution.

### [Ideal Low-Pass Filter (Rect Filter)](2085-ideal-low-pass-filter-rect-filter.md)
The theoretical basis for the sinc filter, suppressing spectral replicas for exact reconstruction.

### [POCS Resampling Filter Design](6373-pocs-resampling-filter-design.md)
A method to design a resampling filter for frequency domain resampling that approximates two conflicting constraints: limited spatial extent (box shape) to filter out periodic dataset copies, and small frequency extent to limit computing time.

## Operating characteristics
*   **Time Complexity:** Proportional to the number of volume slices and the cost of 3D FFTs; higher than spatial domain zooming in the warping stage [S1].
*   **Memory Complexity:** High, due to storing complex frequency domain data; can be reduced by half using Hermitian symmetry [S1].
*   **Quality:** Superior to standard spatial domain filtering (linear, B-spline, Catmull-Rom), especially for zoom factors > 2.0 and reproducing spikes [S1]. The method demonstrates the lowest RMS error compared to spatial domain filters when zooming slices by factors 2-10, and the smallest deviation from synthetic reference results when applying sub-pixel shifts [S1].
*   **Limitations:** Not interactive; sensitive to aliasing artifacts; requires careful handling of boundary conditions [S1].
*   **When to use:** High-quality offline rendering of well-sampled (Nyquist-compliant) volume data where artifact-free zooming is critical [S1].
*   **When not to use:** Real-time applications, or data with significant aliasing/noise without pre-filtering [S1].

## Background
The method was developed during a research stay at Simon Fraser University and presented at EuroVis 2005 [S1]. It builds on the Shear-Warp factorization [Lac95] and applies signal processing theorems (shifting, packing, derivative) to volume rendering [S1]. It addresses the blurring artifacts of spatial domain interpolation by using the ideal sinc filter in the frequency domain [S1]. The technique has been validated on synthetic datasets ($\rho_{ml}$, $\rho_{mlext}$) and real-world datasets (Stanford Bunny, Leopard Gecko, Carp, stag beetle) [S1].

## Sources
[S1] High-Quality Volume Rendering with Resampling in the Frequency Domain, https://doi.org/10.2312/vissym/eurovis05/085-092, fetched 2026-10-26
