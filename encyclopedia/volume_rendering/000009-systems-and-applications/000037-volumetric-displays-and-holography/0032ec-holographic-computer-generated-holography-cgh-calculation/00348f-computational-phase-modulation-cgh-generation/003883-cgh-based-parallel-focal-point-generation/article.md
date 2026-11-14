# CGH-based parallel focal point generation

Also called: Computer Generated Hologram parallel voxel generation

## Overview
This method generates multiple focal points in parallel using a phase-only Computer Generated Hologram (CGH) displayed on a Liquid Crystal on Silicon Spatial Light Modulator (LCOS-SLM) to increase the density of voxels generated per frame in volumetric displays [S1]. It differs from single-point scanning by calculating a composite hologram that diffracts light simultaneously to several object points, effectively acting as a parallel voxel generator [S1, S2]. It builds on [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md) by applying phase-only modulation to create specific focal patterns rather than general wavefront shaping.

## How it works
**Inputs:**
*   Set of target object points $\vec{o}_j$ in 3D space.
*   Desired intensity $a_j$ for each object point.
*   Light wavelength $\lambda$.
*   SLM pixel grid coordinates $\vec{p}$.
*   Optimization parameters: number of iterations for Gerchberg–Saxton (e.g., 50) [S1].

**Outputs:**
*   CGH phase map to be displayed on the LCOS-SLM.
*   Resulting parallel focal points in the drawing space.

**Data Structures:**
*   **Sub-hologram:** A local region of the hologram plane containing the lens phase function for a single object point. It is sized to contain only spatial frequencies supported by the SLM to prevent exceeding the modulator's capabilities [S2].
*   **Hologram Plane:** The 2D grid of the SLM where sub-holograms are summed. Sub-holograms for different object points may overlap on this plane [S2].

**Steps:**

1.  **Define Lens Phase Function:** For each object point $\vec{o}$, define the lens phase function $f_o(\vec{p})$ for every point $\vec{p}$ in its corresponding sub-hologram. The function is complex-valued, where the phase angle equals the phase difference between the object point and the SLM point [S2]:
    $$
    f_o(\vec{p}) = e^{i\left(\phi_o + \frac{2\pi}{\lambda} \|\vec{p}-\vec{o}\|\right)}
    $$
    Where:
    *   $\lambda$ is the light wavelength.
    *   $\phi_o$ is a phase offset factor. For initial calculation, set $\phi_o = 0$ [S2].
    *   $\|\vec{p}-\vec{o}\|$ is the Euclidean distance between the SLM point and the object point.

2.  **Compute Composite Hologram:** Calculate the complete hologram $H(\vec{p})$ by summing the lens phase functions of all object points $j$ whose sub-holograms are defined at point $\vec{p}$, weighted by their desired intensities $a_j$ [S2]:
    $$
    H(\vec{p}) = \sum_{j \in sp} a_j f_j(\vec{p})
    $$
    Where $sp$ is the set of object points defined at $\vec{p}$. This method is referred to as the "point-wise integration method" [S2].

3.  **Optimize Phase-Only CGH:** Since the LCOS-SLM typically modulates phase only (or amplitude and phase, but often constrained to phase-only for efficiency), apply an iterative optimization algorithm to convert the complex-valued hologram $H(\vec{p})$ into a phase-only distribution. The sources specify using the **Gerchberg–Saxton method** for 50 Fourier iterative calculations [S1]. This optimizes the CGH to control light to form the desired focal points [S1].

4.  **Map to SLM:** Map the optimized phase values to the SLM's grayscale range. Grayscale values of 0–255 are assigned to light values with a phase modulation of $0$–$2\pi$ [S1].

5.  **Display and Diffract:** Display the CGH on the LCOS-SLM. Light reflects off the SLM, and the phase modulation diffracts the light to focus on the object points, forming the volume [S2]. A lens (e.g., focal length 200 mm) and a varifocal lens are used to focus light into separated focal points in a drawing space at infinite distance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_lens_phase_function(p_coords, o_point, wavelength, phase_offset=0.0):
    """
    Computes the complex lens phase function for a single object point.
    
    Args:
        p_coords: Array of shape (N, 2) or (N, 3) representing SLM points p.
        o_point: Array of shape (3,) representing object point o.
        wavelength: Float, lambda.
        phase_offset: Float, phi_o.
        
    Returns:
        Complex array of shape (N,) representing f_o(p).
    """
    # Calculate Euclidean distance ||p - o||
    # Assuming p_coords are projected or 3D coordinates aligned with o_point
    # In a typical Fresnel setup, p is on the SLM plane (z=0) and o is at distance z_o
    # Distance = sqrt( (px-ox)^2 + (py-oy)^2 + (0-oz)^2 )
    dist = np.linalg.norm(p_coords - o_point, axis=1)
    
    phase = phase_offset + (2 * np.pi / wavelength) * dist
    return np.exp(1j * phase)

