# Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)

Also called: Fast CGH method for point clouds, HPO WRP method

## Overview
This method generates computer-generated holograms (CGH) for point clouds by computing wavefield propagation across multiple parallel planes, adapted for horizontal parallax only (HPO) displays. It solves the computational bottleneck of full-parallax CGH by restricting diffraction to 1D hololines, while handling occlusion and diffuse reflections through Gaussian volume attribution and inverted filters. It is distinguished from full-parallax WRP methods by its 1D propagation geometry and from single-plane methods by its ability to handle deep scenes with occlusion via intermediate recording planes.

## How it works

### Inputs and Outputs
**Inputs:**
*   Point cloud data: Set of points $\mathcal{P} = \{(\mathbf{r}_i, A_i, \mathbf{c}_i)\}$, where $\mathbf{r}_i$ is the 3D position, $A_i$ is the complex amplitude, and $\mathbf{c}_i$ is the RGB color vector.
*   Display parameters: Pixel pitch $p_x, p_y$, resolution $N_x, N_y$, wavelength $\lambda$, and number of Wavefront Recording Planes (WRPs) $M$.
*   Propagation distance $z$ between adjacent WRPs.

**Outputs:**
*   Hologram plane field $U_{holo}(x, y)$, typically phase-only or complex-valued, ready for Spatial Light Modulator (SLM) loading.

### Data Structures
1.  **Point Cloud:** List of points with 3D coordinates $(x_i, y_i, z_i)$, complex amplitude $A_i$, and RGB intensities $(R_i, G_i, B_i)$.
2.  **WRP Grid:** $M$ parallel planes perpendicular to the optical axis (z-axis). Each WRP is a 2D grid of complex values. For HPO, the hologram plane is effectively a set of 1D hololines, but the intermediate WRPs are 2D to support the 2D Angular Spectrum propagation between planes.
3.  **Look-Up Tables (LUTs):** Pre-computed propagation kernels $h(x, y, z)$ and transfer functions $H(u, v, z)$ for the specific distance $z$ and wavelength $\lambda$.

### Steps

1.  **Point Volume Attribution:**
    Assign a Gaussian distribution profile to each point to model a continuous volume rather than a singularity. This smooths the reconstruction surface. The amplitude at a position $\mathbf{r}$ from point $i$ at $\mathbf{r}_i$ is modulated by a Gaussian kernel $G(\mathbf{r} - \mathbf{r}_i)$.

2.  **LUT Generation (Band-Limited Angular Spectrum):**
    Compute the propagation kernel and transfer function using the Band-Limited Angular Spectrum Method (BL-ASM) to handle small sampling pitches and aperiodic functions.
    *   Calculate the spatial frequency $w(u, v)$:
        $$ w(u, v) = \begin{cases} (\lambda^{-2} - u^2 - v^2)^{1/2} & u^2 + v^2 \le \lambda^{-2} \\ 0 & \text{otherwise} \end{cases} $$
    *   Define the transfer function $H(u, v; z) = \exp(i 2\pi w z)$.
    *   Apply bandwidth limiting to prevent aliasing and edge errors from circular convolution:
        $$ H'(u, v; z) = H(u, v; z) \cdot \text{rect}\left(\frac{u}{2u_{limit}}\right) \cdot \text{rect}\left(\frac{v}{2v_{limit}}\right) $$
    *   Store $H'(u, v; z)$ in the LUT.

3.  **Initialize WRPs:**
    Initialize the first WRP (closest to the object) with zero field.

4.  **Object to First WRP Propagation:**
    Compute the contribution of each point in the point cloud to the first WRP.
    *   For HPO, compute the contribution to 1D hololines. However, since intermediate propagation is 2D, the initial field on the first WRP is computed by summing the spherical wave contributions from all points, restricted to the WRP plane.
    *   Apply random phase to the spectral response of the LUT content to account for diffuse reflections, scattering light over a wide angular range [S1].

