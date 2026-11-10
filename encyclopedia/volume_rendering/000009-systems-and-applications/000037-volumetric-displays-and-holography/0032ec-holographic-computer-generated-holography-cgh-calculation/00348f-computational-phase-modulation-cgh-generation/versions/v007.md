# Computational Phase Modulation (CGH Generation)
Also called: ORA method, Optimization with Random Phase Algorithm

## Overview
Computational Phase Modulation generates a phase-only Computer-Generated Hologram (CGH) displayed on a Liquid Crystal Spatial Light Modulator (LCOS-SLM) to reconstruct a desired 3D light field, typically a uniform array of focal spots for volumetric display. It solves the problem of encoding complex amplitude information into a phase-only constraint using the Optimization with Random Phase Algorithm (ORA). It differs from direct diffraction calculations by iteratively optimizing phase variations to maximize reconstruction intensity uniformity, rather than relying solely on analytical propagation models [S1, S2].

## How it works
**Inputs:**
- Desired intensity distribution $I_r^{(d)}$ at the reconstruction plane (e.g., a grid of spots).
- Wavelength $\lambda$ and propagation distance (focal length) $f$.
- SLM pixel grid dimensions.

**Outputs:**
- Phase-only CGH pattern $\phi_h(x,y)$ to be displayed on the SLM.

**Data Structures:**
- **Hologram Plane:** 2D array of phase values $\phi_h(x,y)$. Amplitude $a_h(x,y)$ is constant (normalized to 1) because the incident light is a uniform plane wave [S1].
- **Reconstruction Plane:** 2D array of complex amplitudes $U_r(\nu_x, \nu_y)$ and intensities $I_r = |U_r|^2$.
- **Weights:** Array $\omega_r$ representing phase weights at reconstruction pixels, updated iteratively [S1].

**Formulas:**
1.  **Propagation (Fourier Transform):**
    The complex amplitude $U_r$ at the reconstruction plane is the Fourier transform of the hologram field $U_h$:
    $$ U_r(\nu_x, \nu_y) = \iint U_h(x, y) e^{-i 2\pi (x\nu_x + y\nu_y)} dx dy $$
    $$ U_h(x, y) = a_h(x, y) \exp[i\phi_h(x, y)] $$
    where $a_h$ is constant [S1].

2.  **Lateral (XY) Control:**
    For $N$ multiple focusing spots, the CGH includes a superposition of $N$ blazed gratings with various azimuth angles [S1].

3.  **Axial (Z) Control:**
    A phase Fresnel lens pattern is added to the base phase $\phi_h$:
    $$ \phi_p(x, y) = \frac{k(x^2 + y^2)}{2f} $$
    where $k = 2\pi/\lambda$ is the wavenumber and $f$ is the focal length [S1].

