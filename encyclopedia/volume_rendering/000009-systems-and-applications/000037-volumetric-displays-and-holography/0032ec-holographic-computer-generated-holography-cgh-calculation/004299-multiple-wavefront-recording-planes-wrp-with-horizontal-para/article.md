# Multiple Wavefront Recording Planes (WRP) with Horizontal Parallax Only (HPO)

Also called: Fast CGH method for point clouds, HPO WRP method

## Overview
This method generates computer-generated holograms (CGH) for point clouds by decomposing the 3D volume into multiple parallel Wavefront Recording Planes (WRPs) and computing Horizontal Parallax Only (HPO) holograms. It solves the computational bottleneck of full-parallax point-cloud CGH by restricting diffraction to 1D hololines per WRP, while using the Band-Limited Angular Spectrum Method (BL-ASM) to handle the high spatial bandwidth product (SBP) and aperiodic boundary errors inherent in high-resolution displays [S1]. It distinguishes itself from full-parallax WRP methods by reducing the 2D propagation problem to 1D line computations, and from single-plane methods by handling occlusion and depth via multiple parallel planes [S1].

## How it works
**Inputs:**
*   Point cloud data: Set of points $P = \{p_i\}$, where each point $p_i$ has position $\mathbf{r}_i = (x_i, y_i, z_i)$ and color/intensity values $(I_R, I_G, I_B)$.
*   Display parameters: Pixel pitch $p_x, p_y$, resolution $N_x, N_y$, wavelength $\lambda$, and number of WRPs $K$.
*   Hologram geometry: Hologram plane at $z=0$, WRPs at depths $z_k$ for $k=1 \dots K$.

**Outputs:**
*   Complex field distribution $U_h(\xi, \eta)$ on the hologram plane, encoded as phase-only values for the SLM.

**Data Structures:**
*   **Point Cloud:** List of tuples $(x, y, z, r, g, b)$.
*   **WRP Grid:** For each WRP $k$, a 2D grid of hololines. Since it is HPO, each "hololine" is a 1D array of complex values along the horizontal axis $x$, indexed by $x$ and depth $z_k$. Vertical resolution is handled by the display's vertical pixel pitch, but the calculation is 1D per line.
*   **Look-Up Tables (LUTs):** Pre-computed propagation kernels $H_{LUT}(u, v; z)$ for various distances $z$ between WRPs and the hologram, or between adjacent WRPs.

**Parameters:**
*   $\lambda$: Wavelength (e.g., 640 nm) [S1].
*   $p_x, p_y$: Pixel pitch (e.g., 0.55 µm horizontal, 23.4 µm vertical) [S1].
*   $N_x, N_y$: Resolution (e.g., 32,768 × 768) [S1].
*   $z_{min}$: Minimum object-to-hologram distance, calculated as $z_{min} = \frac{N p^2}{\lambda}$ [S1].
*   $\theta_{max}$: Maximum diffraction angle, $\sin \theta_{max} = \frac{\lambda}{2p}$ [S1].

**Steps:**

1.  **Point Volume Attribution:**
    Assign a Gaussian distribution profile to each point $p_i$ to create a smooth, continuous surface appearance rather than discrete speckles [S1]. The amplitude $A_i$ of point $i$ is distributed over the WRP grid based on its proximity.

2.  **LUT Generation (Band-Limited Angular Spectrum):**
    Pre-compute the transfer functions for all required propagation distances $z$ using the Band-Limited Angular Spectrum Method (BL-ASM) to handle aperiodic functions and prevent edge errors [S1].
    *   Define the spatial frequency domain grid $(u, v)$.
    *   Compute the standard transfer function $H(u, v; z) = \exp(i 2\pi w z)$, where $w = \sqrt{\lambda^{-2} - u^2 - v^2}$ for $u^2 + v^2 \le \lambda^{-2}$ and $0$ otherwise [S1].
    *   Apply bandwidth limiting: $H'(u, v; z) = H(u, v; z) \cdot \text{rect}\left(\frac{u}{2u_{limit}}\right) \cdot \text{rect}\left(\frac{v}{2v_{limit}}\right)$ [S1].
    *   Store $H'$ in the LUT.

3.  **Inter-WRP Propagation (Occlusion Handling):**
    Propagate the wavefield from the object plane (or previous WRP) to the current WRP $k$ using BL-ASM.
    *   To model occlusion, apply inverted Gaussian filters during propagation. This locally blocks light from points that are behind other points relative to the WRP [S1].
    *   Use the convolutional theorem: $G(u, v; z) = G(u, v; 0) H'(u, v; z)$ [S1].
    *   **BL-ASM Padding:** To avoid circular convolution errors from aperiodic functions, double the calculation window size in the source plane by padding with zeros. After the FFT-based propagation, crop the output window back to the original size at the destination plane [S1].

