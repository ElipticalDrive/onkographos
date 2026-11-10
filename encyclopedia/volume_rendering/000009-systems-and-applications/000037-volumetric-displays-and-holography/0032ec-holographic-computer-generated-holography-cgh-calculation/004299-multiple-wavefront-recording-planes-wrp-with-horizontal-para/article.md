# Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)

Also called: Fast CGH method for point clouds, HPO WRP method

## Overview
This method generates computer-generated holograms (CGH) for point clouds by distributing the calculation across multiple parallel Wavefront Recording Planes (WRPs), adapted specifically for Horizontal Parallax Only (HPO) displays. It solves the computational bottleneck of full-parallax CGH by restricting diffraction to 1D hololines, while maintaining occlusion handling and high-quality surface reconstruction through Gaussian volume attribution and inverted Gaussian filtering. It is distinguished from full-parallax WRP methods by its 1D propagation constraint and from single-plane methods by its ability to handle large depth ranges without aliasing via multi-plane decomposition.

## How it works
### Inputs and Outputs
*   **Inputs:** A 3D point cloud where each point has a 3D position $(x, y, z)$ and a color value $(R, G, B)$. The display parameters include pixel pitch ($p_x, p_y$), resolution ($N_x, N_y$), wavelength ($\lambda$), and the number of WRPs ($M$).
*   **Outputs:** A complex-valued hologram field $U_h(x, y)$ (or phase-only map) for the target hologram plane, or intermediate complex fields for each WRP.

### Data Structures
1.  **Point Cloud:** List of points $P_i = (x_i, y_i, z_i, R_i, G_i, B_i)$.
2.  **WRP Grid:** A set of $M$ parallel planes at depths $z_1, z_2, \dots, z_M$. Each WRP is a 2D grid of complex amplitudes. For HPO, the WRP data is stored as 1D hololines (rows) corresponding to the horizontal parallax, with vertical resolution matching the display's vertical pixel count or a subsampled equivalent.
3.  **Look-Up Tables (LUTs):** Pre-computed propagation kernels $h(x, y, z)$ and their Fourier transforms $H(u, v; z)$ for specific distances between WRPs and between WRPs and the hologram plane.

### Algorithm Steps

1.  **Initialization:**
    *   Define the hologram plane at $z=0$.
    *   Define $M$ WRP planes at depths $z_k$ for $k=1 \dots M$.
    *   Initialize the complex field on each WRP to zero.

2.  **Point Attribution (Gaussian Volume):**
    *   To generate smoother reconstructions, attribute a Gaussian volume to each point $P_i$. The amplitude contribution of point $i$ at a location $(x, y)$ on a WRP at distance $z$ is modeled by a Gaussian profile rather than a delta function. This reduces high-frequency artifacts.

3.  **LUT Generation (Band-Limited Angular Spectrum):**
    *   Compute the propagation kernel $h(x, y, z)$ for the required distances.
    *   Apply the Band-Limited Angular Spectrum Method (BL-ASM) to handle small sampling pitches and aperiodic functions.
    *   The transfer function $H(u, v; z)$ is clipped to a bandwidth limit:
        $$ H'(u, v; z) = H(u, v; z) \cdot \text{rect}\left(\frac{u}{2u_{limit}}\right) \cdot \text{rect}\left(\frac{v}{2v_{limit}}\right) $$
    *   To implement BL-ASM in practice:
        *   Double the area of the calculation window in the source plane.
        *   Pad additional sampling points with zeros.
        *   Perform the FFT-based convolution.
        *   Crop the output window to the original window size at the destination plane.
    *   Store $H'(u, v; z)$ in LUTs for forward and backward propagation.

4.  **WRP Field Computation (HPO Adaptation):**
    *   For each point $P_i$ in the point cloud:
        *   Determine which WRP(s) it contributes to based on its depth $z_i$.
        *   Compute the contribution of $P_i$ to the hololines of the relevant WRP(s).
        *   Since this is HPO, the contribution is computed only along the horizontal line (1D hologram) of the WRP. The vertical dimension is handled by the display's native resolution or by summing contributions from multiple vertical slices if necessary, but the core diffraction calculation is 1D.
        *   **Occlusion Handling:** Use inverted Gaussian filters in the propagation process to model occlusions. If a point is occluded by another point closer to the viewer, its contribution to the WRP is locally blocked or attenuated.
        *   **Diffuse Reflections:** Modify the spectral response of the LUT content with a random phase to account for diffuse reflections, scattering light over a wide angular range. This ensures a wide field of view.

