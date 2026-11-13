# Computational Phase Modulation (CGH Generation)
Also called: ORA method, Optimization with Random Phase Algorithm

## Overview
Computational Phase Modulation generates Computer-Generated Holograms (CGH) by iteratively optimizing the phase profile of a spatial light modulator (SLM) to produce a desired intensity distribution in the reconstruction plane, typically a uniform array of focal spots for volumetric display [S1]. It solves the problem of driving phase-only hardware (such as Liquid Crystal SLMs) with complex-valued light fields derived from volumetric rendering by discarding amplitude information and encoding the required phase variations through iterative optimization [S1]. Unlike direct diffraction calculations, this method uses an optimization loop to converge the reconstructed intensity $I_r$ to a target intensity $I_r^{(d)}$, allowing for precise control over lateral (XY) and axial (Z) focusing positions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Target intensity distribution $I_r^{(d)}$ on the reconstruction plane; wavelength $\lambda$; SLM pixel pitch (implicit in Fourier domain scaling); number of iterations.
*   **Output:** Phase map $\phi_h(x,y)$ to be displayed on the SLM.

**Data Structures:**
*   **Hologram Plane:** Complex amplitude array $U_h(x,y)$ of size $W \times H$ (SLM resolution).
*   **Reconstruction Plane:** Complex amplitude array $U_r(\nu_x, \nu_y)$ of size $W \times H$ (Fourier plane).
*   **Weights:** Array $\omega_r$ of size $W \times H$ storing phase weights for the reconstruction plane.

**Formulas:**
The relationship between the hologram plane and reconstruction plane is governed by the Fourier Transform:
$$ U_r(\nu_x, \nu_y) = \iint U_h(x, y) e^{-i 2\pi (x\nu_x + y\nu_y)} dx dy = a_r(\nu_x, \nu_y) e^{i\phi_r(\nu_x, \nu_y)} $$
where $a_r$ and $\phi_r$ are the amplitude and phase of the reconstruction [S1].
The hologram plane complex amplitude is defined as:
$$ U_h(x, y) = a_h(x, y) \exp[i\phi_h(x, y)] $$
where $a_h$ is the amplitude of the hologram plane. In the standard ORA implementation, $a_h$ is constant because the irradiation light is a plane wave with uniform intensity distribution [S1].

**Steps:**
1.  **Initialization:** Initialize the phase map $\phi_h^{(0)}(x,y)$ randomly or uniformly. Set iteration counter $i=0$. Initialize weights $\omega_r^{(0)}$ (e.g., to 1 or based on target intensity).
2.  **Forward Propagation:** Calculate the complex amplitude $U_r^{(i)}$ at the reconstruction plane by performing a Fourier Transform on $U_h^{(i)}$:
    $$ U_h^{(i)}(x, y) = a_h \exp[i\phi_h^{(i)}(x, y)] $$
    $$ U_r^{(i)} = \mathcal{F}\{U_h^{(i)}\} $$
    Calculate the current intensity $I_r^{(i)} = |U_r^{(i)}|^2$ and phase $\phi_r^{(i)} = \arg(U_r^{(i)})$ [S1].
3.  **Weight Update:** Update the phase weights $\omega_r^{(i)}$ on the reconstruction plane to guide the intensity toward the target $I_r^{(d)}$:
    $$ \omega_r^{(i)} = \omega_r^{(i-1)} \left( \frac{I_r^{(d)}}{I_r^{(i)}} \right)^\alpha $$
    where $\alpha$ is a constant (typically $\alpha=1$ or a small value for stability) [S1].
