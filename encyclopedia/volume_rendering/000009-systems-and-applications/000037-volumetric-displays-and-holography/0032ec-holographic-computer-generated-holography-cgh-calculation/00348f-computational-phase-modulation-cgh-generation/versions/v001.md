# Computational Phase Modulation (CGH Generation)
Also called: ORA method, Optimization with Random Phase Algorithm

## Overview
Computational Phase Modulation generates Computer-Generated Holograms (CGHs) to control the phase of a laser beam reflected from a Liquid Crystal Spatial Light Modulator (LCSLM), enabling the creation of arbitrary 3D focal spot arrays for volumetric displays. It solves the problem of mapping desired 3D voxel intensities to the phase-only constraints of an SLM by iteratively optimizing the hologram phase to match a target intensity distribution in the reconstruction plane. It differs from [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by explicitly using an iterative optimization algorithm (ORA) to resolve phase ambiguities for uniform intensity spot arrays, rather than direct analytical mapping or inverse design.

## How it works
**Inputs and Outputs:**
- **Input:** Target intensity distribution $I_r^{(d)}$ on the reconstruction plane (defined by desired voxel positions and intensities); wavelength $\lambda$; SLM pixel pitch (implicit in Fourier transform limits).
- **Output:** Phase pattern $\phi_h(x, y)$ to be displayed on the LCSLM.

**Data Structures:**
- **Hologram Plane:** 2D grid of complex amplitude $U_h(x, y) = a_h(x, y) \exp[i\phi_h(x, y)]$.
    - $a_h(x, y)$: Amplitude. Constant across the SLM because the incident laser is a plane wave with uniform intensity.
    - $\phi_h(x, y)$: Phase. The variable optimized by the algorithm.
- **Reconstruction Plane:** 2D grid of complex amplitude $U_r(\nu_x, \nu_y) = a_r(\nu_x, \nu_y) \exp[i\phi_r(\nu_x, \nu_y)]$.
    - $\nu_x, \nu_y$: Spatial frequencies corresponding to lateral coordinates.
    - $a_r, \phi_r$: Amplitude and phase of the reconstructed field.
- **Weights:** $\omega_r^{(i)}$: Phase weight at pixel $r$ on the reconstruction plane during iteration $i$.

**Formulas:**
1.  **Fourier Propagation:** The complex amplitude in the reconstruction plane is the Fourier transform of the hologram plane:
    $$ U_r(\nu_x, \nu_y) = \iint U_h(x, y) e^{-i 2\pi (x\nu_x + y\nu_y)} dx dy $$
2.  **Axial Control (Z-axis):** To focus at depth $f$, a phase Fresnel lens pattern is added to the hologram phase:
    $$ \phi_p(x, y) = \frac{k(x^2 + y^2)}{2f} $$
    where $k = 2\pi/\lambda$ is the wave number. This is added to the base hologram phase $\phi_h$.
3.  **Lateral Control (XY-axis):** For $N$ multiple focusing spots, the CGH is designed as a superposition of complex amplitudes of blazed gratings with various azimuth angles.

**Algorithm Steps (Optimization with Random Phase Algorithm - ORA):**
The algorithm iteratively updates the hologram phase $\phi_h$ and reconstruction weights $\omega_r$ to minimize the difference between current intensity $I_r^{(i)}$ and desired intensity $I_r^{(d)}$.

1.  **Initialize:** Set iteration counter $i = 1$. Initialize hologram phase $\phi_h^{(0)}$ (e.g., random or zero). Initialize reconstruction weights $\omega_r^{(0)} = 1$ (or based on initial guess). Set amplitude $a_h$ to constant.
2.  **Forward Propagation:** Calculate the complex amplitude $U_r^{(i)}$ at the reconstruction plane by performing the Fourier transform of $U_h^{(i)} = a_h \exp[i\phi_h^{(i)}]$.
    $$ U_r^{(i)} = \mathcal{F}\{ a_h \exp[i\phi_h^{(i)}] \} $$
    Extract amplitude $a_r^{(i)}$ and phase $\phi_r^{(i)}$ from $U_r^{(i)}$.
3.  **Calculate Phase Variation:** For each pixel $h$ on the hologram plane, calculate the phase update $\Delta\phi_h^{(i)}$ using the weighted sum of contributions from all reconstruction pixels $r$:
    $$ \Delta\phi_h^{(i)} = \tan^{-1}\left(\frac{S_2}{S_1}\right) $$
    where:
    $$ S_2 = \sum_r \omega_r^{(i)} a_h \sin(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
    $$ S_1 = \sum_r \omega_r^{(i)} a_h \cos(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
    Here, $\phi_{hr}$ is the phase contribution from hologram pixel $h$ to reconstruction pixel $r$ due to propagation (part of the Fourier kernel).
4.  **Update Hologram Phase:**
    $$ \phi_h^{(i)} = \phi_h^{(i-1)} + \Delta\phi_h^{(i)} $$
5.  **Update Reconstruction Weights:** Update the phase weight $\omega_r$ at the reconstruction plane to steer intensity towards the target:
    $$ \omega_r^{(i)} = \omega_r^{(i-1)} \left( \frac{I_r^{(d)}}{I_r^{(i)}} \right)^\alpha $$
    where $I_r^{(i)} = |U_r^{(i)}|^2$ is the current intensity, $I_r^{(d)}$ is the desired intensity, and $\alpha$ is a constant (typically $\alpha=1$ or similar, source does not specify exact value, but implies a ratio-based update). [G] $\alpha$ is a convergence parameter; if not specified, use $\alpha=1$.
6.  **Check Convergence:** If $I_r^{(i)}$ is nearly equal to $I_r^{(d)}$ (e.g., error metric below threshold) or maximum iterations reached, stop. Otherwise, increment $i$ and return to Step 2.

**Parameters:**
- $\lambda$: Wavelength of the laser.
- $f$: Focal length for axial positioning.
- $\alpha$: Constant in weight update. Not given in the sources.
- $a_h$: Constant amplitude.

**Edge Cases and Pitfalls:**
- **Precision:** The algorithm relies on accurate Fourier transforms. Numerical precision in the phase update $\tan^{-1}(S_2/S_1)$ must be maintained to avoid phase wrapping errors.
- **Boundaries:** The spatial resolution of the SLM determines the minimum focal length achievable for axial control.
- **Degenerate Input:** If $I_r^{(d)}$ is zero at a pixel, the weight update may become unstable. [G] Handle by clamping $I_r^{(d)}$ to a small positive value or skipping update for zero-intensity targets.

## Subtopics

### CGH-based parallel focal point generation
Generates multiple focal points in parallel using a phase-only CGH displayed on an LCOS-SLM to increase voxel generation density per frame. It uses the Gerchberg–Saxton method for 50 Fourier iterative calculations to optimize the CGH, which is then used with a lens and varifocal lens to focus light into separated focal points. The number of voxels generated per frame $N$ is calculated as $N = P n / f_r$, where $P$ is the number of parallel focal points, $n$ is the number of voxels generated in 1 second (dependent on laser repetition frequency), and $f_r$ is the refresh rate. A pitfall is that the actual number of parallel voxels $P$ achieved (e.g., $P=4$) may be significantly less than estimated (e.g., $P=708$) due to irradiated pulse energy reduction to prevent varifocal lens damage. This is handled by using a varifocal lens with higher light resistance or combining with dynamic optimization for CGH diffraction efficiency. Parameters include a lens focal length of 200 mm, duty cycle $D=50\%$, and threshold energy $E_{th} = 0.004$ mJ.

## Operating characteristics
- **Time Complexity:** Iterative. Each iteration requires a forward Fourier transform and phase updates. Convergence depends on the complexity of the target intensity distribution.
- **Memory Complexity:** Requires storage for the hologram plane ($N_h \times N_h$ complex numbers) and reconstruction plane ($N_r \times N_r$ complex numbers).
- **Measured Performance:** With a femtosecond laser, estimated $P=708$ parallel voxels per pulse, leading to $N=35,400$ voxels per frame at $f_r = 20$ Hz. Actual performance was limited to $P=4$ due to hardware constraints (varifocal lens damage threshold).
- **Strengths:** Enables high-density voxel generation per frame by parallelizing focal points. Allows precise 3D positioning (XY and Z) of voxels.
- **Limitations:** Performance is limited by the laser repetition rate and the damage threshold of optical components (varifocal lens). The iterative optimization can be computationally expensive.
- **When to use:** When high voxel density per frame is required and the hardware supports high-repetition-rate lasers and robust optical components.
- **When not to use:** When hardware limitations (lens damage) prevent achieving the estimated parallelization factor $P$.

## Background
The method originates from research into femtosecond laser volumetric displays. The ORA method was introduced by Hayasaki et al. in 2005 for laser processing to generate arbitrary patterns of laser spots [S1]. It was later adapted for volumetric graphics rendering, combining computational holography with femtosecond laser excitation of gases (e.g., Xe-filled glass cells) to create visible voxels [S1, S2]. The parallel focal point generation variant was proposed to increase the efficiency of voxel generation in such displays [S2].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
