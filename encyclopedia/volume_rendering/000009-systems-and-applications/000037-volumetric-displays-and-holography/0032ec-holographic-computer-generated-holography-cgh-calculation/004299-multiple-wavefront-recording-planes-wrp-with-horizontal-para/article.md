# Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)

Also called: Fast CGH method for point clouds, HPO WRP method

## Overview
This method generates computer-generated holograms (CGH) for point clouds by distributing the calculation across multiple parallel wavefront recording planes (WRPs) to handle occlusion and reduce computational load, specifically adapted for Horizontal Parallax Only (HPO) displays. It differs from full-parallax WRP methods by computing point contributions to 1D hololines rather than the full 2D plane, and from naive point-source summation by using pre-computed look-up tables (LUTs) and band-limited angular spectrum propagation to manage the high spatial bandwidth product (SBP) required for wide viewing angles [S1].

## How it works
**Inputs:**
*   Point cloud data: Set of $M$ points, each with 3D coordinates $(x_i, y_i, z_i)$ and color/intensity attributes.
*   Display parameters: Pixel pitch $p$ (horizontal $p_x$, vertical $p_y$), resolution $N$ (pixels), wavelength $\lambda$, and number of WRPs.
*   Hologram geometry: HPO configuration where diffraction is computed only along horizontal lines (hololines).

**Outputs:**
*   Complex-valued hologram data (or phase-only quantized data) for the HPO display.

**Data Structures:**
*   **Point Cloud:** Array of structures containing position $(x, y, z)$ and color $(R, G, B)$.
*   **Look-Up Table (LUT):** Pre-computed complex propagation kernels for a specific distance $z$. The LUT stores the complex amplitude $H(u, v; z)$ in the frequency domain, modified by a random phase mask for diffuse reflection modeling.
*   **Wavefront Recording Planes (WRPs):** A set of parallel planes positioned between the object (point cloud) and the hologram plane. Each WRP holds a complex field representing the wavefront at that depth.
*   **Hololines:** 1D arrays representing the hologram data for a single vertical row of the display.

**Parameters:**
*   $\lambda$: Wavelength of reconstruction light (e.g., 640 nm) [S1].
*   $p_x, p_y$: Horizontal and vertical pixel pitches (e.g., 0.55 µm, 23.4 µm) [S1].
*   $N$: Resolution of the hologram plane (e.g., 32,768 × 768) [S1].
*   $z_{min}$: Minimum object-to-hologram distance to satisfy Shannon criterion, calculated as $z_{min} = \frac{N p^2}{\lambda}$ [S1].
*   $\theta$: Maximum diffraction angle, determined by $\sin \theta = \frac{\lambda}{2p}$ [S1].

**Steps:**

1.  **Point Volume Attribution:**
    Assign a Gaussian distribution profile to each point in the point cloud to model a continuous volume rather than a singular point. This generates smoother, continuous surface reconstructions [S1]. The amplitude $A_i$ of point $i$ at a distance $r$ from its center is modeled as:
    $$ A_i(r) = A_{0,i} \exp\left(-\frac{r^2}{2\sigma^2}\right) $$
    where $\sigma$ controls the spread. [G: $\sigma$ is typically chosen based on the desired surface smoothness and pixel pitch; not explicitly defined in sources, but implied by "Gaussian distribution profile"].

2.  **LUT Generation (Source Plane to First WRP):**
    Pre-compute the propagation kernel from the point source plane to the first WRP distance $z_{LUT}$.
    *   Calculate the angular spectrum transfer function $H(u, v; z_{LUT})$ using the Band-Limited Angular Spectrum Method (BL-ASM).
    *   Apply a random phase mask to the LUT content to account for diffuse reflections, scattering light over a wide angular range [S1].
    *   Store the complex kernel $H_{LUT}(u, v)$ in the LUT.

