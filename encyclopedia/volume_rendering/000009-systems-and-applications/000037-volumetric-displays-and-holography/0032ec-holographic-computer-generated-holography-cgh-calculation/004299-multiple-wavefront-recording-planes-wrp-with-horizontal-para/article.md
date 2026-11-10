# Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)

Also called: Fast CGH method for point clouds, HPO WRP method

## Overview
This method generates computer-generated holograms (CGH) for point clouds by computing wavefields on multiple parallel Wavefront Recording Planes (WRPs) and projecting them onto a hologram plane, adapted specifically for Horizontal Parallax Only (HPO) displays. It solves the computational bottleneck of direct point-cloud CGH ($O(M \cdot N)$) by leveraging pre-computed Look-Up Tables (LUTs) and angular spectrum propagation, while inherently handling occlusion through inverted Gaussian filters. Unlike full-parallax WRP methods, this variant computes contributions to 1D hololines rather than 2D planes, reducing complexity for displays with constrained vertical resolution. It builds on [Holographic Computer-Generated Holography (CGH) Calculation](6884-holographic-computer-generated-holography-cgh-calculation.md) by optimizing the superposition process for specific hardware constraints.

## How it works
### Inputs and Outputs
**Inputs:**
*   Point cloud data: Set of $M$ points, each with 3D position $(x_i, y_i, z_i)$, color $(R_i, G_i, B_i)$, and intensity.
*   Display parameters: Pixel pitch $p_x, p_y$, resolution $N_x, N_y$, wavelength $\lambda$, and number of WRPs $K$.
*   HPO constraint: Vertical dimension is treated as a single line or minimal sampling; computation is performed along horizontal hololines.

**Outputs:**
*   Complex amplitude field on the hologram plane (or individual hololines for HPO).
*   Quantized phase map suitable for phase-only SLMs.

### Data Structures
1.  **Point Cloud:** Array of structures containing `position` (float3), `color` (float3), `intensity` (float).
2.  **Look-Up Table (LUT):** Pre-computed complex amplitude kernels $H_{LUT}(x, y)$ for a reference distance $z_{ref}$. Stored as complex arrays (real/imaginary pairs).
3.  **Wavefront Recording Planes (WRPs):** Array of $K$ parallel planes positioned between the object and hologram. Each WRP stores a complex field $U_k(x, y)$.
4.  **Hologram Plane:** 1D array (for HPO) or 2D array storing the final superposed complex amplitude.

### Numbered Steps

1.  **Initialize WRPs:** Create $K$ parallel planes perpendicular to the optical axis, spaced at intervals $z_k$. Initialize all WRP fields to zero.
2.  **Attribute Gaussian Volume to Points:** To generate smoother reconstructions, attribute a Gaussian distribution profile to each point $i$. The amplitude $A_i$ at a distance $r$ from the point center is modulated by a Gaussian kernel $G(r) = \exp(-r^2 / 2\sigma^2)$, where $\sigma$ controls the spread. This replaces the point source assumption with a volumetric source.
3.  **Generate LUTs:** Pre-compute the complex propagation kernel from a reference plane at distance $z_{ref}$ to the hologram plane (or WRP plane) using the Band-Limited Angular Spectrum Method (BL-ASM). Store these kernels in the LUT.
    *   *Note:* To account for diffuse reflections and wide field of view, modify the spectral response of the LUT content with a random phase mask $\Phi_{rand}(x,y)$, such that the kernel becomes $H_{LUT}' = H_{LUT} \cdot \exp(i \Phi_{rand})$.
4.  **Propagate Points to WRPs:** For each point $i$ in the point cloud:
    *   Determine the closest WRP $k$ to the point's $z$-coordinate $z_i$.
    *   Calculate the distance $d = |z_i - z_k|$.
    *   Retrieve the pre-computed LUT kernel corresponding to distance $d$ (or interpolate between LUT entries).
    *   Project the point's Gaussian-modulated wavefield onto the WRP $k$ using the LUT kernel. This involves shifting the kernel to the point's $(x_i, y_i)$ position and adding the complex amplitude to the WRP field.
    *   *Occlusion Handling:* Use inverted Gaussian filters during propagation. If a point is occluded by another point closer to the hologram, its contribution to the WRP is locally blocked or attenuated. This is modeled by applying a mask derived from the occluding point's Gaussian profile to the contributing point's field on the WRP.
5.  **Inter-WRP Propagation:** Propagate the complex fields from all WRPs to the hologram plane (or to the next WRP in the chain, depending on the specific WRP architecture).
    *   Use the Angular Spectrum (AS) method for all wavefield propagations between parallel planes.
    *   Specifically, use the Band-Limited Angular Spectrum Method (BL-ASM) to handle small sampling pitches and aperiodic functions.
6.  **Superposition for HPO:** For HPO, compute the contribution of the points to hololines. Instead of summing to a 2D plane, sum the propagated fields along the horizontal direction for each vertical line (or single vertical line if strictly 1D). The final hologram field $U_h(x)$ is the sum of contributions from all WRPs and points, integrated only over the horizontal parallax dimension.
7.  **Reconstruction/Output:** Extract the phase $\phi(x) = \arg(U_h(x))$ from the final complex field. Quantize to $0 \dots 2\pi$ for phase-only SLMs.

