# CGH-based parallel focal point generation

Also called: CGH-Based Multi-Voxel Generation, Computer Generated Hologram parallel voxel generation, Computer Generated Holography multi-point rendering

## Overview
This method uses phase-only Computer-Generated Holograms (CGH) displayed on a Liquid Crystal on Silicon Spatial Light Modulator (LCOS-SLM) to split a single laser beam into multiple focal spots simultaneously [S1, S2]. It solves the bottleneck of low voxel generation density in holographic volumetric displays by increasing the number of voxels generated per frame [S2]. Unlike single-point rendering, it allows for parallel voxel generation, though it introduces trade-offs in scanning trajectory optimization and hardware constraints [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Target voxel distribution (number of focal points $P$), laser parameters (repetition frequency, pulse energy), LCOS-SLM specifications (maximum spatial frequency, grayscale range).
*   **Output:** A phase-only CGH pattern displayed on the SLM, resulting in $P$ simultaneous focal points in the drawing space.

**Data Structures:**
*   **CGH Pattern:** A 2D array of phase values.
    *   **Layout:** Grayscale values $0$ to $255$ (8-bit).
    *   **Mapping:** Grayscale value $g \in [0, 255]$ maps linearly to phase modulation $\phi \in [0, 2\pi]$.
    *   **Spatial Frequency Constraint:** The encoded focal points must be arranged within a spatial frequency range up to $1/8$ of the LCOS-SLM's maximum spatial frequency [S1].

**Algorithm Steps:**
1.  **CGH Generation:** Generate a phase-only hologram to create $P$ focal points.
    *   Use the Gerchberg–Saxton algorithm with 50 Fourier iterative calculations to optimize the phase distribution [S2].
    *   Ensure the spatial frequency of the generated pattern does not exceed $1/8$ of the SLM's maximum spatial frequency [S1].
2.  **SLM Display:** Load the calculated CGH onto the LCOS-SLM. The SLM modulates the phase of the incident laser beam according to the grayscale values [S1, S2].
3.  **Optical Focusing:** Pass the modulated beam through a lens (focal length $f_{lens}$) and a varifocal lens. The lens focuses the light into separated focal points in the drawing space at an effective infinite distance [S2].
4.  **Scanning:** Apply the scanning trajectory to the beam.
    *   **Note:** If using a trajectory optimized for single-point rendering, blur may occur when rendering multiple spatially separated focal points [S1]. Optimizing the scanning trajectory for multi-point rendering is required to resolve this [S1].

**Formulas:**
The estimated number of voxels generated per frame $N$ is calculated as:
$$ N = \frac{P \cdot n}{f_r} $$
Where:
*   $P$: Number of focal points (voxels) generated in parallel per pulse [S2].
*   $n$: Number of voxels generated in 1 second, dependent on the laser repetition frequency [S2].
*   $f_r$: Refresh rate of the display system [S2].

**Parameters (Typical Values from Sources):**
*   **Lens Focal Length ($f_{lens}$):** 200 mm [S2].
*   **Threshold Energy ($E_{th}$):** 0.004 mJ [S2].
*   **Duty Cycle / Distribution Factor ($D$):** 50% [S2].
*   **Estimated Parallel Voxels ($P$):** 708 (theoretical estimate based on $D=50\%$ and $E_{th}$) [S2].
*   **Actual Achieved Parallel Voxels ($P$):** 4 (due to hardware constraints) [S2].
*   **Refresh Rate ($f_r$):** 20 Hz [S2].
*   **Estimated Voxels per Frame ($N$):** 35,400 [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_cgh_for_parallel_focal_points(target_points, slm_width, slm_height, max_spatial_freq):
    """
    Generates a phase-only CGH to create multiple focal points using Gerchberg-Saxton.
    """
    # Initialize phase array with random phase or uniform phase
    phase = np.random.uniform(0, 2 * np.pi, (slm_height, slm_width))
    
    # Define target intensity distribution in the focal plane
    # This requires defining the positions of the P focal points
    # constrained within 1/8 of the max spatial frequency
    target_intensity = create_target_distribution(target_points, max_spatial_freq)
    
    # Gerchberg-Saxton Iteration
    for iteration in range(50):
        # Forward Fourier Transform to focal plane
        field = np.fft.fft2(np.exp(1j * phase))
        
        # Apply amplitude constraint in focal plane
        # Keep phase, replace amplitude with sqrt(target_intensity)
        amplitude = np.abs(field)
        phase_focal = np.angle(field)
        new_amplitude = np.sqrt(target_intensity)
        
        # Reconstruct field in focal plane
        field_focal = new_amplitude * np.exp(1j * phase_focal)
        
        # Inverse Fourier Transform back to SLM plane
        field_slm = np.fft.ifft2(field_focal)
        
        # Apply amplitude constraint in SLM plane (constant amplitude)
        # Keep phase, replace amplitude with 1 (or normalized value)
        phase = np.angle(field_slm)
        
    # Map phase to 8-bit grayscale
    # Phase 0 -> 0, Phase 2pi -> 255
    cg_h_grayscale = np.round(phase / (2 * np.pi) * 255) % 256
    return cg_h_grayscale

def calculate_voxels_per_frame(parallel_points, laser_rep_freq, refresh_rate):
    """
    Calculates N = P * n / f_r
    """
    # n is voxels per second, dependent on laser rep freq
    # Assuming 1 voxel per pulse per focal point for simplicity in this formula context
    # n = laser_rep_freq * parallel_points
    # However, source defines n as "number of voxels generated in 1 s"
    # and N = P * n / f_r. 
    # If P is parallel points per pulse, and n is total pulses per second?
    # Source says: "n is the number of voxels that can be generated in 1 s... N = Pn/fr"
    # This implies P is a multiplier factor or n is base rate.
    # Based on source: N = 35400, P=708 (est), fr=20.
    # 35400 = 708 * n / 20 => n = 35400 * 20 / 708 = 1000.
    # So n=1000 voxels/sec? This seems low for a femtosecond laser.
    # Re-reading: "n is the number of voxels that can be generated in 1 s... depends on repetition frequency".
    # If P is the *estimated* parallel count (708), and actual was 4.
    # The formula N = P * n / f_r likely assumes P is the number of voxels generated *per scan step* or similar.
    # Given the ambiguity, we implement the formula as stated.
    
    # n: voxels generated in 1 second (depends on laser rep freq)
    # P: number of voxels generated in parallel (per frame? or per pulse?)
    # Source: "N is the number of voxels generated per frame"
    # Source: "P = 708 voxels" (estimated parallel)
    # Source: "N = 35,400 voxels per frame at fr = 20 Hz"
    # If N is per frame, and fr is 20 Hz, then N is total voxels in one frame.
    # The formula N = P * n / f_r is dimensionally suspicious if P is count and n is count/sec.
    # P * n / f_r -> [count] * [count/sec] / [1/sec] = [count]^2.
    # There is likely a typo in the source or a specific definition of n.
    # However, we must follow the source formula exactly.
    
    return (parallel_points * laser_rep_freq_based_n) / refresh_rate
```

**Edge Cases and Pitfalls:**
*   **Blur:** Using a scanning trajectory optimized for single-point rendering causes increased blur when rendering multiple spatially separated focal points. The trajectory must be optimized for multi-point rendering [S1].
*   **Hardware Damage:** Actual parallelized voxels achieved ($P=4$) were significantly less than estimated ($P=708$) because irradiated pulse energy was reduced to prevent damage to the varifocal lens [S2].
*   **Solution for Damage:** Use a varifocal lens with higher light resistance or combine with dynamic optimization for CGH diffraction efficiency to improve $P$ [S2].

## Operating characteristics
*   **Time Complexity:** CGH generation requires 50 Fourier iterative calculations (Gerchberg–Saxton), which is computationally intensive but done offline or per-frame depending on system capability [S2].
*   **Memory Complexity:** Requires storage for the CGH pattern (2D array of 8-bit values) [S1, S2].
*   **Performance:**
    *   **Estimated:** 35,400 voxels per frame at 20 Hz refresh rate with estimated $P=708$ [S2].
    *   **Actual (Experimental):** 4 parallel focal points achieved due to energy constraints [S2].
*   **Strengths:** Increases voxel generation density per frame; enables simultaneous multi-voxel rendering [S1, S2].
*   **Limitations:** Requires careful scanning trajectory optimization to avoid blur; limited by varifocal lens damage thresholds; actual parallel count may be much lower than theoretical estimates [S1, S2].
*   **When to Use:** When high voxel density is required and hardware can support the necessary pulse energy and lens resistance [S2].
*   **When Not to Use:** When single-point precision is critical and multi-point blur cannot be tolerated without trajectory optimization [S1].

## Background
This method was developed for holographic laser-excited volumetric display systems [S1, S2]. It builds on Computational Phase Modulation (CGH Generation) [Parent]. The technique was demonstrated in experimental systems using femtosecond lasers and LCOS-SLMs to generate volumetric graphics in gas-filled cells [S2]. The Gerchberg–Saxton algorithm is a standard method for phase retrieval in holography [S2].

## Sources
[S1] Dual-Path Holographic Laser-Excited Volumetric Display, https://doi.org/10.1145/3816042, fetched 2026-10-27
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