3.  **Inter-WRP Propagation:**
    Propagate the wavefield from one WRP to the next parallel WRP using BL-ASM.
    *   For each WRP pair separated by distance $\Delta z$, compute the transfer function $H(u, v; \Delta z)$.
    *   Apply BL-ASM constraints: Double the calculation window area in the source plane by padding with zeros. Perform FFT, multiply by the transfer function, and IFFT. Crop the output to the original window size to avoid edge errors from circular convolution [S1].

4.  **HPO Hololine Computation:**
    Adapt the standard WRP method for HPO by computing contributions only to 1D hololines.
    *   For each point $i$, determine its contribution to the hololines on the hologram plane.
    *   Instead of propagating to the full 2D hologram plane, propagate the wavefield from the last WRP to the hologram plane along the horizontal direction only.
    *   Use inverted Gaussian filters during propagation to model occlusions, locally blocking light from points behind others [S1].

5.  **Superposition:**
    Sum the complex contributions from all points $i$ onto the respective hololines of the hologram plane.
    $$ U_h(x, y) = \sum_{i=1}^{M} A_i \exp(i \phi_i(x, y)) $$
    where $\phi_i(x, y)$ is the phase derived from the propagation distance and the Gaussian volume model [S1, S3].

6.  **Color Channel Handling:**
    For color holograms, compute the hologram for a single reference wavelength (e.g., red, 640 nm) but assign RGB intensity values to the points. The final hologram consists of three channels computed with the same wavelength but different intensities per point to maintain color information and wide viewing angle [S1].

7.  **Quantization (Optional):**
    If using a phase-only SLM, quantize the complex field $U_h(x, y)$ to phase values $0$ to $2\pi$. [G: Standard phase-only encoding techniques such as detour phase or error diffusion may be applied, but specific algorithm details are not provided in the source].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_hpo_wrp_cgh(points, display_params, wrp_distances, lut_distance):
    """
    Generate HPO CGH using Multiple WRP method.
    
    Args:
        points: List of dicts with keys 'x', 'y', 'z', 'r', 'g', 'b'.
        display_params: Dict with 'lambda', 'px', 'py', 'Nx', 'Ny'.
        wrp_distances: List of z-distances for each WRP plane.
        lut_distance: Distance for LUT generation.
        
    Returns:
        hologram: 2D array (Ny, Nx) of complex values.
    """
    lambda_val = display_params['lambda']
    px = display_params['px']
    Nx = display_params['Nx']
    Ny = display_params['Ny']
    
    # Initialize hologram plane (HPO: 1D hololines summed vertically)
    # Hologram is Ny x Nx
    hologram = np.zeros((Ny, Nx), dtype=complex)
    
    # Pre-compute LUT for point-to-first-WRP propagation
    # LUT stores frequency domain transfer function with random phase
    lut = generate_lut(lut_distance, lambda_val, px, Nx, Ny)
    
    # Initialize WRPs
    # Each WRP is a 2D complex field. For HPO, we can optimize to 1D per row,
    # but for generality, we use 2D arrays.
    wrp_fields = [np.zeros((Ny, Nx), dtype=complex) for _ in range(len(wrp_distances))]
    
    # Step 1: Project points to first WRP (or source plane)
    # Since points are discrete, we treat them as sources.
    # For each point, calculate contribution to the first WRP.
    # To optimize, we use the LUT if the point is at the source plane distance.
    
    for point in points:
        # Gaussian volume attribution
        # Calculate amplitude and phase at the first WRP plane (z = wrp_distances[0])
        # Distance from point to WRP plane
        dz = wrp_distances[0] - point['z']
        if dz <= 0:
            continue # Point behind WRP
            
        # For HPO, we compute hololines. 
        # The contribution of a point to a hololine at y_h is determined by the 
        # horizontal distance and the vertical position.
        
        # Simplified: Propagate point to first WRP using LUT
        # The LUT is pre-computed for a point source at z=0 propagating to z=lut_distance.
        # We assume points are projected to a common source plane or handled individually.
        
        # Create a sparse representation of the point at the source plane
        # In practice, this involves placing the Gaussian amplitude at the point's (x,y) on the source plane
        source_field = np.zeros((Ny, Nx), dtype=complex)
        
        # Map point coordinates to grid indices
        # [G: Coordinate mapping logic depends on specific pixel grid alignment]
        ix = int(point['x'] / px)
        iy = int(point['y'] / px) # Assuming square pixels for mapping, though py != px
        
        # Apply Gaussian spread
        # [G: Exact sigma value not given, assume small spread relative to pixel]
        sigma = px 
        for dy in range(-int(sigma/px), int(sigma/px)+1):
            for dx in range(-int(sigma/px), int(sigma/px)+1):
                if 0 <= iy+dy < Ny and 0 <= ix+dx < Nx:
                    dist = np.sqrt(dx**2 + dy**2) * px
                    amp = point['r'] * np.exp(-dist**2 / (2*sigma**2)) # Use R as base intensity
                    source_field[iy+dy, ix+dx] += amp
                    
        # Propagate source_field to first WRP using LUT (BL-ASM)
        wrp_fields[0] += propagate_bl_asm(source_field, lut, lambda_val, px, Nx, Ny)
        
    # Step 2: Propagate between WRPs
    for i in range(len(wrp_distances) - 1):
        dz = wrp_distances[i+1] - wrp_distances[i]
        # Generate transfer function for this distance
        h = get_transfer_function(dz, lambda_val, px, Nx, Ny)
        # Apply BL-ASM
        wrp_fields[i+1] = propagate_bl_asm(wrp_fields[i], h, lambda_val, px, Nx, Ny)
        
    # Step 3: Propagate from last WRP to Hologram Plane (HPO)
    dz_final = display_params['z_hologram'] - wrp_distances[-1]
    h_final = get_transfer_function(dz_final, lambda_val, px, Nx, Ny)
    
    # For HPO, we only compute horizontal diffraction.
    # This implies we might process each row (hololine) independently or use 1D FFTs.
    # However, the source states "computing the contribution of the points to hololines... only diffracting along the line".
    # This suggests a 1D propagation in frequency domain for each row.
    
    for y in range(Ny):
        # Extract row from last WRP
        field_row = wrp_fields[-1][y, :]
        
        # 1D BL-ASM propagation to hologram plane
        # [G: 1D version of BL-ASM]
        hologram_row = propagate_bl_asm_1d(field_row, h_final, lambda_val, px, Nx)
        
        # Add to hologram
        hologram[y, :] += hologram_row
        
    return hologram