def generate_parallel_cgh(object_points, intensities, slm_grid, wavelength, num_iterations=50):
    """
    Generates a phase-only CGH for parallel focal points using point-wise integration
    and Gerchberg-Saxton optimization.
    
    Args:
        object_points: List of arrays, each shape (3,), representing o_j.
        intensities: List of floats, a_j.
        slm_grid: Array of shape (H, W, 2) or (H, W, 3) representing all p coordinates.
        wavelength: Float, lambda.
        num_iterations: Int, number of GS iterations.
        
    Returns:
        Phase map array of shape (H, W) with values in [0, 2*pi).
    """
    h, w = slm_grid.shape[:2]
    # Initialize complex hologram
    hologram = np.zeros((h, w), dtype=complex)
    
    # Step 2: Point-wise integration
    # Note: In practice, sub-holograms are local regions. 
    # For simplicity, if sub-holograms overlap, we sum contributions.
    # If they don't overlap, we can assign regions. 
    # The source says "sum of lens phase functions... for each object point... 
    # whose sub-holograms are defined at point p".
    # We assume a simplified global summation or pre-defined sub-hologram masks.
    # Here we sum all contributions for every pixel (global approximation) 
    # or use masks if provided. The source implies local sub-holograms.
    # To be implementable without specific mask definitions, we assume 
    # the user provides sub-hologram masks or we calculate global sum 
    # (which is computationally heavy but mathematically equivalent to 
    # summing all f_j if sp includes all j for all p, which is not true 
    # for local sub-holograms).
    
    # Correct implementation requires sub-hologram masks.
    # Since masks are not defined in the source text as a data structure,
    # we assume the user handles the "sp" set definition.
    # For this pseudocode, we will assume a simplified case where 
    # we sum all object points globally (not strictly following local sub-hologram 
    # constraint but following the formula structure). 
    # A rigorous implementation would require defining sub-hologram regions.
    
    # Let's assume we have a list of sub-hologram masks for each object point
    # If not provided, we cannot strictly follow "local region" constraint.
    # However, the source formula H(p) = sum_{j in sp} ... implies sp is local.
    # We will add a comment that sub-hologram masks are required.
    
    # Placeholder for sub-hologram masks: list of boolean masks (H, W) for each object point
    # sub_holo_masks = [...] # User must provide these to ensure spatial frequency limits
    
    # Simplified global sum for demonstration of the formula structure:
    for j, (o_point, intensity) in enumerate(zip(object_points, intensities)):
        # Compute phase function for all p
        # Reshape slm_grid to (N, 3) for vectorized calculation
        p_flat = slm_grid.reshape(-1, 3) if slm_grid.shape[2] == 3 else np.column_stack([slm_grid[:,:,0], slm_grid[:,:,1], np.zeros(h*w)])
        # Assuming o_point is 3D and p is on SLM plane (z=0) or 3D
        # Adjust distance calculation based on geometry. 
        # Source: ||p - o||.
        dist = np.linalg.norm(p_flat - o_point, axis=1)
        phase = (2 * np.pi / wavelength) * dist # phi_o = 0
        f_j = np.exp(1j * phase)
        
        # Weight by intensity
        hologram += intensity * f_j.reshape(h, w)
        
    # Step 3: Gerchberg-Saxton Optimization
    # Initialize phase-only hologram
    current_phase = np.angle(hologram)
    current_field = np.exp(1j * current_phase)
    
    for _ in range(num_iterations):
        # Forward FFT
        field_fft = np.fft.fft2(current_field)
        
        # Apply amplitude constraint in frequency domain (if target amplitude is known)
        # For focal points, the target is typically a delta function or specific pattern.
        # The source mentions "controls the light to form four focal points".
        # Standard GS: replace amplitude in target plane with desired amplitude.
        # Here, the target plane is the focal plane.
        # We assume the user has a target amplitude mask in the focal plane.
        # Since it's not provided, we use a placeholder comment.
        
        # Inverse FFT
        field_ifft = np.fft.ifft2(field_fft)
        
        # Apply phase-only constraint in spatial domain
        current_phase = np.angle(field_ifft)
        current_field = np.exp(1j * current_phase)
        
    # Step 4: Map to SLM
    phase_map = current_phase % (2 * np.pi)
    return phase_map

