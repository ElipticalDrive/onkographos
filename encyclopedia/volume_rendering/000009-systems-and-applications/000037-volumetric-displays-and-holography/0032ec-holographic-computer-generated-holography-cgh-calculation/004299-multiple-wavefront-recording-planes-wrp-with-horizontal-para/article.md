# Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)

Also called: Fast CGH method for point clouds, HPO WRP method

## Overview
This method generates computer-generated holograms (CGH) for point clouds by decomposing the 3D scene into multiple parallel Wavefront Recording Planes (WRPs) and computing the hologram as the superposition of wavefields propagated from these planes. It is specifically adapted for Horizontal Parallax Only (HPO) displays by restricting the diffraction calculation to 1D hololines rather than full 2D planes, significantly reducing computational load. It distinguishes itself from full-parallax WRP methods by optimizing for displays with high horizontal resolution but lower vertical resolution, and from direct point-to-hologram methods by using intermediate planes to handle occlusion and enable look-up table (LUT) acceleration.

## How it works

### Inputs and Outputs
**Inputs:**
*   Point cloud data: A set of $M$ points, where each point $i$ has position $(x_i, y_i, z_i)$, amplitude $A_i$, and phase $\phi_i$. For color, points have RGB intensity values.
*   Display parameters: Pixel pitch $p_x, p_y$, resolution $N_x, N_y$, wavelength $\lambda$, and minimum object distance $z_{min}$.
*   WRP configuration: Number of WRP planes $K$, their positions $z_k$ relative to the hologram plane, and the spatial resolution of each WRP.

**Outputs:**
*   Complex amplitude distribution $U_h(x,y)$ or phase $\phi_h(x,y)$ on the hologram plane.

### Data Structures
1.  **Point Cloud:** List of tuples `(x, y, z, r, g, b)`.
2.  **WRP Planes:** A list of $K$ 2D grids. Each grid stores complex amplitude values. The grid dimensions match the hologram resolution or a scaled version thereof.
3.  **Look-Up Tables (LUTs):** Pre-computed propagation kernels. Since the method uses parallel planes, kernels are stored for specific distances $z$. The LUT contains the complex transfer function $H(u,v;z)$ or the spatial domain kernel $h(x,y;z)$ depending on the implementation variant (frequency domain is preferred for speed).
4.  **Hologram Plane:** A 2D complex grid of size $N_x \times N_y$ (or $N_x \times 1$ for HPO hololines).

### Parameters and Typical Values
*   **Wavelength ($\lambda$):** 640 nm (Red). For color, RGB channels are computed at this wavelength to maintain a wide viewing angle, modulating only intensity [S1].
*   **Pixel Pitch ($p$):** Horizontal $p_x = 0.55 \, \mu\text{m}$, Vertical $p_y = 23.4 \, \mu\text{m}$ [S1].
*   **Resolution:** $32,768 \times 768$ pixels [S1].
*   **Minimum Distance ($z_{min}$):** Calculated as $z_{min} = \frac{N p^2}{\lambda}$. For the above parameters, $z_{min} \approx 6.3 \, \text{mm}$ [S1].
*   **Maximum Diffraction Angle ($\theta$):** Determined by $\sin \theta = \frac{\lambda}{2p}$. With $p_x=0.55 \, \mu\text{m}$ and $\lambda=640 \, \text{nm}$, $\theta \approx 35.5^\circ$, yielding a FOV of $\approx 71^\circ$ [S1].

### Algorithm Steps

1.  **Pre-computation of LUTs:**
    *   Calculate the Band-Limited Angular Spectrum (BL-ASM) transfer function $H'(u,v;z)$ for all required propagation distances $z$ between the hologram and WRP planes, and between adjacent WRP planes.
    *   The transfer function is defined as:
        $$ H'(u, v; z) = \exp(i 2\pi w z) \cdot \text{rect}\left(\frac{u}{2u_{limit}}\right) \cdot \text{rect}\left(\frac{v}{2v_{limit}}\right) $$
        where $w = \sqrt{\lambda^{-2} - u^2 - v^2}$ if $u^2 + v^2 \le \lambda^{-2}$, else $0$ [S1].
    *   The bandwidth limits $u_{limit}, v_{limit}$ are chosen to satisfy the Nyquist criterion for the small pixel pitch, preventing aliasing and edge errors caused by circular convolution in FFT [S1].

2.  **Initialization of WRPs:**
    *   Initialize $K$ WRP planes with zero complex amplitude.