### Formulas

**Grating Equation (Max Diffraction Angle):**
$$ \sin \theta = \frac{\lambda}{2p} $$
where $\theta$ is the maximum angle of diffraction, $\lambda$ is wavelength, and $p$ is pixel pitch [S1].

**Minimum Object-Hologram Distance (Shannon Criterion):**
$$ z_{min} = \frac{N p^2}{\lambda} $$
where $N$ is pixel resolution (square dimensions), $p$ is pixel pitch, and $\lambda$ is wavelength [S1].

**2D Propagation in Spatial Domain:**
$$ g(x, y, z) = g(x, y, 0) * h(x, y, z) $$
where $g(x,y,0)$ is the source field, $h(x,y,z)$ is the propagation kernel, and $*$ denotes convolution [S1].

**Propagation Kernel:**
$$ h(x, y, z) = \frac{\exp(i 2\pi r/\lambda)}{r} \left( \frac{1}{r} + \frac{1}{i 2\pi \lambda} \right) $$
where $r = \sqrt{x^2 + y^2 + z^2}$ and $\lambda$ is wavelength [S1].

**2D Propagation in Frequency Domain:**
$$ G(u, v; z) = G(u, v; 0) H(u, v; z) $$
where $G$ is the spectrum, $H$ is the transfer function, and $F$ represents the Fourier transform [S1].

**Transfer Function Spatial Frequency $w$:**
$$ w = w(u, v) = \begin{cases} (\lambda^{-2} - u^2 - v^2)^{1/2} & u^2 + v^2 \le \lambda^{-2} \\ 0 & \text{otherwise} \end{cases} $$
where $u, v$ are Fourier frequencies in x, y directions [S1].

**Band-Limited Transfer Function:**
$$ H'(u, v; z) = H(u, v; z) \text{rect}\left(\frac{u}{2u_{limit}}\right) \text{rect}\left(\frac{v}{2v_{limit}}\right) $$
where $2u_{limit}$ and $2v_{limit}$ are the bandwidth clipping limits [S1].

### Parameters
*   **Display:** Holografika HoloVizio 722RC light field display [S1].
*   **Viewing Angle:** 70° [S1].
*   **Hologram Resolution:** 32,768 (horizontal) by 768 (vertical) pixels [S1].
*   **Pixel Pitch:** 0.55 µm (horizontal), 23.4 µm (vertical) [S1].
*   **Wavelength:** 640 nm [S1].
*   **Minimum Distance:** 6.3 mm (calculated from Eq. 2) [S1].

