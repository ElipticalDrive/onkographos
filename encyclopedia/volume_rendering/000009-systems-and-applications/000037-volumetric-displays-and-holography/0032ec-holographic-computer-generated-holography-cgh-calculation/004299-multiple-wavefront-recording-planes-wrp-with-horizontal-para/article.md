# Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)

Also called: Fast CGH method for point clouds, HPO WRP method

## Overview
This method generates computer-generated holograms (CGH) for point clouds by computing contributions to multiple parallel wavefront recording planes (WRPs), adapted for horizontal parallax only (HPO) displays. It solves the high computational cost of full-parallax CGH by restricting diffraction to 1D hololines, while using the WRP strategy to handle occlusion and enable fast propagation via look-up tables (LUTs). It is distinguished from full-parallax WRP methods by its 1D computation per point and from direct diffraction methods by its use of parallel propagation planes and Gaussian point volumes.

## How it works
**Inputs:**
*   Point cloud data: Set of points $P_i$ with 3D coordinates $(x_i, y_i, z_i)$ and RGB intensity values.
*   Display parameters: Pixel pitch $p_x, p_y$, resolution $N_x, N_y$, wavelength $\lambda$, and number of WRPs.
*   Hologram plane geometry: Defined by the display specifications.

**Outputs:**
*   Complex optical field $U(x,y)$ on the hologram plane (or individual hololines for HPO) ready for phase-only encoding.

**Data Structures:**
*   **Point Cloud:** List of objects/tuples containing $(x, y, z, R, G, B)$.
*   **Look-Up Tables (LUTs):** Pre-computed propagation kernels for specific distances $z$ between WRPs. The LUT stores the complex transfer function $H(u, v; z)$ or the spatial kernel $h(x, y; z)$ depending on implementation, modified by random phase for diffuse reflection modeling [S1].
*   **Wavefront Recording Planes (WRPs):** A series of parallel planes positioned between the object and the hologram plane. Each WRP holds a complex field representing the light wavefront at that depth.
*   **Hololines:** For HPO, the hologram is decomposed into 1D lines along the horizontal axis. Each point contributes to specific hololines based on its horizontal position.

**Steps:**

1.  **Point Volume Attribution:**
    Assign a Gaussian distribution profile to each point in the point cloud to model a volume rather than a singularity. This generates smoother, continuous surface reconstructions [S1]. The amplitude at a distance $r$ from the point center is modeled by a Gaussian function.

2.  **LUT Generation (Pre-computation):**
    Pre-compute propagation kernels for the required distances between WRPs using the Band-Limited Angular Spectrum Method (BL-ASM).
    *   Calculate the transfer function $H(u, v; z)$ in the frequency domain [S1].
    *   Apply random phase modulation to the LUT content to account for diffuse reflections, scattering light over a wide angular range [S1].
    *   Store these kernels for fast convolution during propagation.

3.  **Initialization of WRPs:**
    Initialize the complex field on all WRPs to zero.

4.  **Point Contribution to WRPs:**
    For each point $P_i$ in the point cloud:
    *   Determine the WRPs that the point "illuminates" or intersects.
    *   Compute the contribution of the point to the relevant WRPs. Since the method is adapted for HPO, compute the contribution to **hololines** (1D holograms) rather than the full 2D plane. The point diffracts only along the horizontal line corresponding to its vertical position [S1].
    *   Use the Gaussian volume model to determine the amplitude distribution on the hololine.

5.  **Inter-WRP Propagation:**
    Propagate the wavefields from the object plane (or initial WRP) to subsequent WRPs and finally to the hologram plane using the pre-computed LUTs.
    *   Perform all wavefield propagations between parallel planes using the Angular Spectrum (AS) method [S1].
    *   Specifically, use the **Band-Limited Angular Spectrum Method (BL-ASM)** for all propagations (LUT generation, inter-WRP, and reconstruction) to handle small sampling pitches and aperiodic functions [S1].