3.  **Point-to-WRP Propagation (Source Plane to WRPs):**
    *   For each point $i$ in the point cloud:
        *   Determine which WRP plane $k$ is closest to the point's $z$-coordinate $z_i$.
        *   Compute the contribution of point $i$ to the WRP plane $k$.
        *   **Gaussian Volume Attribution:** Instead of a delta function, attribute a Gaussian distribution profile to the point to create smoother reconstructions. The amplitude at position $(x,y)$ on the WRP is:
            $$ A_{i,k}(x,y) = A_i \exp\left(-\frac{(x-x_i)^2 + (y-y_i)^2}{2\sigma^2}\right) $$
            where $\sigma$ controls the spread [S1].
        *   **Random Phase for Diffuse Reflections:** Modify the spectral response with a random phase $\phi_{rand}$ to scatter light over a wide angular range, enhancing the field of view:
            $$ U_{i,k}(x,y) = A_{i,k}(x,y) \exp(i (\phi_i + \phi_{rand})) $$
        *   **HPO Adaptation:** Since the display is HPO, the calculation is restricted to 1D hololines. The point contributes only to the row (or column, depending on orientation) corresponding to its $y$-coordinate (or $x$-coordinate). The diffraction is computed only along this line.
        *   **Occlusion Handling:** Use inverted Gaussian filters during propagation to model occlusions. If a point is behind another object, its contribution is locally blocked [S1].
        *   Add $U_{i,k}(x,y)$ to the WRP plane $k$.

4.  **Inter-WRP Propagation:**
    *   Propagate the wavefields from the source WRPs to the hologram plane.
    *   If points are distributed across multiple WRPs, propagate the wavefield from each WRP to the hologram plane using the pre-computed LUTs.
    *   **BL-ASM Implementation:**
        *   Double the area of the calculation window in the source plane (WRP) by padding zeros.
        *   Perform FFT, multiply by $H'(u,v;z)$, and inverse FFT.
        *   Crop the output window to the original size at the destination plane (Hologram) to avoid edge artifacts [S1].
    *   Sum the contributions from all WRP planes to get the total complex amplitude on the hologram plane:
        $$ U_h(x,y) = \sum_{k=1}^{K} U_{WRP_k \to h}(x,y) $$

5.  **Color Computation:**
    *   For color holograms, repeat steps 3-4 for Red, Green, and Blue channels.
    *   Use the same wavelength $\lambda$ (e.g., 640 nm) for all channels to maintain a consistent wide viewing angle.
    *   Modulate the intensity of each point based on its RGB texture values [S1].

6.  **Phase Extraction:**
    *   Extract the phase $\phi_h(x,y) = \arg(U_h(x,y))$ for phase-only SLMs.
    *   Apply encoding techniques (e.g., detour phase, Gerchberg-Saxton) if necessary to mitigate twin-image artifacts and noise [S1, Parent].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_bl_asm_transfer_function(u, v, z, lambda_w, u_limit, v_limit):
    """
    Computes the Band-Limited Angular Spectrum transfer function.
    """
    w = np.sqrt(1.0 / lambda_w**2 - u**2 - v**2)
    # Mask for evanescent waves
    mask = (u**2 + v**2) <= (1.0 / lambda_w**2)
    w[~mask] = 0
    
    # Band-limiting
    rect_u = np.where(np.abs(u) <= u_limit, 1.0, 0.0)
    rect_v = np.where(np.abs(v) <= v_limit, 1.0, 0.0)
    
    H = np.exp(1j * 2 * np.pi * w * z) * rect_u * rect_v
    return H

def gaussian_profile(x, y, x0, y0, sigma):
    """
    Computes a 2D Gaussian profile centered at (x0, y0).
    """
    dist_sq = (x - x0)**2 + (y - y0)**2
    return np.exp(-dist_sq / (2 * sigma**2))