def generate_lut(distance, lambda_val, px, Nx, Ny):
    """Generate LUT with random phase for diffuse reflection."""
    # Compute standard AS transfer function
    h = get_transfer_function(distance, lambda_val, px, Nx, Ny)
    # Add random phase
    random_phase = np.exp(1j * 2 * np.pi * np.random.rand(Ny, Nx))
    return h * random_phase

def propagate_bl_asm(field, h, lambda_val, px, Nx, Ny):
    """Band-Limited Angular Spectrum Method propagation."""
    # Double the window size
    Ny_pad = 2 * Ny
    Nx_pad = 2 * Nx
    
    # Pad with zeros
    field_padded = np.zeros((Ny_pad, Nx_pad), dtype=complex)
    field_padded[:Ny, :Nx] = field
    
    # FFT
    F = np.fft.fft2(field_padded)
    
    # Resize/Interpolate transfer function to match padded size if necessary,
    # or generate it for the padded size.
    # [G: H must be generated for the padded grid spacing]
    h_padded = interpolate_or_generate_h(h, Nx_pad, Ny_pad)
    
    # Multiply
    G = F * h_padded
    
    # IFFT
    field_out_padded = np.fft.ifft2(G)
    
    # Crop to original size
    field_out = field_out_padded[:Ny, :Nx]
    
    return field_out