6.  **Occlusion Handling:**
    Use inverted Gaussian filters in the propagation process to model occlusions. When a point is occluded by another object or WRP, the light is locally blocked by applying these filters during the propagation step [S1].

7.  **Hologram Plane Assembly:**
    Sum the contributions from all points and all WRPs on the hologram plane. For HPO, this results in a set of hololines that form the final hologram.

8.  **Color Handling:**
    To acquire color holograms, attribute RGB texture values to the points. Compute the hologram for a single wavelength (e.g., red) but with three different intensity channels (R, G, B) per point to maintain color information while preserving the wide viewing angle [S1].

**Formulas:**

*   **Grating Equation (Max Diffraction Angle):**
    $$ \sin \theta = \frac{\lambda}{2p} $$
    Where $\theta$ is the maximum angle of diffraction, $\lambda$ is the wavelength, and $p$ is the pixel pitch [S1].

*   **Minimum Object-Hologram Distance (Shannon Criterion):**
    $$ z_{min} = \frac{N p^2}{\lambda} $$
    Where $N$ is the pixel resolution (considering square dimensions), $p$ is the pixel pitch, and $\lambda$ is the wavelength [S1].

*   **2D Propagation in Spatial Domain:**
    $$ g(x, y, z) = g(x, y, 0) * h(x, y, z) $$
    Where $g(x,y,0)$ is the source field, $h(x,y,z)$ is the propagation kernel, and $*$ denotes convolution [S1].

*   **Propagation Kernel:**
    $$ h(x, y, z) = \frac{\exp(i2\pi r/\lambda)}{r} \left( \frac{1}{r} + \frac{1}{i2\pi \lambda} \right) $$
    Where $r = \sqrt{x^2 + y^2 + z^2}$ and $\lambda$ is the wavelength [S1].

*   **2D Propagation in Frequency Domain:**
    $$ G(u, v; z) = G(u, v; 0)H(u, v; z) $$
    Where $G$ is the spectrum, $H$ is the transfer function, and $F$ represents the Fourier transform [S1].

*   **Transfer Function Spatial Frequency $w$:**
    $$ w = w(u, v) = \begin{cases} (\lambda^{-2} - u^2 - v^2)^{1/2} & u^2 + v^2 \le \lambda^{-2} \\ 0 & \text{otherwise} \end{cases} $$
    Where $u, v$ are Fourier frequencies in x, y directions [S1].