5.  **Inter-WRP Propagation:**
    Propagate the field from WRP $k$ to WRP $k+1$ using the Angular Spectrum method.
    *   Perform FFT of the field on WRP $k$: $G_k(u, v) = \mathcal{F}\{U_k(x, y)\}$.
    *   Multiply by the pre-computed transfer function: $G_{k+1}(u, v) = G_k(u, v) H'(u, v; z)$.
    *   Perform Inverse FFT to get the field on WRP $k+1$: $U_{k+1}(x, y) = \mathcal{F}^{-1}\{G_{k+1}(u, v)\}$.
    *   **BL-ASM Padding:** To handle the strong diffusion of the light field and aperiodic function errors, double the area of the calculation window in the source plane by padding zeros. After calculation, crop the output window to the original window size at the destination plane [S1].

6.  **Occlusion Handling:**
    Use inverted Gaussian filters in the propagation process to model occlusions. As the field propagates from WRP $k$ to $k+1$, apply a filter that locally blocks light based on the depth of points in the point cloud relative to the propagation path. This ensures that points behind other points do not contribute to the field [S1].

7.  **Final Hologram Generation:**
    Propagate the field from the last WRP to the hologram plane (SLM plane).
    *   For HPO, the final hologram consists of 1D hololines. The contribution of each point to the hologram is computed only along the horizontal line corresponding to the point's vertical position.
    *   Sum the contributions from all points to form the complex hologram field $U_{holo}(x, y)$.

8.  **Color Processing:**
    For color holograms, compute the hologram for the red channel wavelength $\lambda_R$ using the RGB intensities $(R_i, G_i, B_i)$ scaled appropriately. The final hologram consists of three channels, all computed for the same wavelength (e.g., red) to maintain a wide viewing angle, but with different intensities per point for each channel [S1].