### Edge Cases and Pitfalls
*   **Small Sampling Pitch:** Leads to strong diffusion of the light field beyond computation window borders in the output plane. The SBP of the hologram is not supported by standard AS propagation due to low bandwidth in the Fourier domain [S1].
*   **Aperiodic Functions:** Cause errors at the edges of the computation window due to circular convolution in FFT. This is handled by BL-ASM [S1].
*   **BL-ASM Implementation:** To mitigate edge errors, double the area of the calculation window in the source plane and pad additional sampling points as zeros. After calculation, crop the output window to the original window size at the destination plane [S1].
*   **Color Holograms:** Computed for the same wavelength (e.g., red) to maintain wide viewing angle, but with 3 different intensities per point for R, G, B channels [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def band_limited_angular_spectrum(field, dz, wavelength, pixel_pitch, window_size):
    """
    Propagates a field using BL-ASM.
    field: 2D complex array (source plane)
    dz: propagation distance
    wavelength: float
    pixel_pitch: float
    window_size: tuple (width, height) of original window
    """
    # Double the window size for padding
    padded_shape = (window_size[0] * 2, window_size[1] * 2)
    padded_field = np.zeros(padded_shape, dtype=complex)
    padded_field[:window_size[0], :window_size[1]] = field
    
    # FFT to frequency domain
    G = np.fft.fft2(padded_field)
    
    # Create frequency grid
    u = np.fft.fftfreq(padded_shape[0], d=pixel_pitch)
    v = np.fft.fftfreq(padded_shape[1], d=pixel_pitch)
    U, V = np.meshgrid(u, v)
    
    # Calculate spatial frequency w
    w = np.sqrt(np.maximum(0, (1/wavelength)**2 - U**2 - V**2))
    
    # Transfer function
    H = np.exp(1j * 2 * np.pi * w * dz)
    
    # Band-limiting: define limits based on Nyquist
    # u_limit and v_limit are derived from the original window's bandwidth
    # For simplicity, assume limits are set to avoid aliasing for the original resolution
    u_limit = 1 / (2 * pixel_pitch)
    v_limit = 1 / (2 * pixel_pitch)
    
    # Apply rect function for band-limiting
    H = H * np.sinc(2 * u_limit * U) * np.sinc(2 * v_limit * V) # Approximation of rect in freq domain or direct multiplication in freq domain?
    # Correction: The source says H' = H * rect(u/2u_limit) * rect(v/2v_limit)
    # rect(x) is 1 if |x|<=0.5, else 0. So rect(u/2u_limit) is 1 if |u| <= u_limit.
    mask = (np.abs(U) <= u_limit) & (np.abs(V) <= v_limit)
    H = H * mask
    
    # Inverse FFT
    propagated_padded = np.fft.ifft2(G * H)
    
    # Crop to original window size
    cropped_field = propagated_padded[:window_size[0], :window_size[1]]
    
    return cropped_field

def generate_hpo_cgh(point_cloud, wrp_positions, hologram_resolution, wavelength, pixel_pitch):
    """
    Generates HPO CGH using WRP method.
    """
    # Initialize WRPs
    # Assuming K WRPs
    K = len(wrp_positions)
    wrp_fields = [np.zeros(hologram_resolution, dtype=complex) for _ in range(K)]
    
    # Pre-compute LUTs for distances from points to WRPs
    # LUT stores complex kernels for various distances
    lut_distances = [...] # List of distances
    lut_kernels = [...] # List of complex arrays
    
    # Process each point
    for point in point_cloud:
        pos = point.position
        color = point.color
        intensity = point.intensity
        
        # Find closest WRP
        closest_wrp_idx = np.argmin([abs(pos.z - z) for z in wrp_positions])
        dz = abs(pos.z - wrp_positions[closest_wrp_idx])
        
        # Get LUT kernel for this distance
        kernel = get_lut_kernel(dz, lut_distances, lut_kernels)
        
        # Apply random phase for diffuse reflection
        random_phase = np.exp(1j * np.random.uniform(0, 2*np.pi, hologram_resolution))
        kernel = kernel * random_phase
        
        # Project point to WRP
        # Shift kernel to point's x,y position
        # Add to WRP field
        # Note: For HPO, this might be 1D projection or 2D with vertical averaging
        # Assuming 2D projection for now, then summing for HPO
        wrp_fields[closest_wrp_idx] += project_point_to_wrp(kernel, pos.x, pos.y, intensity, color)
        
    # Propagate WRPs to Hologram Plane
    hologram_field = np.zeros(hologram_resolution, dtype=complex)
    for i in range(K):
        propagated = band_limited_angular_spectrum(wrp_fields[i], 
                                                   hologram_resolution.z - wrp_positions[i], 
                                                   wavelength, pixel_pitch, hologram_resolution)
        hologram_field += propagated
        
    # HPO Summation: Sum along vertical axis (or keep only horizontal lines)
    # For strict HPO, we might only compute 1D hololines
    hpo_hologram = np.sum(hologram_field, axis=0) # Sum over vertical dimension
    
    return hpo_hologram
```

## Subtopics

### Full Parallax WRP
Computes contributions to 2D planes. Chosen when full 3D parallax is required and computational resources allow. [Link to sibling if available, otherwise note distinction].

### Holographic Stereogram
Uses multiple 2D holograms viewed from different angles. Chosen for simpler hardware but limited continuous parallax. [Link to sibling](7067-holographic-stereogram.md).

### Volume Rendering based CGH
Directly renders volumetric data. Chosen for dense volumetric scenes rather than point clouds. [Link to sibling](808-volume-rendering-based-cgh-calculation.md).

## Operating characteristics
*   **Time Complexity:** Significantly reduced compared to direct calculation. Complexity depends on the number of points $M$, number of WRPs $K$, and hologram resolution $N$. The use of LUTs reduces the per-point cost to $O(1)$ for kernel retrieval and $O(N \log N)$ for FFT-based propagation. Total complexity is roughly $O(M \cdot K \cdot N \log N)$ or better depending on optimization.
*   **Memory Complexity:** Requires storage for $K$ WRP fields and LUTs. LUT size depends on the range of distances and resolution.
*   **Measured Performance:** Demonstrated on Holografika HoloVizio 722RC with 32,768x768 resolution, 0.55 µm pitch, 640 nm wavelength. Achieved 70° viewing angle. Minimum object distance 6.3 mm [S1].
*   **Strengths:** Handles occlusion naturally; high-quality smooth reconstructions via Gaussian attribution; wide field of view via random phase scattering; efficient for point clouds.
*   **Limitations:** HPO restricts vertical parallax; requires pre-computation of LUTs; sensitive to sampling pitch (requires BL-ASM); color requires multi-wavelength or intensity modulation at single wavelength.
*   **When to Use:** Point cloud rendering for HPO light field displays requiring wide viewing angles and smooth surfaces.
*   **When Not to Use:** Full parallax requirements without sufficient computational power; dense volumetric data better suited for volume rendering CGH.

## Background
The method originates from research on fast CGH generation for point clouds using multiple wavefront recording planes. It was adapted for HPO displays to leverage specific hardware like the HoloVizio light field display. The core WRP technique was previously proposed for full parallax [S1, ref 15]. The adaptation for HPO involves computing contributions to 1D hololines. The use of BL-ASM addresses the limitations of standard AS propagation for small pixel pitches [S1].

## Sources
[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