4.  **HPO Hololine Computation:**
    For each WRP $k$ at depth $z_k$:
    *   The hologram is composed of hololines (1D holograms).
    *   For each horizontal position $x$ on the WRP, compute the contribution of all visible points in the point cloud to that specific 1D line.
    *   Since it is HPO, diffraction is computed only along the horizontal line; vertical integration is handled by the display's physical pixel structure or subsequent vertical propagation if full parallax were required (but here it is restricted) [S1].
    *   The complex field on the WRP, $U_{WRP}(x, z_k)$, is the superposition of spherical waves from the attributed point volumes, modified by the occlusion filters [S1].

5.  **Propagation to Hologram Plane:**
    Propagate the field from each WRP $k$ to the hologram plane at $z=0$ using BL-ASM.
    *   Retrieve the pre-computed LUT for distance $z_k$.
    *   Perform FFT of the WRP field, multiply by $H'(u, v; z_k)$, and inverse FFT.
    *   Apply BL-ASM padding/cropping if not already handled in the WRP generation step [S1].
    *   Sum the contributions from all $K$ WRPs to get the total field $U_{total}(\xi, \eta)$ on the hologram plane [S1].

6.  **Color Encoding:**
    For color holograms, compute the hologram for a single reference wavelength (e.g., Red, 640 nm) [S1].
    *   Attribute texture colors $(I_R, I_G, I_B)$ to the points.
    *   Scale the complex field amplitudes for the Red, Green, and Blue channels by their respective intensity values $I_R, I_G, I_B$ [S1].
    *   This maintains a wide viewing angle for all channels by using the same phase structure [S1].