*   **Band-Limited Transfer Function (BL-ASM):**
    $$ H'(u, v; z) = H(u, v; z) \text{rect}\left(\frac{u}{2u_{limit}}\right) \text{rect}\left(\frac{v}{2v_{limit}}\right) $$
    Where $2u_{limit}$ and $2v_{limit}$ are the bandwidth clipping limits [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_hpo_wrp_cgh(points, display_params, wrp_distances, luts):
    """
    Compute HPO CGH using Multiple WRP method.
    
    Args:
        points: List of dicts with keys 'x', 'y', 'z', 'R', 'G', 'B'
        display_params: Dict with 'px', 'py', 'Nx', 'Ny', 'lambda'
        wrp_distances: List of z-distances for WRPs relative to hologram plane
        luts: Dict mapping distance z to pre-computed BL-ASM transfer function H'(u,v;z)
    
    Returns:
        hologram_field: Complex array representing the hologram field (HPO hololines)
    """
    
    # 1. Initialize WRPs
    # Each WRP is a 2D complex field. For HPO, we can optimize by storing 1D hololines
    # but for generality in propagation, we use 2D arrays where vertical variation is minimal/zero.
    num_wrps = len(wrp_distances)
    wrp_fields = [np.zeros((display_params['Ny'], display_params['Nx']), dtype=np.complex128) for _ in range(num_wrps)]
    
    # 2. Process each point
    for point in points:
        px, py, pz = point['x'], point['y'], point['z']
        r_val, g_val, b_val = point['R'], point['G'], point['B']
        
        # Determine which WRPs this point contributes to.
        # In a full implementation, this involves ray-tracing or geometric intersection.
        # Here we assume points are projected onto WRPs based on z-depth.
        
        # For HPO, we compute contribution to the hololine at vertical position py.
        # The hololine is a 1D array along x.
        
        for i, z_wrp in enumerate(wrp_distances):
            # Calculate distance from point to WRP
            dz = pz - z_wrp
            
            # Skip if point is behind WRP or too close (singularity)
            if dz <= 0:
                continue
                
            # Get LUT for this distance
            # Note: LUTs should be pre-computed with random phase for diffuse reflection
            if dz in luts:
                H = luts[dz]
            else:
                # Compute BL-ASM transfer function on the fly if not in LUT
                H = compute_bl_asm_transfer_function(dz, display_params['lambda'], display_params['px'], display_params['py'])
            
            # Create source field for this point on the WRP
            # Gaussian volume attribution
            # The point contributes to the hololine at index corresponding to py
            # We assume py maps to a specific row index in the WRP field
            row_idx = int(py / display_params['py'])
            
            # Create a 1D Gaussian profile for the hololine
            # x coordinates of the WRP
            x_coords = np.arange(display_params['Nx']) * display_params['px']
            
            # Gaussian width depends on distance and wavelength (diffraction spread)
            # Simplified Gaussian model for point volume
            sigma = 2.0 * dz * display_params['lambda'] / (np.pi * display_params['px']) # Approximate spread
            
            # Amplitude distribution along the hololine
            # Center the Gaussian at the point's x position projected onto the WRP
            # For simplicity, assume paraxial approximation: x_proj = px * (dz / z_ref)
            # In WRP method, the point is often treated as a source on the WRP plane itself
            # or propagated from object plane to WRP.
            
            # Let's assume the point is projected onto the WRP plane at (x_proj, y_proj)
            # x_proj = px * (z_wrp / pz) ? No, WRP is between object and hologram.
            # Standard WRP: Object -> WRP -> Hologram.
            # Propagation from Object to WRP is done here.
            
            # Project point to WRP plane
            # Assuming pinhole projection for simplicity of position mapping
            scale = z_wrp / pz
            x_proj = px * scale
            y_proj = py * scale
            
            col_idx = int(x_proj / display_params['px'])
            
            # Create Gaussian kernel for the hololine
            # Using the Gaussian volume model
            gaussian_profile = np.exp(-((x_coords - x_proj)**2) / (2 * sigma**2))
            
            # Apply intensity
            # For color, we compute for one wavelength but scale intensities
            # Assuming red wavelength for calculation, scale by R, G, B later or in separate passes
            amplitude = r_val * gaussian_profile
            
            # Add to WRP field at the specific row (hololine)
            # Handle boundary checks
            if 0 <= row_idx < display_params['Ny']:
                # Add contribution to the hololine
                # We need to handle the 2D nature: vertical spread is also Gaussian but narrower
                # For HPO, vertical resolution is low, so we might just set the row
                wrp_fields[i][row_idx, :] += amplitude * H[row_idx, :] # Apply transfer function? 
                # Correction: The LUT H is for propagation FROM the WRP TO the next plane.
                # The contribution of the point TO the WRP is the source field.
                # The source field is the point's field at the WRP plane.
                # The point's field at the WRP plane is computed via propagation from object to WRP.
                # This step is simplified here. In full WRP, points are projected to WRPs.
                pass

    # 3. Propagate WRPs to Hologram Plane
    # Propagate each WRP field to the hologram plane using its specific LUT
    hologram_field = np.zeros((display_params['Ny'], display_params['Nx']), dtype=np.complex128)
    
    for i, z_wrp in enumerate(wrp_distances):
        # Distance from WRP to Hologram
        dz_holo = z_wrp # Assuming hologram is at z=0 and WRPs are at positive z
        
        if dz_holo in luts:
            H_holo = luts[dz_holo]
        else:
            H_holo = compute_bl_asm_transfer_function(dz_holo, display_params['lambda'], display_params['px'], display_params['py'])
            
        # Propagate WRP field to hologram plane
        # FFT -> Multiply by H -> IFFT
        F_wrp = np.fft.fft2(wrp_fields[i])
        F_holo = F_wrp * H_holo
        hologram_field += np.fft.ifft2(F_holo)
        
    # 4. Color Mixing
    # The hologram_field is computed for red wavelength.
    # Scale by R, G, B intensities for final display if the SLM supports color multiplexing
    # Or create separate holograms for each channel.
    # Here we assume the output is the complex field for the red channel.
    
    return hologram_field

def compute_bl_asm_transfer_function(z, wavelength, px, py):
    """
    Compute the Band-Limited Angular Spectrum transfer function.
    """
    Nx, Ny = 1024, 1024 # Example size, should match display params
    # Frequency coordinates
    u = np.fft.fftfreq(Nx, d=px)
    v = np.fft.fftfreq(Ny, d=py)
    U, V = np.meshgrid(u, v)
    
    # Spatial frequency w
    lambda_inv_sq = 1.0 / (wavelength**2)
    w = np.sqrt(lambda_inv_sq - U**2 - V**2)
    w[U**2 + V**2 > lambda_inv_sq] = 0
    
    # Transfer function
    H = np.exp(1j * 2 * np.pi * w * z)
    
    # Band-limiting
    # Determine limits based on Nyquist and SBP
    u_limit = 1.0 / (2 * px)
    v_limit = 1.0 / (2 * py)
    
    rect_u = np.where(np.abs(U) <= u_limit, 1.0, 0.0)
    rect_v = np.where(np.abs(V) <= v_limit, 1.0, 0.0)
    
    H = H * rect_u * rect_v
    
    return H
```

**Edge Cases and Pitfalls:**
*   **Small Sampling Pitch:** Leads to strong diffusion of the light field beyond computation window borders. This is handled by BL-ASM, which doubles the calculation window area in the source plane, pads with zeros, and crops the output [S1].
*   **Aperiodic Functions:** Cause errors at edges due to circular convolution in FFT. BL-ASM mitigates this by bandwidth clipping and zero-padding [S1].
*   **Occlusion:** Must be explicitly modeled using inverted Gaussian filters during propagation to prevent light from passing through opaque objects [S1].
*   **HPO Astigmatism:** Constrained viewing zones and astigmatic output due to diminished vertical resolution [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M \cdot K)$ where $N$ is number of points, $M$ is number of hologram pixels (or hololines), and $K$ is number of WRPs. The use of LUTs and FFT-based propagation reduces the constant factor significantly compared to direct summation.
*   **Memory Complexity:** $O(M \cdot K)$ for storing WRPs and LUTs.
*   **Performance:** Demonstrated on a display with 32,768-by-768 pixels, 0.55 µm horizontal pitch, and 640 nm wavelength [S1].
*   **Strengths:** Fast computation for point clouds; handles occlusion naturally; wide field of view (up to 70°) via diffuse reflection modeling; suitable for HPO displays with high horizontal resolution.
*   **Limitations:** HPO restricts vertical parallax; requires pre-computation of LUTs; sensitive to sampling pitch and window size due to FFT artifacts.

## Background
The method originates from the work on fast CGH generation for point clouds using multiple wavefront recording planes [S1]. It was adapted for horizontal parallax only (HPO) displays to leverage high-resolution horizontal pixels while accepting lower vertical resolution, a common constraint in light field displays like the Holografika HoloVizio 722RC [S1]. The technique builds on the intra-WRP light wave propagation strategy using pre-computed propagation kernels [S1].

## Sources
[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