9.  **Phase Quantization:**
    Convert the complex hologram field $U_{holo}(x, y)$ to phase values $\phi_h \in [0, 2\pi)$ for the phase-only SLM. This can be done using detour phase or other phase-only encoding techniques.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_hpo_wrp_cgh(points, display_params, wrp_params):
    """
    Generate HPO CGH using Multiple Wavefront Recording Planes.
    
    Args:
        points: List of dicts with keys 'x', 'y', 'z', 'R', 'G', 'B', 'A'
        display_params: Dict with 'px', 'py', 'Nx', 'Ny', 'lambda', 'z_min'
        wrp_params: Dict with 'M' (number of WRPs), 'z_step' (distance between WRPs)
    
    Returns:
        hologram: 2D array of phase values
    """
    lambda_w = display_params['lambda']
    px = display_params['px']
    py = display_params['py']
    Nx = display_params['Nx']
    Ny = display_params['Ny']
    M = wrp_params['M']
    z_step = wrp_params['z_step']
    
    # 1. Initialize WRPs
    # WRPs are 2D grids. HPO reduces final output to 1D, but intermediate steps are 2D.
    wrp_fields = [np.zeros((Ny, Nx), dtype=complex) for _ in range(M)]
    
    # 2. Generate LUTs using BL-ASM
    # Create frequency grids
    u = np.fft.fftfreq(Nx, d=px)
    v = np.fft.fftfreq(Ny, d=py)
    U, V = np.meshgrid(u, v)
    
    # Calculate spatial frequency w
    w = np.sqrt(np.maximum(lambda_w**-2 - U**2 - V**2, 0))
    
    # Transfer function
    H = np.exp(1j * 2 * np.pi * w * z_step)
    
    # Band-limiting
    # Determine limits based on Nyquist and display constraints
    # Not given in sources: exact calculation of u_limit, v_limit. 
    # Typically related to the maximum diffraction angle.
    # For implementation, we assume limits are provided or calculated from display_params.
    u_limit = 1 / (2 * px) # Example limit
    v_limit = 1 / (2 * py)
    
    H_bl = H * np.sinc(2 * u_limit * U) * np.sinc(2 * v_limit * V) # rect function approximated by sinc in freq domain? 
    # Note: rect in freq domain corresponds to sinc in spatial, but here we multiply in freq domain.
    # rect(u/2u_limit) is 1 if |u| < u_limit, 0 otherwise.
    H_bl = H * ((np.abs(U) <= u_limit) & (np.abs(V) <= v_limit))
    
    # 3. Propagate from Object to First WRP
    # For HPO, we compute contributions to hololines. 
    # However, the first WRP is a 2D plane. We sum contributions from all points.
    for point in points:
        # Gaussian volume attribution
        # Not given in sources: exact Gaussian width. [G] Assume sigma is a parameter.
        sigma = 1.0 # Placeholder for Gaussian width
        dx = np.arange(Nx) * px - point['x']
        dy = np.arange(Ny) * py - point['y']
        X, Y = np.meshgrid(dx, dy)
        dist_sq = X**2 + Y**2 + point['z']**2
        dist = np.sqrt(dist_sq)
        
        # Spherical wave propagation with Gaussian envelope
        # Amplitude A_i / r * exp(i k r) * Gaussian
        gaussian_envelope = np.exp(-dist_sq / (2 * sigma**2))
        phase = 2 * np.pi * dist / lambda_w
        amplitude = point['A'] / dist * gaussian_envelope
        
        # Add to first WRP
        wrp_fields[0] += amplitude * np.exp(1j * phase)
        
    # 4. Inter-WRP Propagation
    for k in range(M - 1):
        # BL-ASM: Double the window size in source plane
        # Pad zeros to double size
        padded_field = np.zeros((2*Ny, 2*Nx), dtype=complex)
        padded_field[:Ny, :Nx] = wrp_fields[k]
        
        # FFT
        F_padded = np.fft.fft2(padded_field)
        
        # Interpolate or pad H_bl to match padded size? 
        # Not given in sources: exact interpolation of H_bl for doubled grid. [G] 
        # Assume H_bl is interpolated or recalculated for the new grid spacing.
        # For simplicity, we assume H_bl is available for the padded grid.
        # In practice, u and v grids must be recalculated for the padded grid.
        u_pad = np.fft.fftfreq(2*Nx, d=px/2) # Pixel pitch effectively halved in spatial domain? 
        # No, padding adds zeros, so the physical size doubles, but the sampling interval remains px.
        # So the frequency resolution halves.
        u_pad = np.fft.fftfreq(2*Nx, d=px)
        v_pad = np.fft.fftfreq(2*Ny, d=py)
        U_pad, V_pad = np.meshgrid(u_pad, v_pad)
        
        w_pad = np.sqrt(np.maximum(lambda_w**-2 - U_pad**2 - V_pad**2, 0))
        H_pad = np.exp(1j * 2 * np.pi * w_pad * z_step)
        
        # Band-limiting for padded grid
        # Limits scale with the new grid? 
        # Not given in sources. [G] Assume same physical limits.
        H_pad_bl = H_pad * ((np.abs(U_pad) <= u_limit) & (np.abs(V_pad) <= v_limit))
        
        # Multiply
        F_out = F_padded * H_pad_bl
        
        # IFFT
        out_padded = np.fft.ifft2(F_out)
        
        # Crop to original size
        wrp_fields[k+1] = out_padded[:Ny, :Nx]
        
        # 5. Occlusion Handling
        # Apply inverted Gaussian filters to block light from points behind others.
        # Not given in sources: exact occlusion filter implementation. [G]
        # Assume a depth-based mask is applied.
        # For each point, if it is behind another point relative to the propagation direction,
        # its contribution is attenuated. This is complex to implement in a field-based method.
        # The source mentions "inverted Gaussian filters... to model occlusions".
        # This likely refers to a pre-computed mask or a filter applied to the field.
        # Without specific details, we skip the exact occlusion filter application.
        
    # 6. Final Propagation to Hologram Plane
    # For HPO, the hologram is 1D hololines.
    # Propagate from last WRP to hologram plane.
    # The hologram plane has resolution Nx x Ny, but we only care about the horizontal lines.
    
    hologram_field = np.zeros((Ny, Nx), dtype=complex)
    
    for point in points:
        # Compute contribution to the hologram plane
        # HPO: only diffract along the line y = point['y']
        # So we only compute the field at y = point['y'] for all x.
        
        dx = np.arange(Nx) * px - point['x']
        dy = 0 # Relative to the hololine
        dist = np.sqrt(dx**2 + dy**2 + point['z_holo']**2) # z_holo is distance from last WRP to hologram
        
        # Propagation from last WRP to hologram
        # This is a single step propagation.
        # Use BL-ASM for this step as well.
        
        # Not given in sources: exact distance from last WRP to hologram. [G] Assume z_holo is a parameter.
        z_holo = wrp_params.get('z_holo', 100e-3) # 100mm example
        
        # Recalculate H for z_holo
        u_h = np.fft.fftfreq(Nx, d=px)
        v_h = np.fft.fftfreq(Ny, d=py)
        U_h, V_h = np.meshgrid(u_h, v_h)
        w_h = np.sqrt(np.maximum(lambda_w**-2 - U_h**2 - V_h**2, 0))
        H_holo = np.exp(1j * 2 * np.pi * w_h * z_holo)
        H_holo_bl = H_holo * ((np.abs(U_h) <= u_limit) & (np.abs(V_h) <= v_limit))
        
        # FFT of last WRP
        F_last = np.fft.fft2(wrp_fields[-1])
        
        # Multiply
        F_out_holo = F_last * H_holo_bl
        
        # IFFT
        field_holo = np.fft.ifft2(F_out_holo)
        
        # Extract the hololine for this point
        # The point's y-coordinate maps to a specific row in the hologram
        y_idx = int(point['y'] / py)
        if 0 <= y_idx < Ny:
            hologram_field[y_idx, :] += field_holo[y_idx, :] * point['A'] * np.exp(1j * 2 * np.pi * dist / lambda_w)
            
    # 7. Color Processing
    # Compute for Red channel, scale by R, G, B intensities
    # Not given in sources: exact color mixing. [G] Assume separate holograms for R, G, B.
    # For simplicity, we return the complex field for the red channel.
    
    # 8. Phase Quantization
    phase_holo = np.angle(hologram_field)
    
    return phase_holo