5.  **Inter-WRP Propagation:**
    *   Propagate the complex fields from the WRPs to the hologram plane using the pre-computed LUTs.
    *   For each WRP $k$, propagate its field $U_{WRP_k}$ to the hologram plane $U_h$ using the BL-ASM method:
        $$ U_h = \sum_{k=1}^{M} \mathcal{F}^{-1} \left\{ \mathcal{F}\{U_{WRP_k}\} \cdot H'(u, v; z_k) \right\} $$
    *   Sum the contributions from all WRPs to form the total hologram field.

6.  **Color Encoding:**
    *   For color holograms, compute the hologram for the red channel wavelength $\lambda_R$ using the RGB intensities $(R_i, G_i, B_i)$ scaled appropriately.
    *   The final hologram consists of three channels (R, G, B), all computed for the same wavelength (e.g., red) to maintain a wide viewing angle, but with different intensities per point for each channel.

7.  **Encoding:**
    *   Convert the complex field $U_h(x, y)$ to a format displayable on the SLM (e.g., phase-only via Gerchberg-Saxton or direct argument extraction).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bl_asm_fft(field, kernel, pad_factor=2):
    """
    Perform Band-Limited Angular Spectrum propagation using FFT.
    field: 2D complex array (source plane)
    kernel: 2D complex array (transfer function H')
    pad_factor: factor to double the window size
    """
    # Double the area by padding with zeros
    h, w = field.shape
    pad_h, pad_w = h * pad_factor, w * pad_factor
    padded_field = np.zeros((pad_h, pad_w), dtype=np.complex128)
    padded_field[:h, :w] = field
    
    # FFT
    F_field = np.fft.fft2(padded_field)
    F_kernel = np.fft.fft2(kernel, s=(pad_h, pad_w))
    
    # Multiply in frequency domain
    F_out = F_field * F_kernel
    
    # Inverse FFT
    out_padded = np.fft.ifft2(F_out)
    
    # Crop to original size
    out = out_padded[:h, :w]
    return out

def compute_hpo_wrp_cgh(point_cloud, wrp_depths, hologram_res, pixel_pitch, wavelength, num_wrps):
    """
    Compute HPO CGH using Multiple WRPs.
    """
    # 1. Initialize WRP fields
    # For HPO, WRP fields are 1D hololines. 
    # Assuming vertical resolution matches display, horizontal is the parallax dimension.
    h, w = hologram_res
    wrp_fields = [np.zeros((h, w), dtype=np.complex128) for _ in range(num_wrps)]
    
    # 2. Pre-compute LUTs using BL-ASM
    # Create frequency grid
    u = np.fft.fftfreq(w, d=pixel_pitch[0])
    v = np.fft.fftfreq(h, d=pixel_pitch[1])
    U, V = np.meshgrid(u, v)
    
    # Bandwidth limits
    u_limit = 1 / (2 * pixel_pitch[0])
    v_limit = 1 / (2 * pixel_pitch[1])
    
    # Transfer function H(u,v;z)
    # w_freq = sqrt(1/lambda^2 - u^2 - v^2)
    w_freq = np.sqrt(np.maximum(0, (1/wavelength**2) - U**2 - V**2))
    
    luts = []
    for z in wrp_depths:
        H = np.exp(1j * 2 * np.pi * w_freq * z)
        # Apply band-limiting
        H = H * np.heaviside(u_limit - np.abs(U), 1.0) * np.heaviside(v_limit - np.abs(V), 1.0)
        luts.append(H)
        
    # 3. Process each point
    for point in point_cloud:
        x, y, z, r, g, b = point
        
        # Determine which WRP this point contributes to
        # Simple depth-based assignment or interpolation
        wrp_idx = find_closest_wrp(z, wrp_depths)
        
        # Compute contribution to WRP
        # For HPO, we compute the 1D hololine contribution
        # This involves propagating the point's Gaussian volume to the WRP plane
        # and applying occlusion/inverted Gaussian filtering
        
        # Simplified: Add point contribution to WRP field
        # In practice, this is a convolution with the point's Gaussian profile
        # and application of random phase for diffuse reflection
        phase = np.random.uniform(0, 2 * np.pi)
        amplitude = np.sqrt(r) # Intensity scaling
        
        # Add to WRP field at corresponding position
        # Note: Exact position mapping depends on coordinate system
        wrp_fields[wrp_idx][y, x] += amplitude * np.exp(1j * phase)
        
    # 4. Propagate WRPs to Hologram Plane
    hologram_field = np.zeros((h, w), dtype=np.complex128)
    
    for i, wrp_field in enumerate(wrp_fields):
        # Propagate from WRP to Hologram using BL-ASM
        # The LUT is pre-computed for the distance from WRP to Hologram
        propagated = bl_asm_fft(wrp_field, luts[i])
        hologram_field += propagated
        
    return hologram_field
```

### Parameters
*   **Pixel Pitch:** $p_x = 0.55 \, \mu\text{m}$, $p_y = 23.4 \, \mu\text{m}$ [S1].
*   **Resolution:** $32,768 \times 768$ pixels [S1].
*   **Wavelength:** $640 \, \text{nm}$ [S1].
*   **Minimum Distance:** $z_{min} = \frac{N p^2}{\lambda} = 6.3 \, \text{mm}$ [S1].
*   **Viewing Angle:** $70^\circ$ [S1].

### Edge Cases and Pitfalls
*   **Aliasing:** Small sampling pitch leads to strong diffusion of the light field beyond computation window borders. This is handled by BL-ASM [S1].
*   **Aperiodic Functions:** Cause errors at edges due to circular convolution in FFT. BL-ASM with zero-padding and cropping mitigates this [S1].
*   **Occlusion:** Without inverted Gaussian filters, occlusion is not modeled, leading to incorrect light distribution [S1].
*   **HPO Limitations:** Constrained viewing zones and astigmatic output due to diminished spatial resolution in the vertical direction [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M \cdot \log(N))$ where $N$ is the number of pixels in the hologram and $M$ is the number of WRPs. The use of LUTs and FFTs makes it faster than direct summation.
*   **Memory Complexity:** $O(N \cdot M)$ for storing WRP fields and LUTs.
*   **Performance:** Suitable for real-time or near-real-time rendering on high-end hardware. The HPO adaptation reduces computational load by a factor of the vertical resolution compared to full parallax.
*   **Strengths:** Handles occlusion, supports wide viewing angles, generates high-quality smooth surfaces.
*   **Limitations:** HPO restricts vertical parallax, leading to astigmatism. Requires careful tuning of BL-ASM parameters to avoid artifacts.

## Background
*   **Origin:** Proposed by [S1] for rendering point clouds on light field displays.
*   **Development:** Built upon the multiple WRP method [S1, S15] and adapted for HPO. Uses BL-ASM [S1, S17] to handle sampling constraints.
*   **Usage:** Demonstrated on Holografika HoloVizio 722RC light field display [S1].

## Sources
[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