def get_transfer_function(distance, lambda_val, px, Nx, Ny):
    """Calculate Angular Spectrum transfer function."""
    # Frequency coordinates
    u = np.fft.fftfreq(Nx, d=px)
    v = np.fft.fftfreq(Ny, d=px) # Assuming px for both for simplicity in freq calc
    U, V = np.meshgrid(u, v)
    
    # w calculation
    w_sq = (1/lambda_val**2) - U**2 - V**2
    w = np.zeros_like(w_sq)
    w[w_sq >= 0] = np.sqrt(w_sq[w_sq >= 0])
    
    H = np.exp(1j * 2 * np.pi * w * distance)
    
    # Band-limiting
    u_limit = 1 / (2 * px)
    v_limit = 1 / (2 * px)
    rect_u = np.abs(U) <= u_limit
    rect_v = np.abs(V) <= v_limit
    H = H * rect_u * rect_v
    
    return H
```

## Subtopics

### [Phase-Only Wave Field Reconstruction/Propagation (POS-WRP)](12886-phase-only-wave-field-reconstruction-propagation-pos-wrp.md)
POS-WRP is a rendering pipeline that encodes mesh-based 3D scenes into complex-valued wave fields with fine-grained textures, photo-realistic shadings, and perspective depth cues. It solves the problem of maintaining continuous depth cues and realistic illumination when projecting from perspective scene space to the orthogonal space of a Spatial Light Modulator (SLM). Unlike standard wave propagation which assumes paraxial orthogonal propagation, POS-WRP explicitly transforms mesh coordinates and textures to correct for projection discrepancies, delivering high-fidelity data with reduced speckle artifacts.

## Operating characteristics
**Time Complexity:**
The complexity is dominated by the number of points $M$, the number of WRPs $K$, and the hologram resolution $N$. Naive point-source summation is $O(M \cdot N)$. The WRP method reduces this by pre-computing propagation kernels (LUTs) and propagating fields between planes. The cost is roughly $O(K \cdot N \log N)$ for the WRP propagations plus $O(M \cdot N)$ for the initial point-to-WRP contributions, though HPO adaptation reduces the final hologram computation to $O(M \cdot N_x)$ per row, effectively $O(M \cdot N)$ total but with smaller constant factors due to 1D processing [S1].

**Memory Complexity:**
Requires storage for the point cloud, $K$ WRP fields, and the LUT. Each WRP field is $O(N)$ complex numbers. The LUT is $O(N)$. Total memory is $O(K \cdot N + N)$. For high-resolution holograms (e.g., 32,768 × 768), this can be significant [S1].

**Measured Performance:**
Demonstrated on a Holografika HoloVizio 722RC light field display with 32,768 × 768 pixel resolution, 0.55 µm horizontal pitch, and 640 nm wavelength [S1]. The method supports a viewing angle of 70° [S1].

**Strengths:**
*   Handles occlusion naturally through the WRP structure and inverted Gaussian filters [S1].
*   High-quality reconstructions with smooth surfaces due to Gaussian volume attribution [S1].
*   Wide field of view achieved through diffuse reflection modeling (random phase LUTs) [S1].
*   Efficient for HPO by reducing 2D propagation to 1D hololines [S1].

**Limitations:**
*   HPO results in constrained viewing zones and astigmatic output due to diminished vertical resolution [S1].
*   Small pixel pitch leads to strong light field diffusion beyond computation window borders, requiring BL-ASM and zero-padding, which increases computational overhead [S1].
*   Aperiodic functions cause edge errors in FFT, mitigated by BL-ASM but not eliminated [S1].
*   Limited to point clouds; mesh-based scenes require different handling (see POS-WRP) [S1].

**When to Use:**
Use for point cloud data on HPO displays requiring wide viewing angles and high resolution. Not suitable for full-parallax displays or mesh-based scenes without modification.

**When Not to Use:**
Avoid for full-parallax holography, mesh-based scenes with complex textures (use POS-WRP), or when computational resources are too limited for BL-ASM overhead.

## Background
The Multiple WRP method was originally proposed for fast computation of holograms supporting full parallax [S1]. It was adapted for HPO displays to leverage the specific constraints of light field displays, such as the Holografika HoloVizio 722RC [S1]. The method builds on the Angular Spectrum method for wave propagation and incorporates Band-Limited Angular Spectrum (BL-ASM) to handle the high spatial bandwidth product of modern holographic displays [S1].

## Sources
[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