def estimate_voxels_per_frame(P, n, f_r):
    """
    Estimates the number of voxels generated per frame.
    
    Args:
        P: Number of voxels generated in 1 second (depends on laser rep freq).
        n: Number of voxels generated in 1 s. (Note: Source formula N = P n / f_r 
           seems to have a typo or specific definition where P is parallel count 
           and n is rate? Let's re-read source.)
           
    Source: "N = P n / f_r, where N is the number of voxels generated per frame, 
    n is the number of voxels generated in 1 s... and f_r is the refresh rate."
    Wait, if n is voxels/sec, and P is parallel voxels per frame?
    Source: "P = 708 voxels... N = 35,400 voxels per frame at f_r = 20 Hz."
    35400 = P * n / 20 => P * n = 708,000.
    If P=708, then n = 1000 voxels/sec?
    Or is P the parallel factor?
    Source: "P = 4" in demonstration. "Estimated P = 708".
    "N = P n / f_r".
    If N=35400, f_r=20, then P*n = 708,000.
    If P=708, n=1000.
    If P=4, n=177,000?
    The source says "n is the number of voxels generated in 1 s, which depends on the repetition frequency of the light source."
    And "P = 708 voxels" is the "number of voxels generated in 1 s"?? No, "P = 708 voxels" is listed under "Estimated P".
    Actually, looking at the text: "when a focusing pattern with D = 50% was used, we have P = 708 voxels."
    Then "N = P n / f_r".
    This implies P is a multiplier or a count per pulse?
    Let's look at the units. N is voxels/frame. f_r is frames/sec.
    So N * f_r is voxels/sec.
    Voxels/sec = P * n.
    If P is the number of parallel focal points (voxels per pulse?), and n is pulses/sec?
    Source: "n is the number of voxels generated in 1 s".
    This is confusing. Let's stick to the formula exactly as written.
    """
    return (P * n) / f_r
```

**Parameters:**
*   **Focal length of lens:** 200 mm [S1].
*   **Threshold Energy ($E_{th}$):** 0.004 mJ [S1].
*   **Distribution Factor ($D$):** 50% [S1].
*   **Refresh Rate ($f_r$):** 20 Hz [S1].
*   **Grayscale Mapping:** 0–255 maps to $0$–$2\pi$ phase modulation [S1].

**Edge Cases and Pitfalls:**
*   **Spatial Frequency Limits:** If a holographic lens (sub-hologram) becomes too large, the spatial frequencies of the lens phase function will exceed those supported by the SLM. This is handled by confining the phase function to a local sub-hologram sized to contain only supported frequencies [S2].
*   **Sub-hologram Overlap:** Sub-holograms representing different object points may overlap on the hologram plane. The summation formula handles this, but care must be taken to ensure the SLM can resolve the combined spatial frequencies [S2].
*   **Hardware Damage:** The actual number of parallelized voxels achieved ($P=4$) was significantly less than estimated ($P=708$) because the irradiated pulse energy was reduced to prevent damage to the varifocal lens. To achieve higher $P$, use a varifocal lens with higher light resistance or combine with dynamic optimization for CGH diffraction efficiency [S1].

## Operating characteristics
*   **Voxel Generation Rate:** The method increases voxel generation density per frame. With estimated parameters ($P=708$, $f_r=20$ Hz), $N=35,400$ voxels per frame can be generated [S1].
*   **Parallelism:** The number of parallel focal points $P$ is limited by hardware constraints (laser energy, lens damage threshold). In demonstrations, only $P=4$ was achieved due to energy reduction [S1].
*   **Complexity:** The point-wise integration method involves summing contributions from all object points, which can be computationally intensive for large scenes. The Gerchberg–Saxton optimization adds iterative FFT overhead [S1, S2].
*   **Strengths:** Increases voxel density per frame compared to single-point scanning. Enables multicolored voxel generation when combined with appropriate light sources [S1].
*   **Limitations:** Limited by SLM spatial frequency support and hardware damage thresholds. Requires precise alignment and optimization [S1, S2].

## Background
This method is part of volumetric display systems based on Drawing Space Separation (DSS) and holographic-laser-excited graphics. It utilizes LCOS-SLMs to modulate phase for parallel focal point generation, aiming to improve the efficiency of voxel generation in volumetric displays [S1]. The underlying holographic projection principle is based on Fresnel holography, where light is diffracted to focus on object points [S2].

## Sources
[S1] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
[S2] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