7.  **Phase-Only Encoding:**
    Convert the complex field $U_h(\xi, \eta)$ to a phase-only hologram $\phi(\xi, \eta)$ suitable for a Phase-Only SLM.
    *   Standard approach: $\phi(\xi, \eta) = \arg(U_h(\xi, \eta))$ [S1, S2].
    *   Alternatively, use iterative algorithms like Gerchberg-Saxton if amplitude fidelity is critical [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_bl_asm_transfer_function(u, v, z, lambda_w, u_limit, v_limit):
    """
    Computes the Band-Limited Angular Spectrum transfer function.
    u, v: 2D grids of spatial frequencies.
    z: Propagation distance.
    lambda_w: Wavelength.
    u_limit, v_limit: Bandwidth clipping limits.
    """
    # Calculate w
    w_sq = (1.0 / lambda_w**2) - u**2 - v**2
    # Handle propagating vs evanescent waves
    w = np.where(w_sq >= 0, np.sqrt(w_sq), 0.0)
    
    # Standard transfer function
    H = np.exp(1j * 2 * np.pi * w * z)
    
    # Band-limiting using rect function
    # rect(x) is 1 if |x| <= 0.5, else 0. 
    # Here rect(u / 2*u_limit) is 1 if |u| <= u_limit
    rect_u = np.where(np.abs(u) <= u_limit, 1.0, 0.0)
    rect_v = np.where(np.abs(v) <= v_limit, 1.0, 0.0)
    
    H_prime = H * rect_u * rect_v
    return H_prime

def propagate_field_BLASM(field_in, z, lambda_w, u, v, u_limit, v_limit, 
                          pad_factor=2.0, crop=True):
    """
    Propagates a field using BL-ASM with zero-padding to handle aperiodic boundaries.
    field_in: 2D complex array.
    """
    # Determine new size for padding
    h, w = field_in.shape
    h_pad = int(h * pad_factor)
    w_pad = int(w * pad_factor)
    
    # Pad with zeros
    field_padded = np.zeros((h_pad, w_pad), dtype=complex)
    # Center the original field in the padded array
    start_h, start_w = (h_pad - h) // 2, (w_pad - w) // 2
    field_padded[start_h:start_h+h, start_w:start_w+w] = field_in
    
    # FFT
    F = np.fft.fft2(field_padded)
    
    # Compute frequencies for the padded grid
    # Note: u and v grids must correspond to the padded size
    # Assuming u, v are passed as grids for the padded size or scaled appropriately
    # For simplicity, assume u_pad, v_pad are precomputed for the padded grid
    # In practice, u_pad = np.fft.fftfreq(h_pad, d=p_x) etc.
    
    H_prime = compute_bl_asm_transfer_function(u, v, z, lambda_w, u_limit, v_limit)
    
    # Multiply in frequency domain
    F_out = F * H_prime
    
    # Inverse FFT
    field_out_padded = np.fft.ifft2(F_out)
    
    # Crop back to original size
    if crop:
        field_out = field_out_padded[start_h:start_h+h, start_w:start_w+w]
    else:
        field_out = field_out_padded
        
    return field_out

def generate_hpo_wrp_cgh(points, wrp_depths, hologram_res, pixel_pitch, 
                         wavelength, u_limit, v_limit):
    """
    Generates HPO CGH using Multiple WRPs.
    """
    # 1. Initialize Hologram Plane
    h, w = hologram_res
    hologram_field = np.zeros((h, w), dtype=complex)
    
    # Precompute frequency grids for the hologram size
    # u, v grids for hologram plane
    u_hol = np.fft.fftfreq(w, d=pixel_pitch[0])
    v_hol = np.fft.fftfreq(h, d=pixel_pitch[1])
    U_hol, V_hol = np.meshgrid(u_hol, v_hol)
    
    # 2. Process each WRP
    for z_wrp in wrp_depths:
        # Initialize WRP field (1D hololines, but stored as 2D for propagation)
        # For HPO, we might compute 1D lines, but for BL-ASM propagation to hologram,
        # we treat it as a 2D plane where vertical variation is minimal or handled by display pitch.
        wrp_field = np.zeros((h, w), dtype=complex)
        
        # 3. Compute contributions from points to WRP
        # This step involves attributing Gaussian volumes and occlusion filtering.
        # Simplified: Superposition of point sources to WRP plane.
        for point in points:
            x, y, z_obj, r, g, b = point
            dist = z_wrp - z_obj
            
            # Check if point is in front of WRP (valid for forward propagation)
            if dist <= 0:
                continue
                
            # Compute contribution to WRP
            # In a full implementation, this would involve the Gaussian volume 
            # and inverted Gaussian occlusion filters.
            # Here, we approximate with a spherical wave contribution to the WRP plane.
            # For HPO, we only care about the horizontal diffraction.
            
            # Calculate phase and amplitude at WRP for this point
            # This is a placeholder for the complex Gaussian/occlusion logic [S1]
            # amplitude = gaussian_profile(point, wrp_field_grid)
            # phase = spherical_wave(point, wrp_field_grid)
            
            # For the sake of the algorithm structure, we assume wrp_field is populated
            # by a function compute_point_contribution_to_wrp
            wrp_field += compute_point_contribution_to_wrp(point, z_wrp, wavelength)
            
        # 4. Propagate WRP field to Hologram Plane using BL-ASM
        # Pad the WRP field
        # Note: u, v grids for WRP might need to be recomputed or scaled if resolution differs
        # Assuming same resolution for simplicity
        hologram_field += propagate_field_BLASM(wrp_field, z_wrp, wavelength, 
                                                U_hol, V_hol, u_limit, v_limit)
        
    # 5. Color Encoding
    # Scale hologram_field by color intensities (assuming single wavelength reference)
    # This is typically done by storing separate channels or summing weighted fields
    # For phase-only, we take the argument of the total complex field
    # If color is required, separate holograms for R, G, B might be generated 
    # or multiplexed. The source suggests computing for one wavelength with 
    # scaled intensities [S1].
    
    # 6. Phase-Only Encoding
    phase_hologram = np.angle(hologram_field)
    
    return phase_hologram

def compute_point_contribution_to_wrp(point, z_wrp, wavelength):
    """
    Placeholder for the complex Gaussian volume and occlusion logic.
    Returns a 2D array representing the field contribution on the WRP.
    """
    # This function would implement:
    # 1. Gaussian distribution attribution [S1]
    # 2. Inverted Gaussian filtering for occlusion [S1]
    # 3. Spherical wave propagation from point to WRP
    pass
```

## Operating characteristics
*   **Time Complexity:** $O(K \cdot N \log N)$ per WRP, where $K$ is the number of WRPs and $N$ is the number of pixels. The BL-ASM FFT operations dominate. HPO reduces the effective computation per WRP compared to full parallax by restricting diffraction to 1D lines, but the 2D FFTs are still used for propagation between planes [S1].
*   **Memory Complexity:** $O(N)$ for the field arrays and LUTs. LUTs must store transfer functions for all required depths.
*   **Performance:** Tested on an Holografika HoloVizio 722RC display (32,768 × 768 pixels, 0.55 µm pitch) [S1]. The method enables high-resolution holograms with wide viewing angles (71.1° FOV) [S1].
*   **Strengths:** Handles occlusion naturally via multiple WRPs; high quality due to Gaussian volume attribution; wide FOV via random phase modification in LUTs; efficient for point clouds [S1].
*   **Limitations:** HPO restricts vertical parallax, leading to astigmatic output and constrained viewing zones vertically [S1]. Requires small pixel pitch for wide FOV, which increases computational load and sensitivity to sampling errors [S1].
*   **When to use:** When high-resolution, wide-FOV holograms are needed for point-cloud data on displays with limited vertical resolution or when vertical parallax is not critical.
*   **When not to use:** When full parallax is required, or when the display has large pixel pitch that cannot support the required SBP for the desired FOV.

## Background
The Multiple WRP method was originally proposed for fast computation of full-parallax holograms [S1]. It was adapted for HPO displays to leverage the specific characteristics of light field displays like the HoloVizio 722RC, which support wide horizontal viewing angles but have coarser vertical resolution [S1]. The use of BL-ASM addresses the limitations of standard Angular Spectrum propagation when dealing with the high spatial bandwidth products of modern high-resolution SLMs [S1].

## Sources
[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