```

### Parameters
*   **Pixel Pitch:** $p_x = 0.55 \, \mu\text{m}$ (horizontal), $p_y = 23.4 \, \mu\text{m}$ (vertical) [S1].
*   **Resolution:** $N_x = 32,768$, $N_y = 768$ [S1].
*   **Wavelength:** $\lambda = 640 \, \text{nm}$ [S1].
*   **Viewing Angle:** $70^\circ$ (implies $2\theta = 70^\circ$) [S1].
*   **Minimum Distance:** $z_{min} = \frac{N p^2}{\lambda} = 6.3 \, \text{mm}$ [S1].

### Edge Cases and Pitfalls
*   **Small Sampling Pitch:** Leads to strong diffusion of the light field beyond the computation window borders. This is handled by BL-ASM [S1].
*   **Aperiodic Functions:** Cause errors at the edges of the computation window due to circular convolution in FFT. BL-ASM with zero-padding and cropping mitigates this [S1].
*   **HPO Astigmatism:** HPO causes constrained viewing zones and astigmatic output due to diminished spatial resolution in the vertical direction [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** $O(M \cdot N_x N_y \log(N_x N_y))$ for $M$ WRPs and resolution $N_x \times N_y$. The BL-ASM adds overhead due to padding and interpolation.
*   **Memory Complexity:** $O(M \cdot N_x N_y)$ to store the WRPs. LUTs require $O(N_x N_y)$ storage.

### Measured Performance
*   **Hardware:** Holografika HoloVizio 722RC light field display [S1].
*   **Resolution:** 32,768-by-768 pixels [S1].
*   **Viewing Angle:** $71.1^\circ$ achieved with the specified parameters [S1].

### Strengths and Limitations
*   **Strengths:** Fast computation for point clouds compared to full-parallax methods. Handles occlusion and diffuse reflections. Suitable for HPO displays with wide viewing angles.
*   **Limitations:** HPO restricts vertical parallax, leading to astigmatism and constrained viewing zones. Requires high-resolution SLMs for small pixel pitches.

### When to Use
*   Use for HPO holographic displays with point cloud data.
*   Use when occlusion and diffuse reflections are important.
*   Do not use for full-parallax displays.

## Background

### Origin
The Multiple WRP method was originally proposed for fast computation of full-parallax holograms [S1]. It was adapted for HPO by computing contributions to 1D hololines.

### History
The method is based on the Angular Spectrum propagation method and uses Look-Up Tables (LUTs) for efficient computation. The BL-ASM variant was introduced to handle the challenges of small sampling pitches [S1].

### Applications
Used in holographic light field displays, such as the Holografika HoloVizio 722RC [S1].

## Sources

[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