def generate_hpo_cgh(point_cloud, wrp_planes, hologram_resolution, lambda_w, pixel_pitch, sigma, random_phase_seed):
    """
    Generates HPO CGH using Multiple WRP method.
    """
    # Initialize hologram plane (HPO: 1D hololines, so shape is (N_x, 1) or similar)
    # Assuming HPO along X, so we compute for each Y line separately.
    # For simplicity, we assume the hologram is 2D but contributions are 1D.
    hologram_complex = np.zeros((hologram_resolution[0], hologram_resolution[1]), dtype=complex)
    
    # Pre-compute LUTs for all relevant z distances
    # This step is done once and cached
    # LUT[z_index] = H'(u,v;z)
    
    for point in point_cloud:
        x, y, z, r, g, b = point
        amplitude = 1.0 # Normalized
        
        # Find closest WRP
        wrp_idx = find_closest_wrp(z, wrp_planes)
        wrp_z = wrp_planes[wrp_idx].z
        wrp_plane = wrp_planes[wrp_idx].grid
        
        # Compute contribution to WRP
        # HPO: Only diffract along the line corresponding to point's y
        # Create a 1D Gaussian profile along x for this y
        x_coords = np.arange(wrp_plane.shape[0])
        y_coords = np.arange(wrp_plane.shape[1])
        
        # Gaussian spread
        gauss = gaussian_profile(x_coords, x, sigma)
        
        # Random phase for diffuse reflection
        np.random.seed(random_phase_seed + wrp_idx)
        rand_phase = np.random.uniform(0, 2 * np.pi, size=wrp_plane.shape[0])
        
        # Complex amplitude on WRP for this point
        # Shape: (N_x,)
        wrp_contribution = amplitude * gauss * np.exp(1j * (point.phase + rand_phase))
        
        # Add to WRP plane at the specific y-row
        wrp_y_idx = int(y / pixel_pitch)
        wrp_plane[:, wrp_y_idx] += wrp_contribution
        
    # Propagate WRPs to Hologram
    for wrp_idx, wrp in enumerate(wrp_planes):
        # Get transfer function for distance z = wrp.z
        H = get_lut(wrp.z, lambda_w, pixel_pitch)
        
        # BL-ASM: Pad source
        source = wrp.grid
        padded_source = np.pad(source, ((0,0), (0, source.shape[1])), mode='constant') # Pad in x? Or both?
        # Note: The source says "double the area... pad additional sampling points as zeros"
        # Usually this means padding in both dimensions or just the one with high frequency content.
        # For HPO, we might only need to pad in X.
        
        # FFT
        F_source = np.fft.fft2(padded_source)
        
        # Multiply by Transfer Function
        # H needs to be padded to match F_source size
        F_propagated = F_source * H_padded
        
        # IFFT
        propagated = np.fft.ifft2(F_propagated)
        
        # Crop to original size
        cropped = propagated[:source.shape[0], :source.shape[1]]
        
        # Add to hologram
        hologram_complex += cropped
        
    return hologram_complex
```

### Edge Cases and Pitfalls
*   **Small Sampling Pitch:** Leads to strong diffusion of the light field beyond computation window borders. This is handled by BL-ASM by doubling the calculation window and padding with zeros [S1].
*   **Aperiodic Functions:** Cause errors at edges due to circular convolution in FFT. BL-ASM handles this by cropping the output window [S1].
*   **Occlusion:** Standard point-to-plane propagation does not handle occlusion. Inverted Gaussian filters are used to locally block light from points behind other objects [S1].
*   **HPO Astigmatism:** HPO causes constrained viewing zones and astigmatic output due to diminished vertical resolution. This is a hardware limitation, not a computational error, but must be accounted for in the design [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** $O(M \cdot K \cdot N_{holo})$, where $M$ is the number of points, $K$ is the number of WRPs, and $N_{holo}$ is the number of pixels in a hololine. The use of LUTs and FFT reduces the constant factor significantly compared to direct point-to-hologram calculation.
*   **Memory Complexity:** $O(K \cdot N_{WRP} + N_{LUT})$, where $N_{WRP}$ is the resolution of a WRP and $N_{LUT}$ is the size of the pre-computed kernels.

### Measured Performance
*   **Hardware:** Holografika HoloVizio 722RC light field display [S1].
*   **Resolution:** $32,768 \times 768$ pixels [S1].
*   **Viewing Angle:** $70^\circ$ [S1].
*   **Minimum Distance:** $6.3 \, \text{mm}$ [S1].

### Strengths and Limitations
*   **Strengths:**
    *   Fast computation due to LUTs and parallel plane propagation.
    *   Handles occlusion naturally via inverted Gaussian filters.
    *   High-quality reconstructions with smooth surfaces due to Gaussian volume attribution.
    *   Wide field of view due to random phase scattering.
*   **Limitations:**
    *   HPO restricts viewing to horizontal parallax only, causing astigmatism and constrained vertical viewing zones.
    *   Requires pre-computation of LUTs, which consumes memory.
    *   BL-ASM requires careful handling of window sizes to avoid artifacts.

### When to Use
*   Use when generating holograms for HPO displays with high horizontal resolution.
*   Use when the scene is represented as a point cloud.
*   Use when occlusion handling is important.

### When Not to Use
*   Use full-parallax methods if vertical parallax is required.
*   Use direct point-to-hologram methods if the number of points is very small and LUT pre-computation is not feasible.

## Background

### Origin
The Multiple WRP method was originally proposed for fast computation of full-parallax holograms [S1]. It was later adapted for HPO displays to leverage the high horizontal resolution of modern light field displays.

### History
The method builds on the Angular Spectrum Method for wavefield propagation. The use of WRPs allows for the decomposition of complex 3D scenes into simpler 2D problems. The adaptation for HPO involves restricting the diffraction to 1D hololines, which reduces computational cost.

### Applications
*   Holographic displays with light field technology.
*   3D rendering of point clouds for holographic visualization.

## Sources

[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
