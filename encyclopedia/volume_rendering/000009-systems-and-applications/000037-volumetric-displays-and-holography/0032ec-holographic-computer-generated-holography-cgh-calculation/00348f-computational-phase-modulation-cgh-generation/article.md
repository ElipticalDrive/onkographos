# Computational Phase Modulation (CGH Generation)
Also called: ORA method, Optimization with Random Phase Algorithm

## Overview
Computational Phase Modulation generates a phase-only computer-generated hologram (CGH) displayed on a spatial light modulator (SLM) to reconstruct a target 3D intensity distribution $I_r$ using iterative optimization [S1, S3]. It differs from direct phase quantization by actively optimizing the phase pattern to achieve uniform intensity across multiple focal spots or complex volumetric shapes, rather than simply encoding the argument of the complex field [S1]. It is distinct from [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) which typically relies on superposition of point sources without iterative feedback, and from [Inverse Design Algorithm for Voxel-based CGVHs](8543-inverse-design-algorithm-for-voxel-based-cgvhs.md) which may use different optimization constraints [S1, S3].

## How it works
**Inputs and Outputs:**
- Input: Desired target intensity distribution $I_r^{(d)}$ on the reconstruction plane; wavelength $\lambda$; SLM pixel pitch and resolution (implicit in propagation kernel); number of iterations.
- Output: Phase-only CGH pattern $\phi_h(x,y)$ to be displayed on the SLM.

**Data Structures:**
- Complex amplitude field $U_h(x,y)$ on the hologram plane (SLM).
- Complex amplitude field $U_r(\nu_x, \nu_y)$ on the reconstruction plane.
- Phase map $\phi_h(x,y)$ and amplitude map $a_h(x,y)$ for the hologram plane.
- Phase map $\phi_r(\nu_x, \nu_y)$ and amplitude map $a_r(\nu_x, \nu_y)$ for the reconstruction plane.
- Weight map $\omega_r$ for intensity control at reconstruction pixels.

**Formulas:**
The relationship between the hologram plane and reconstruction plane is governed by the Fourier Transform:
$$ U_r(\nu_x, \nu_y) = \iint U_h(x, y) e^{-i 2\pi (x\nu_x + y\nu_y)} dx dy = a_r(\nu_x, \nu_y) e^{i\phi_r(\nu_x, \nu_y)} $$
$$ U_h(x, y) = a_h(x, y) \exp[i\phi_h(x, y)] $$
where $a_h$ is constant (uniform plane wave illumination) [S1].

**Algorithm Steps (ORA Method):**
1. **Initialization:** Set iteration counter $i=0$. Initialize the phase of the hologram $\phi_h^{(0)}(x,y)$ randomly or uniformly. Set initial weights $\omega_r^{(0)} = 1$ for all pixels $r$ in the reconstruction plane. Set constant $\alpha$ (typically 1 or a small value for stability, though not explicitly defined in sources, it scales the weight update) [S1].
2. **Forward Propagation:** Compute the complex amplitude $U_r^{(i)}$ at the reconstruction plane by applying the Fourier Transform to $U_h^{(i)}(x,y) = a_h \exp[i\phi_h^{(i)}(x,y)]$. Extract amplitude $a_r^{(i)}$ and phase $\phi_r^{(i)}$ [S1].
3. **Weight Update:** Update the weight $\omega_r$ for each pixel $r$ in the reconstruction plane based on the ratio of desired intensity $I_r^{(d)}$ to current intensity $I_r^{(i)} = |U_r^{(i)}|^2$:
   $$ \omega_r^{(i)} = \omega_r^{(i-1)} \left( \frac{I_r^{(d)}}{I_r^{(i)}} \right)^\alpha $$
   If $I_r^{(i)}$ is zero, handle division by zero (e.g., keep previous weight or set to 1) [G].
