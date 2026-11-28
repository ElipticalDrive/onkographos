# Computational Phase Modulation (CGH Generation)
Also called: Fresnel Holography, Fresnel holograms, Lens Phase Function Method, ORA method, Optimization with Random Phase Algorithm, Point-based wavefront propagation

## Overview
Computational Phase Modulation generates Computer-Generated Holograms (CGH) by calculating the complex wavefield contributions of scene elements to a hologram plane, typically displayed on a Spatial Light Modulator (SLM) [S2]. It solves the problem of reconstructing 3D light fields from digital scene data by explicitly modeling the diffraction of light from source elements to the modulator plane [S2]. It is distinguished from [Point-Cloud Based CGH Calculation](13819-point-cloud-based-cgh-calculation.md) by its specific reliance on the "lens phase function" abstraction for sub-holograms and the "point-wise integration" summation model [S2], and from [Layer-Based Holographic Algorithm](13372-layer-based-holographic-algorithm.md) by its direct point-to-plane propagation model rather than layer-by-layer accumulation [S2].

## How it works
**Inputs:**
*   Scene definition: A set of $M$ object points $\vec{o}_j = (x_j, y_j, z_j)$ with desired intensities $a_j$ [S2].
*   Hologram plane parameters: Resolution $(W, H)$, pixel pitch $p$, wavelength $\lambda$ [S2, S3].
*   SLM constraints: Maximum supported spatial frequency, which determines the maximum radius of a sub-hologram [S2].

**Outputs:**
*   A complex-valued hologram pattern $H(\vec{p})$ or phase-only pattern $\phi_h(x,y)$ ready for SLM display [S2, S1].

**Data Structures:**
*   **Hologram Plane:** A 2D grid of complex values $H(x,y)$ or phase values $\phi_h(x,y)$, size $W \times H$ [S2].
*   **Sub-hologram Map:** A mapping from hologram pixel coordinates $\vec{p}$ to the set of object points $\text{sp}(\vec{p})$ whose sub-holograms cover $\vec{p}$ [S2].

**Method:**
The method computes the hologram by summing the contributions of all object points [S2]. Each object point $j$ contributes a "lens phase function" $f_j(\vec{p})$ to every hologram pixel $\vec{p}$ that falls within its designated sub-hologram region [S2].

1.  **Initialize** the hologram plane $H(\vec{p}) = 0$ for all $\vec{p}$ [S2].
2.  **For each** object point $j$ in the scene:
    a.  **Determine Sub-hologram Region:** Calculate the bounding box on the hologram plane that contains the spatial frequencies supported by the SLM for the distance $z_j$ and wavelength $\lambda$ [S2].
    b.  **For each** pixel $\vec{p}$ in the region:
        i.  Calculate the distance $r_{j,\vec{p}} = \|\vec{p} - \vec{o}_j\|$ [S2].
        ii. Calculate the lens phase function contribution:
        $$ f_j(\vec{p}) = \exp\left(i \left(\phi_{0,j} + \frac{2\pi}{\lambda} r_{j,\vec{p}}\right)\right) $$
        where $\phi_{0,j}$ is an initial phase offset (typically 0) [S2].
        iii. Add the weighted contribution to the hologram plane:
        $$ H(\vec{p}) \leftarrow H(\vec{p}) + a_j f_j(\vec{p}) $$
3.  **Post-processing (Optional):** If the SLM is phase-only, convert the complex $H(\vec{p})$ to a phase-only pattern $\phi_h(x,y)$ using an approximation method [S2, S1].
    *   *Note:* The source [S1] describes an iterative optimization (ORA) to refine phase $\phi_h$ for uniform intensity spot arrays, which is a specific variant of this post-processing step.

**Formulas:**
*   **Lens Phase Function:**
    $$ f_o(\vec{p}) = e^{i(\phi_o + \frac{2\pi}{\lambda} \|\vec{p}-\vec{o}\|)} $$
    where $\vec{o}$ is the object point, $\vec{p}$ is the hologram pixel, $\lambda$ is wavelength, and $\phi_o$ is phase offset [S2].
*   **Hologram Summation:**
    $$ H(\vec{p}) = \sum_{j \in \text{sp}(\vec{p})} a_j f_j(\vec{p}) $$
    where $\text{sp}(\vec{p})$ is the set of object points whose sub-holograms include $\vec{p}$, and $a_j$ is the intensity weight [S2].
*   **Reconstruction (Verification):**
    $$ U_r(\nu_x, \nu_y) = \iint U_h(x, y) e^{-i 2\pi (x\nu_x + y\nu_y)} dx dy $$
    where $U_h$ is the hologram pattern and $U_r$ is the reconstructed complex amplitude [S1].

**Parameters:**
*   $\lambda$: Wavelength of reconstruction light [S2].
*   $a_j$: Desired intensity of object point $j$ [S2].
*   $\phi_o$: Phase offset, typically 0 [S2].
*   Sub-hologram size: Determined by SLM spatial frequency limits [S2].