4.  **Phase Variation Calculation:** For each pixel $h$ on the hologram plane, calculate the phase variation $\Delta\phi_h^{(i)}$ required to maximize the sum of light intensity at the reconstruction pixels $r$. This involves summing contributions from all reconstruction pixels $r$:
    $$ S_2 = \sum_r \omega_r^{(i)} a_h \sin(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
    $$ S_1 = \sum_r \omega_r^{(i)} a_h \cos(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
    $$ \Delta\phi_h^{(i)} = \tan^{-1}\left(\frac{S_2}{S_1}\right) $$
    where $\phi_{hr}$ is the phase contribution from hologram pixel $h$ to reconstruction pixel $r$ (derived from the Fourier kernel phase) [S1].
5.  **Phase Update:** Update the hologram phase map:
    $$ \phi_h^{(i)} = \phi_h^{(i-1)} + \Delta\phi_h^{(i)} $$
6.  **Convergence Check:** If $I_r^{(i)}$ is nearly equal to $I_r^{(d)}$ (within a threshold) or the maximum number of iterations is reached, stop. Otherwise, increment $i$ and return to Step 2 [S1].

**Lateral (XY) and Axial (Z) Control:**
*   **Lateral Control:** To focus at multiple lateral positions, the CGH is designed as a superposition of complex amplitudes of blazed gratings with various azimuth angles. If $N$ focusing spots are required, the CGH includes $N$ blazed gratings [S1].
*   **Axial Control:** To control the focal depth (Z), a phase Fresnel lens pattern is added to the optimized phase $\phi_h$:
    $$ \phi_p(x, y) = \frac{k(x^2 + y^2)}{2f} $$
    where $k = 2\pi/\lambda$ is the wavenumber and $f$ is the focal length. The final phase becomes $\phi_h^{final} = \phi_h + \phi_p$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ora_cgh(target_intensity, wavelength, num_iterations, alpha=1.0):
    """
    Computes the phase-only CGH using the Optimization with Random Phase Algorithm (ORA).
    
    Args:
        target_intensity: 2D array, desired intensity distribution I_r^(d).
        wavelength: float, wavelength lambda in meters.
        num_iterations: int, number of ORA iterations.
        alpha: float, convergence constant for weight update.
        
    Returns:
        phase_map: 2D array, the optimized phase phi_h(x,y).
    """
    # Dimensions
    H, W = target_intensity.shape
    
    # Constants
    k = 2 * math.pi / wavelength
    
    # 1. Initialization
    # Initialize phase map randomly or uniformly
    phase_map = np.random.uniform(0, 2 * math.pi, (H, W))
    
    # Amplitude of hologram plane is constant (plane wave illumination)
    a_h = 1.0 
    
    # Initialize weights on reconstruction plane
    # Weights omega_r guide the intensity. Start with uniform or target-based weights.
    weights = np.ones((H, W))
    
    # Precompute Fourier kernel phases for efficiency if needed, 
    # but standard FFT handles the complex exponential.
    # phi_hr is implicit in the FFT operation.
    
    for i in range(1, num_iterations + 1):
        # 2. Forward Propagation
        # Construct complex hologram field
        # U_h = a_h * exp(i * phi_h)
        U_h = a_h * np.exp(1j * phase_map)
        
        # Fourier Transform to get reconstruction field
        # U_r = FFT(U_h)
        U_r = np.fft.fft2(U_h)
        
        # Calculate current intensity and phase
        I_r = np.abs(U_r) ** 2
        phi_r = np.angle(U_r)
        
        # Avoid division by zero in weight update
        I_r_safe = np.where(I_r > 1e-10, I_r, 1e-10)
        
        # 3. Weight Update
        # omega_r^(i) = omega_r^(i-1) * (I_r^(d) / I_r^(i))^alpha
        weights = weights * (target_intensity / I_r_safe) ** alpha
        
        # 4. Phase Variation Calculation
        # We need to calculate delta_phi_h for each pixel h.
        # This requires summing contributions from all r.
        # Direct summation is O(N^2). Using FFT properties or explicit kernel summation.
        # The formula for delta_phi_h involves S1 and S2.
        
        delta_phase_map = np.zeros((H, W))
        
        # To implement S1 and S2 efficiently without O(N^2) loops:
        # Note: The standard ORA derivation often uses the gradient of the cost function.
        # The provided formula is:
        # S2 = sum_r omega_r * a_h * sin(phi_r - phi_hr - phi_h)
        # S1 = sum_r omega_r * a_h * cos(phi_r - phi_hr - phi_h)
        # phi_hr is the phase of the kernel exp(-i 2pi (x nu_x + y nu_y))
        
        # Let's construct the kernel explicitly for clarity, though FFT is faster for propagation.
        # For a pixel h=(x,y), phi_hr(x,y, nu_x, nu_y) = -2pi*(x*nu_x + y*nu_y)
        # However, the summation is over r (nu_x, nu_y).
        
        # Create coordinate grids
        x, y = np.meshgrid(np.arange(W), np.arange(H))
        # Normalize coordinates for Fourier domain
        # Assuming unit spacing, frequencies are in cycles/pixel
        nu_x, nu_y = np.meshgrid(np.fft.fftfreq(W), np.fft.fftfreq(H))
        
        # Kernel phase phi_hr for each h and r
        # phi_hr = -2 * pi * (x * nu_x + y * nu_y)
        # This is a 4D tensor if done naively. 
        # Instead, we can use the property that the gradient of the intensity 
        # with respect to phase can be computed via inverse FFT of weighted fields.
        
        # Construct the weighted field in frequency domain:
        # W_r = omega_r * exp(i * phi_r) ? No, the formula uses sin/cos differences.
        # Let's rewrite the sum:
        # Sum_r omega_r * exp(i * (phi_r - phi_hr)) * exp(-i * phi_h)
        # = exp(-i * phi_h) * Sum_r omega_r * exp(i * phi_r) * exp(-i * phi_hr)
        
        # Let V_r = omega_r * exp(i * phi_r)
        V_r = weights * np.exp(1j * phi_r)
        
        # The term Sum_r V_r * exp(-i phi_hr) is the Inverse FFT of V_r?
        # phi_hr is the phase of the forward FFT kernel.
        # Forward FFT: U_r = sum_h U_h * exp(-i phi_hr)
        # Inverse FFT: U_h = sum_r U_r * exp(i phi_hr)
        
        # So, Sum_r V_r * exp(-i phi_hr) is NOT a standard FFT.
        # However, note that exp(-i phi_hr) is the kernel of the Forward FFT.
        # Let's look at the structure:
        # S1 - i S2 = sum_r omega_r * a_h * exp(i(phi_r - phi_hr - phi_h))
        #            = a_h * exp(-i phi_h) * sum_r omega_r * exp(i phi_r) * exp(-i phi_hr)
        
        # Let Q_r = omega_r * exp(i phi_r).
        # We need sum_r Q_r * exp(-i phi_hr).
        # This is equivalent to the Inverse FFT of Q_r if we swap signs?
        # Standard IFFT: sum_r Q_r * exp(+i 2pi (x nu_x + y nu_y))
        # Our kernel: exp(-i 2pi (x nu_x + y nu_y))
        # So it is the FFT of the conjugate or similar.
        
        # Actually, let's use the explicit summation for correctness in pseudocode, 
        # acknowledging it is O(N^2) per pixel, or use the FFT trick.
        # FFT Trick:
        # sum_r Q_r * exp(-i 2pi (x nu_x + y nu_y))
        # = IFFT( Q_r ) evaluated at (x,y) ? No.
        # IFFT(Q) = sum_r Q_r * exp(+i 2pi (x nu_x + y nu_y)) / N
        # We have a minus sign. So it is FFT(Q) conjugated? Or IFFT(Q*)?
        # Let's stick to the definition:
        # Let K(x,y,nu_x,nu_y) = exp(-i 2pi (x nu_x + y nu_y))
        # We want sum_r Q_r * K(x,y,nu_x,nu_y).
        # This is the Inverse Fourier Transform of Q_r if we define IFT with + sign?
        # Standard IFT: f(x) = int F(nu) exp(i 2pi nu x) dnu.
        # Our kernel has minus. So it is the Forward Fourier Transform of Q_r?
        # FFT(Q) = sum_r Q_r exp(-i 2pi (x nu_x + y nu_y)).
        # Yes. So sum_r Q_r * exp(-i phi_hr) is simply FFT(Q_r).
        
        Q_r = weights * np.exp(1j * phi_r)
        # Compute FFT of Q_r
        # Note: np.fft.fft2 computes sum_n sum_m x[m,n] exp(-i 2pi (um/M + vn/N))
        # This matches our kernel definition.
        sum_term = np.fft.fft2(Q_r)
        
        # sum_term is a complex array of shape (H, W)
        # sum_term[h] = sum_r Q_r * exp(-i phi_hr)
        
        # Now, S1 - i S2 = a_h * exp(-i phi_h) * sum_term
        # Let Z = a_h * exp(-i phi_h) * sum_term
        Z = a_h * np.exp(-1j * phase_map) * sum_term
        
        # S1 = Re(Z), S2 = Im(Z)
        S1 = np.real(Z)
        S2 = np.imag(Z)
        
        # Calculate delta_phi_h
        # delta_phi_h = atan2(S2, S1)
        delta_phase_map = np.arctan2(S2, S1)
        
        # 5. Phase Update
        phase_map = phase_map + delta_phase_map
        
        # Optional: Normalize phase to [0, 2pi]
        phase_map = np.mod(phase_map, 2 * math.pi)
        
    return phase_map
```

**Edge Cases and Pitfalls:**
*   **Precision:** The iterative update $\Delta\phi_h$ can accumulate numerical errors. Phase wrapping ($\mod 2\pi$) should be handled carefully to avoid discontinuities if the update step is large.
*   **Boundaries:** The Fourier Transform assumes periodic boundary conditions. If the target intensity has sharp edges, ringing artifacts may occur. Windowing functions can be applied but may reduce efficiency.
*   **Degenerate Input:** If the target intensity $I_r^{(d)}$ is zero at some pixels, the weight update involves division by zero. A small epsilon floor (e.g., $10^{-10}$) must be applied to $I_r$ before division [S1, G].
*   **Convergence:** The algorithm may converge to a local minimum. The choice of $\alpha$ and the initialization of $\phi_h$ affect convergence speed and quality [S1].

## Subtopics

### [Phase-Only Hologram Approximations](13999-phase-only-hologram-approximations.md)
Phase-only hologram approximations are techniques for generating holograms suitable for Liquid Crystal Spatial Light Modulators (LCSLMs) by discarding or encoding the amplitude information of a complex object wave into phase values. They solve the problem of driving phase-only hardware with complex-valued light fields derived from volumetric rendering. The amplitude discard method is the simplest approach, keeping only the phase, while the double phase and binary phase methods encode amplitude and phase into two or two-level phase values respectively, requiring specific optical filtering or temporal averaging.

### CGH-based parallel focal point generation
This variant generates multiple focal points in parallel using a phase-only CGH displayed on an LCOS-SLM to increase voxel generation density per frame [S2]. It optimizes the CGH using Fourier iterative calculations (e.g., Gerchberg–Saxton) to control light into separated focal points in a drawing space at infinite distance [S2]. The number of voxels generated per frame $N$ is estimated by $N = P n / f_r$, where $P$ is the number of parallel focal points, $n$ is the number of voxels generated in 1 second (dependent on laser repetition frequency), and $f_r$ is the refresh rate [S2]. A key pitfall is that the actual number of parallelized voxels $P$ may be significantly less than estimated due to irradiated pulse energy limits imposed by varifocal lens damage thresholds (e.g., $E_{th} = 0.004$ mJ) [S2]. For example, with a 200 mm focal length lens and 50% duty cycle, $P$ was estimated at 708 but achieved only 4 in demonstration due to energy constraints [S2].

## Operating characteristics
*   **Time Complexity:** The naive implementation of the ORA method involves an iterative loop where each iteration requires a Forward FFT and an Inverse FFT (or equivalent summation). Each FFT is $O(N \log N)$ where $N$ is the number of pixels. The weight update and phase calculation are $O(N)$. Thus, one iteration is $O(N \log N)$. Total time is $O(I \cdot N \log N)$ where $I$ is the number of iterations [S1, G].
*   **Memory Complexity:** $O(N)$ to store the hologram plane, reconstruction plane, weights, and intermediate arrays [S1, G].
*   **Measured Performance:** In a demonstration using a femtosecond laser and LCOS-SLM, the method enabled the generation of volumetric graphics. With a refresh rate of 20 Hz and estimated parameters, up to 35,400 voxels per frame were estimated, though practical limits reduced this to 4 parallel points in the specific setup due to hardware constraints [S2].
*   **Strengths:** Allows for high-quality reconstruction of arbitrary intensity patterns (e.g., uniform spot arrays) on phase-only devices. Provides independent control over lateral and axial focusing positions [S1].
*   **Limitations:** Computationally intensive due to iterative nature. Sensitive to hardware limitations such as SLM resolution and laser power constraints [S1, S2]. Convergence is not guaranteed to be global [S1].
*   **When to use:** When high-quality, arbitrary intensity distributions are required on phase-only SLMs, and computational resources allow for iterative optimization.
*   **When not to use:** When real-time performance is critical and iterative methods are too slow, or when amplitude modulation is required and cannot be approximated by phase-only methods.

## Background
The Computational Phase Modulation method, specifically the Optimization with Random Phase Algorithm (ORA), was introduced by Hayasaki et al. in 2005 [S1]. It was developed to enable the control of light intensity at multiple points using a Liquid Crystal SLM (LCSLM) acting as an optical phased array. The method builds on the principle that spatial phase control of light enables control of focusing position along both lateral (XY) and axial (Z) directions [S1]. It is used in volumetric displays and laser processing to generate arbitrary patterns of laser spots [S1]. The method has been extended to parallel focal point generation for volumetric displays using femtosecond lasers, as demonstrated in subsequent research [S2].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