4. **Phase Variation Calculation:** For each pixel $h$ on the hologram plane, calculate the phase variation $\Delta\phi_h^{(i)}$ using:
   $$ \Delta\phi_h^{(i)} = \tan^{-1}\left(\frac{S_2}{S_1}\right) $$
   where:
   $$ S_2 = \sum_r \omega_r^{(i)} a_h \sin(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
   $$ S_1 = \sum_r \omega_r^{(i)} a_h \cos(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
   Here, $\phi_{hr}$ is the phase contribution from hologram pixel $h$ to reconstruction pixel $r$, derived from the propagation kernel (e.g., Fresnel or Angular Spectrum phase term) [S1]. Note: The summation over $r$ implies a back-propagation or gradient-like step where contributions from all reconstruction pixels are aggregated to update the hologram phase [S1].
5. **Phase Update:** Update the hologram phase:
   $$ \phi_h^{(i)} = \phi_h^{(i-1)} + \Delta\phi_h^{(i)} $$
6. **Convergence Check:** Compute the current intensity $I_r^{(i)}$. If $I_r^{(i)}$ is sufficiently close to $I_r^{(d)}$ (e.g., mean squared error below threshold) or maximum iterations reached, stop. Otherwise, increment $i$ and return to Step 2 [S1].

**Lateral (XY) Focusing Control:**
To control focusing position in the lateral direction, the CGH is designed as a superposition of complex amplitudes of blazed gratings with various azimuth angles. If $N$ multiple focusing spots are required, the CGH includes $N$ blazed gratings [S1].

**Axial (Z) Focusing Control:**
To control focusing position in the axial direction, a phase Fresnel lens pattern is added to the hologram phase $\phi_h$:
$$ \phi_p(x, y) = \frac{k(x^2 + y^2)}{2f} $$
where $k = 2\pi/\lambda$ is the wavenumber and $f$ is the focal length. The minimum focal length is determined by the SLM's spatial resolution [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ora_cgh(target_intensity, num_iterations, alpha=1.0):
    # Initialize hologram phase randomly or uniformly
    phi_h = random_phase(hologram_shape)
    # Amplitude is constant (uniform illumination)
    a_h = 1.0
    
    # Initialize weights for reconstruction plane pixels
    weights = np.ones(reconstruction_shape)
    
    # Precompute propagation phase terms phi_hr between all h and r
    # This depends on the specific propagation model (Fresnel/ASM)
    phi_hr = compute_propagation_phase(hologram_shape, reconstruction_shape)
    
    for i in range(num_iterations):
        # 1. Forward Propagation
        # Construct complex field on hologram plane
        U_h = a_h * np.exp(1j * phi_h)
        # Fourier Transform to get reconstruction field
        U_r = np.fft.fft2(U_h)
        
        # Extract amplitude and phase
        a_r = np.abs(U_r)
        phi_r = np.angle(U_r)
        
        # Current intensity
        I_r = a_r ** 2
        
        # 2. Weight Update
        # Avoid division by zero
        safe_I_r = np.where(I_r > 1e-10, I_r, 1e-10)
        weights = weights * (target_intensity / safe_I_r) ** alpha
        
        # 3. Phase Variation Calculation
        # S2 and S1 summations over all reconstruction pixels r for each hologram pixel h
        # This requires efficient vectorization or loop over h
        S1 = np.zeros(hologram_shape)
        S2 = np.zeros(hologram_shape)
        
        for h in hologram_pixels:
            # Vectorized operation over all r
            # phi_hr[h, r] is the phase from h to r
            phase_diff = phi_r - phi_hr[h, :] - phi_h[h]
            
            # S1 and S2 components
            # Note: a_h is constant, can be factored out or kept inside
            term_sin = weights * a_h * np.sin(phase_diff)
            term_cos = weights * a_h * np.cos(phase_diff)
            
            S1[h] = np.sum(term_sin) # Wait, S2 is sin, S1 is cos in source? 
            # Source: S2 = sum ... sin(...), S1 = sum ... cos(...)
            # Correction:
            S1[h] = np.sum(weights * a_h * np.cos(phase_diff))
            S2[h] = np.sum(weights * a_h * np.sin(phase_diff))
            
        # 4. Update Phase
        delta_phi_h = np.arctan2(S2, S1)
        phi_h = phi_h + delta_phi_h
        
        # Check convergence (optional)
        if np.mean((I_r - target_intensity)**2) < threshold:
            break
            
    return phi_h
```

## Subtopics

### CGH-based parallel focal point generation
Generates multiple focal points in parallel using a phase-only CGH displayed on an LCOS-SLM to increase voxel generation density per frame [S2]. It uses the Gerchberg–Saxton method for optimization (50 Fourier iterative calculations) rather than ORA in the cited implementation, mapping grayscale values 0–255 to phase modulation 0–$2\pi$ [S2]. The number of voxels generated per frame is estimated by $N = P n / f_r$, where $P$ is the number of parallel focal points, $n$ is the number of voxels generated per second (dependent on laser repetition frequency), and $f_r$ is the refresh rate [S2]. A pitfall is that the actual number of parallelized voxels $P$ may be significantly less than estimated due to irradiated pulse energy limits required to prevent varifocal lens damage; this is handled by using varifocal lenses with higher light resistance or combining with dynamic optimization for CGH diffraction efficiency [S2]. In the cited experiment, estimated $P=708$ but actual $P=4$ was achieved [S2].

## Operating characteristics
- **Time Complexity:** Iterative methods like ORA and Gerchberg-Saxton require multiple forward and backward propagations (Fourier transforms). Each iteration is $O(N \log N)$ where $N$ is the number of pixels in the hologram plane. Convergence speed depends on the target complexity and the number of iterations (e.g., 50 iterations cited in [S2]).
- **Memory Complexity:** Requires storing complex fields for the hologram and reconstruction planes, plus propagation kernels. $O(N)$ memory.
- **Performance:** ORA allows high-quality reconstruction with uniform intensity across spot arrays [S1]. Parallel focal point generation increases voxel density per frame, but is limited by laser pulse energy and SLM refresh rate [S2].
- **Strengths:** Can generate complex 3D intensity distributions with high uniformity; allows independent control of lateral and axial focusing positions [S1].
- **Limitations:** Iterative convergence is not guaranteed and can be slow; sensitive to initial phase; requires precise calibration of the SLM and optical system; parallel generation is limited by hardware constraints (laser energy, lens damage thresholds) [S1, S2].
- **When to use:** When high-quality, uniform intensity distribution is required for multiple focal points or complex shapes, and computational time is acceptable.
- **When not to use:** When real-time performance is critical and iterative methods are too slow, or when hardware constraints (laser energy) prevent the desired number of parallel focal points [S2].

## Background
The ORA (Optimization with Random Phase Algorithm) method was introduced by Hayasaki et al. in 2005 for controlling laser focusing positions in 3D space using a liquid crystal spatial light modulator (LCSLM) [S1]. It is based on adding adequate phase variation calculated by an iterative optimization process into the CGH to obtain a reconstruction composed of a spot array with uniform intensity [S1]. The method leverages the LCSLM's ability to act as an optical phased array, where molecule directions are controlled by electrodes (pixels) to modulate the phase of reflected light [S1]. Parallel focal point generation using CGH was further explored in 2021 for volumetric displays, utilizing the Gerchberg-Saxton method to optimize CGHs for generating multiple focal points in a drawing space separation system [S2].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