**Edge Cases and Pitfalls:**
*   **Sub-hologram Overlap:** Sub-holograms for different object points may overlap on the hologram plane [S2].
*   **Spatial Frequency Limits:** If a sub-hologram is too large, the phase gradient exceeds the SLM's Nyquist limit [S2].
*   **Phase-Only Modulation:** Most SLMs can only modulate phase or amplitude, not both simultaneously, requiring direct or iterative approximation methods for complex-valued fields [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cgh_phase_modulation(object_points, hologram_width, hologram_height, pixel_pitch, wavelength):
    """
    Computes the complex hologram H(p) using the lens phase function method.
    
    Args:
        object_points: List of tuples (x, y, z, intensity) for each point in the scene.
        hologram_width: Width of the hologram plane in pixels.
        hologram_height: Height of the hologram plane in pixels.
        pixel_pitch: Physical size of one pixel (e.g., in meters).
        wavelength: Wavelength of the reconstruction laser (e.g., in meters).
        
    Returns:
        H: 2D numpy array of complex values representing the hologram plane.
    """
    import numpy as np
    
    # Initialize hologram plane with zeros
    H = np.zeros((hologram_height, hologram_width), dtype=np.complex128)
    
    # Create coordinate grids for the hologram plane
    x_coords = np.arange(hologram_width) * pixel_pitch - (hologram_width * pixel_pitch) / 2
    y_coords = np.arange(hologram_height) * pixel_pitch - (hologram_height * pixel_pitch) / 2
    X, Y = np.meshgrid(x_coords, y_coords)
    
    k = 2 * np.pi / wavelength
    
    for point in object_points:
        ox, oy, oz, intensity = point
        
        # Calculate distance from object point to each hologram pixel
        dx = X - ox
        dy = Y - oy
        dz = -oz
        R = np.sqrt(dx**2 + dy**2 + dz**2)
        
        # Calculate lens phase function
        # f_o(p) = exp(i * (phi_0 + k * R))
        # phi_0 is set to 0 as per source [S2]
        phase = k * R
        f_o = np.exp(1j * phase)
        
        # Add weighted contribution to hologram
        H += intensity * f_o
        
    return H

def apply_phase_only_constraint(H):
    """
    Converts complex hologram to phase-only using Amplitude Discard method.
    """
    phase = np.angle(H)
    return np.exp(1j * phase)
```

## Subtopics

### [CGH-based parallel focal point generation](8277-cgh-based-parallel-focal-point-generation.md)
Uses phase-only CGHs on an LCOS-SLM to split a laser beam into multiple focal spots simultaneously, increasing voxel generation density for volumetric displays. It differs from single-point rendering by enabling parallel voxel generation, though it requires optimization of scanning trajectories and hardware constraints.

### [Phase-Only Hologram Approximations](13999-phase-only-hologram-approximations.md)
Methods for encoding complex-valued object waves into phase-only holograms for LCSLMs without iterative optimization. Includes Amplitude Discard (simplest), Double Phase (encodes amplitude and phase into two adjacent values), and Binary Phase (quantizes to two levels). Trades image quality and energy efficiency for computational simplicity.

### [Extended Phase Range SLM Modulation](17675-extended-phase-range-slm-modulation.md)
Mitigates depth replicas in simultaneous color holography by exploiting the wavelength-dependent phase response of LCoS SLMs. Uses SLMs with phase ranges $> 2\pi$ to break wavelength-depth ambiguity, differing from [Computational Phase Modulation](7284-computational-phase-modulation-cgh-generation.md) by addressing multi-wavelength crosstalk via physical device dispersion rather than algorithmic approximation.

### SLM Phase Delay Modeling
A model for the phase delay $\phi_\lambda$ imparted by an LCoS SLM, accounting for wavelength dispersion [S4]. The phase delay is $\phi_\lambda = \frac{2\pi d}{\lambda} n(s, \lambda)$, where $d$ is the LC layer thickness, $n(s, \lambda)$ is the refractive index controlled by digital input $s$ and dependent on $\lambda$ [S4]. Quantization errors increase with larger phase ranges, requiring at least 6 bits of granularity per $2\pi$ range to maintain PSNR/SSIM [S4].

## Operating characteristics
*   **Complexity:** The naive point-wise integration scales with $O(M \cdot N)$, where $M$ is the number of object points and $N$ is the number of hologram pixels [G]. This is extremely expensive for high-resolution holograms and large scenes [G].
*   **Memory:** Requires storing the complex hologram plane $H(\vec{p})$, size $W \times H$ complex values [S2].
*   **Strengths:** Physically accurate modeling of diffraction from point sources; supports arbitrary 3D point distributions [S2].
*   **Limitations:** Computationally intensive for large scenes [G]; sub-hologram overlap is handled by complex summation [S2]; phase-only SLMs require approximation methods since they cannot modulate amplitude and phase simultaneously [S3].
*   **When to use:** When high-fidelity point-source reconstruction is required and computational resources allow for direct summation [S2].
*   **When not to use:** For real-time applications with very large scenes or high-resolution SLMs without acceleration techniques (e.g., GPU parallelization, FFT-based approximations) [G].

## Background
Originating from the principles of Fresnel holography, this method was formalized in the context of holographic near-eye displays and volumetric graphics [S2, S1]. The "lens phase function" and "point-wise integration" terminology was popularized by Maimone et al. [S2] for holographic projectors. The ORA (Optimization with Random Phase Algorithm) variant was developed by Hayasaki et al. [S1] for femtosecond laser processing and volumetric graphics. The extended phase range approach was introduced to address color crosstalk in simultaneous color holography [S4].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
[S3] Holographic near-eye displays based on overlap-add stereograms, https://doi.org/10.1145/3355089.3356517, fetched 2026-11-01
[S4] Simultaneous Color Computer Generated Holography, https://doi.org/10.48550/arxiv.2303.11287, fetched 2026-10-30