4.  **ORA Iteration:**
    The goal is to maximize $\sum_r |U_r^{(i)}|^2$ to match $I_r^{(d)}$.
    
    Update phase variation $\Delta\phi_h^{(i)}$ at pixel $h$:
    $$ \Delta\phi_h^{(i)} = \tan^{-1}\left(\frac{S_2}{S_1}\right) $$
    where:
    $$ S_2 = \sum_r \omega_r^{(i)} a_h \sin(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
    $$ S_1 = \sum_r \omega_r^{(i)} a_h \cos(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
    Here, $\phi_{hr}$ is the phase contribution from pixel $h$ to pixel $r$ due to propagation [S1].

    Update CGH phase:
    $$ \phi_h^{(i)} = \phi_h^{(i-1)} + \Delta\phi_h^{(i)} $$
    [S1]

    Update reconstruction weight $\omega_r$:
    $$ \omega_r^{(i)} = \omega_r^{(i-1)} \left( \frac{I_r^{(d)}}{I_r^{(i)}} \right)^\alpha $$
    where $I_r^{(i)} = |U_r^{(i)}|^2$ is the current intensity, $I_r^{(d)}$ is the desired intensity, and $\alpha$ is a constant [S1].

**Algorithm Steps:**
1.  Initialize phase $\phi_h^{(0)}$ randomly or with a base pattern (e.g., Fresnel lens for Z-control, blazed gratings for XY-control) [S1].
2.  Initialize weights $\omega_r^{(0)} = 1$ (or appropriate initial guess) [G].
3.  For iteration $i = 1$ to $N_{iter}$:
    a. Compute complex field $U_h^{(i-1)}(x,y) = \exp[i\phi_h^{(i-1)}(x,y)]$ (assuming $a_h=1$) [S1].
    b. Compute reconstruction field $U_r^{(i)}$ via Fourier Transform of $U_h^{(i-1)}$ [S1].
    c. Compute intensity $I_r^{(i)} = |U_r^{(i)}|^2$ [S1].
    d. Calculate $S_1$ and $S_2$ for each hologram pixel $h$ using current $\phi_r^{(i)}$ (phase of $U_r^{(i)}$), $\phi_{hr}$, and $\omega_r^{(i)}$ [S1].
    e. Calculate $\Delta\phi_h^{(i)} = \tan^{-1}(S_2/S_1)$ [S1].
    f. Update $\phi_h^{(i)} = \phi_h^{(i-1)} + \Delta\phi_h^{(i)}$ [S1].
    g. Update weights $\omega_r^{(i)} = \omega_r^{(i-1)} (I_r^{(d)} / I_r^{(i)})^\alpha$ [S1].
4.  Stop when $I_r^{(i)} \approx I_r^{(d)}$ (convergence) [S1].

**Parameters:**
-   $a_h$: Constant amplitude of hologram plane. Set to 1 (uniform plane wave illumination) [S1].
-   $\alpha$: Constant in weight update. Value not specified in sources [S1].
-   $N_{iter}$: Number of iterations. Typically 50 for Gerchberg-Saxton based optimizations in similar contexts, though ORA convergence criteria is "nearly equal" [S1, S2].

**Edge Cases and Pitfalls:**
-   **Precision:** The inverse tangent function $\tan^{-1}$ requires careful handling of quadrants (use `atan2` in implementation) to ensure correct phase wrapping [S1].
-   **Boundary:** The SLM spatial resolution determines the minimum focal length $f$ for Z-control [S1].
-   **Hardware Limits:** In parallel focal point generation, the actual number of parallel voxels $P$ may be significantly lower than estimated due to laser pulse energy limits required to prevent damage to optical components (e.g., varifocal lenses) [S2].

## Subtopics

### CGH-based parallel focal point generation
Generates multiple focal points in parallel using a phase-only CGH displayed on an LCOS-SLM to increase voxel generation density per frame [S2]. It uses a lens (focal length 200 mm) and varifocal lens to focus light into separated focal points in a drawing space at infinite distance [S2]. The CGH is optimized using 50 Fourier iterative calculations based on the Gerchberg–Saxton method [S2]. The number of voxels generated per frame is estimated by $N = P n / f_r$, where $P$ is parallel voxels, $n$ is voxels generated per second (dependent on laser repetition frequency), and $f_r$ is refresh rate [S2]. Typical parameters include grayscale 0–255 mapped to phase 0–2$\pi$, threshold energy $E_{th} = 0.004$ mJ, and duty cycle $D=50\%$ [S2]. A pitfall is that actual parallel voxels $P$ (e.g., 4) may be much lower than estimated (e.g., 708) due to energy limits to protect the varifocal lens; this is mitigated by using higher light-resistance lenses or dynamic optimization for diffraction efficiency [S2].

## Operating characteristics
-   **Time Complexity:** Iterative. Each iteration requires a forward and backward Fourier Transform (or equivalent propagation). Convergence depends on the complexity of the target intensity distribution [S1, S2].
-   **Memory Complexity:** Proportional to the SLM resolution (2D arrays for phase, amplitude, and weights) [S1].
-   **Performance:** Enables high-quality spot arrays with uniform intensity [S1]. In parallel generation, estimated voxel throughput can reach 35,400 voxels/frame at 20 Hz refresh rate with specific laser parameters, though practical limits reduce this [S2].
-   **Strengths:** Allows precise lateral (XY) and axial (Z) focusing control via blazed gratings and Fresnel lenses respectively [S1]. Compatible with phase-only LCOS-SLMs [S1, S2].
-   **Limitations:** Convergence is not guaranteed to be fast. Hardware constraints (laser energy, lens damage thresholds) can severely limit the number of parallel focal points compared to theoretical estimates [S2].

## Background
Originated from the need to control light intensity and phase for aerial and volumetric graphics using femtosecond lasers [S1]. The ORA method was developed to optimize CGHs for uniform spot arrays, building on phase-only modulation capabilities of LCOS-SLMs [S1]. It is used in volumetric displays that excite voxels in a medium (e.g., Xe-filled glass cell) using focused laser beams [S2]. The method combines analytical phase patterns (gratings, lenses) with iterative optimization to handle the phase-only constraint [S1, S2].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
