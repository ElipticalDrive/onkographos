# Computational Phase Modulation (CGH Generation)

Also called: ORA method, Optimization with Random Phase Algorithm

## Overview
Computational Phase Modulation, specifically via the Optimization with Random Phase Algorithm (ORA), is an iterative method for generating phase-only Computer-Generated Holograms (CGHs) that produce a target intensity distribution in a reconstruction plane. It solves the problem of encoding complex optical fields into phase-only Liquid Crystal Spatial Light Modulators (LC-SLMs) by optimizing the phase pattern to minimize the error between the reconstructed intensity and a desired spot array. Unlike direct phase quantization, which discards amplitude information and introduces noise, ORA iteratively adjusts phase variations to maximize the sum of light intensities at target pixels [S1]. It is distinct from [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) because it operates on a global optimization of the hologram plane to control focal spots rather than summing point-source contributions directly, and from [Inverse Design Algorithm for Voxel-based CGVHs](8543-inverse-design-algorithm-for-voxel-based-cgvhs.md) by focusing on spot-array uniformity rather than voxel-based volumetric density directly.

## How it works
**Inputs:**
*   Target intensity distribution $I_r^{(d)}$ on the reconstruction plane (e.g., a spot array).
*   Wavelength $\lambda$ and propagation distance (implicit in the Fourier transform relationship for far-field or lens-based systems).
*   SLM pixel grid dimensions.

**Outputs:**
*   Phase-only hologram pattern $\phi_h(x, y)$ to be displayed on the LC-SLM.

**Data Structures:**
*   `U_h`: Complex field at the hologram plane, shape $(H, W)$. Since the SLM is phase-only and illumination is a uniform plane wave, the amplitude $a_h$ is constant (typically 1.0).
*   `phi_h`: Phase array at the hologram plane, shape $(H, W)$, values in $[0, 2\pi)$.
*   `U_r`: Complex field at the reconstruction plane, shape $(H_r, W_r)$.
*   `I_r`: Intensity array at the reconstruction plane, shape $(H_r, W_r)$.
*   `omega_r`: Weight/phase array at the reconstruction plane, shape $(H_r, W_r)$, used to control intensity distribution.
*   `I_r_d`: Target intensity array, shape $(H_r, W_r)$.

**Parameters:**
*   $\alpha$: Constant for intensity update (source does not specify value, typically a small positive constant or 1.0 for direct ratio).
*   Max Iterations: Not specified in [S1], but [S2] uses 50 iterations for Gerchberg-Saxton. ORA typically requires convergence checking.
*   $k = 2\pi/\lambda$: Wavenumber.
*   $f$: Focal length (if axial Z-control is applied via a Fresnel lens).

**Steps:**

1.  **Initialization:**
    Initialize the hologram phase $\phi_h^{(0)}(x, y)$ with random values uniformly distributed in $[0, 2\pi)$ or zeros. Set reconstruction weights $\omega_r^{(0)}$ to 1.0 or based on target intensity. Set iteration counter $i = 0$.

2.  **Forward Propagation (Hologram to Reconstruction):**
    Compute the complex amplitude at the hologram plane:
    $$ U_h^{(i)}(x, y) = a_h \exp\left[i \phi_h^{(i)}(x, y)\right] $$
    where $a_h$ is the constant amplitude of the incident plane wave.
    Propagate to the reconstruction plane using the Fourier Transform (assuming far-field or lens-based Fourier plane):
    $$ U_r^{(i)}(\nu_x, \nu_y) = \mathcal{F} \left\{ U_h^{(i)}(x, y) \right\} $$
    Compute the current intensity:
    $$ I_r^{(i)}(\nu_x, \nu_y) = \left| U_r^{(i)}(\nu_x, \nu_y) \right|^2 $$

3.  **Update Reconstruction Weights (Intensity Constraint):**
    Update the phase/weight $\omega_r$ at each pixel $r$ in the reconstruction plane to steer energy toward the target intensity $I_r^{(d)}$:
    $$ \omega_r^{(i)} = \omega_r^{(i-1)} \left( \frac{I_r^{(d)}(r)}{I_r^{(i)}(r) + \epsilon} \right)^\alpha $$
    where $\epsilon$ is a small constant to prevent division by zero, and $\alpha$ is a constant (e.g., 1.0). [S1] defines this update but does not specify $\alpha$ or $\epsilon$. [G] $\epsilon$ should be set to a small value like $10^{-6}$ to ensure numerical stability.

4.  **Calculate Phase Variation ($\Delta \phi_h$):**
    For each pixel $h$ in the hologram plane, calculate the phase variation $\Delta \phi_h^{(i)}$ to maximize the sum of light intensities at the reconstruction plane pixels $r$.
    First, compute the contribution from pixel $h$ to all reconstruction pixels $r$. Let $u_{hr}^{(i)}$ be the complex amplitude contribution from hologram pixel $h$ to reconstruction pixel $r$. In the Fourier domain, this is related to the kernel of the transform. Specifically, the phase $\phi_{hr}$ is the phase of the propagation kernel from $h$ to $r$.
    Calculate sums $S_1$ and $S_2$:
    $$ S_1 = \sum_{r} \omega_r^{(i)} a_h \cos\left( \phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)} \right) $$
    $$ S_2 = \sum_{r} \omega_r^{(i)} a_h \sin\left( \phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)} \right) $$
    where $\phi_r^{(i)} = \arg(U_r^{(i)}(r))$ is the current phase at reconstruction pixel $r$, and $\phi_{hr}$ is the phase of the propagation kernel from $h$ to $r$.
    Compute the phase update:
    $$ \Delta \phi_h^{(i)} = \tan^{-1}\left( \frac{S_2}{S_1} \right) $$
    [G] Use `atan2(S2, S1)` to handle the quadrant correctly.

5.  **Update Hologram Phase:**
    $$ \phi_h^{(i+1)}(x, y) = \phi_h^{(i)}(x, y) + \Delta \phi_h^{(i)}(x, y) $$
    Normalize $\phi_h^{(i+1)}$ to $[0, 2\pi)$ if necessary.

6.  **Convergence Check:**
    Check if the difference between $I_r^{(i)}$ and $I_r^{(d)}$ is below a threshold or if max iterations are reached. If not, increment $i$ and return to Step 2.

**Lateral (XY) Focusing Control:**
To control the position of focusing spots in the lateral plane, the CGH is designed as a superposition of complex amplitudes of blazed gratings with various azimuth angles. If $N$ multiple focusing spots are required, the CGH includes $N$ blazed gratings. This is implicitly handled by the target intensity distribution $I_r^{(d)}$ having peaks at specific coordinates $(\nu_x, \nu_y)$ corresponding to the desired spot positions.

**Axial (Z) Focusing Control:**
To control the axial position, a phase Fresnel lens pattern is added to the optimized phase $\phi_h$:
$$ \phi_p(x, y) = \frac{k(x^2 + y^2)}{2f} $$
where $f$ is the focal length. The final hologram phase becomes:
$$ \phi_{final}(x, y) = \phi_h(x, y) + \phi_p(x, y) $$
The spatial resolution of the SLM determines the minimum focal length $f$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_cgh_ora(target_intensity, wavelength, slm_shape, num_iterations=100, alpha=1.0, epsilon=1e-6):
    """
    Computes a phase-only CGH using the Optimization with Random Phase Algorithm (ORA).
    
    Args:
        target_intensity: 2D numpy array, desired intensity distribution I_r^(d).
        wavelength: float, wavelength lambda.
        slm_shape: tuple (H, W), dimensions of the SLM.
        num_iterations: int, number of ORA iterations.
        alpha: float, constant for intensity update.
        epsilon: float, small constant for numerical stability.
        
    Returns:
        phi_h: 2D numpy array, optimized phase pattern.
    """
    H, W = slm_shape
    k = 2 * np.pi / wavelength
    
    # Initialize phase randomly in [0, 2*pi)
    phi_h = np.random.uniform(0, 2 * np.pi, (H, W))
    
    # Initialize weights in reconstruction plane
    # Assuming reconstruction plane has same shape as SLM for FFT-based propagation
    omega_r = np.ones((H, W))
    
    # Precompute propagation kernel phases if using FFT
    # The Fourier Transform handles the propagation. 
    # We need the phase of the kernel for the S1/S2 calculation.
    # In FFT, the kernel is exp(-i 2pi (x*nu_x + y*nu_y)).
    # However, ORA derivation in [S1] uses u_hr which is the contribution.
    # For FFT-based propagation, U_r = FFT(U_h).
    # The term inside the sum for S1/S2 involves phi_r - phi_hr - phi_h.
    # phi_hr is the phase of the propagation from h to r.
    # In the context of the Fourier Transform, the kernel is global.
    # Let's implement the sums directly using the kernel phases.
    
    # Create coordinate grids
    x, y = np.meshgrid(np.arange(W), np.arange(H))
    nu_x, nu_y = np.meshgrid(np.arange(W), np.arange(H)) # Spatial frequencies in cycles per frame
    
    # Propagation kernel phase: phi_hr = -2*pi*(x*nu_x + y*nu_y)
    # Note: FFT convention might vary. Standard numpy FFT uses exp(-2pi i (kx x/N + ky y/M))
    # We assume standard FFT.
    kernel_phase = -2 * np.pi * (x * nu_x / W + y * nu_y / H)
    
    for i in range(num_iterations):
        # Step 2: Forward Propagation
        # U_h = a_h * exp(i * phi_h). a_h is constant, assume 1.
        U_h = np.exp(1j * phi_h)
        
        # FFT to get U_r
        U_r = np.fft.fft2(U_h)
        
        # Current intensity
        I_r = np.abs(U_r)**2
        
        # Step 3: Update Reconstruction Weights
        # omega_r = omega_r * (I_r_d / I_r)^alpha
        # Avoid division by zero
        ratio = target_intensity / (I_r + epsilon)
        omega_r = omega_r * (ratio ** alpha)
        
        # Current phase at reconstruction plane
        phi_r = np.angle(U_r)
        
        # Step 4: Calculate Phase Variation
        # S1 = sum_r omega_r * a_h * cos(phi_r - phi_hr - phi_h)
        # S2 = sum_r omega_r * a_h * sin(phi_r - phi_hr - phi_h)
        # Note: phi_h is local to pixel h, phi_r and phi_hr are local to r and (h,r).
        # The sum is over r.
        # We need to compute this for each h.
        # This is a matrix operation.
        # Let's compute the term inside the sum for all h and r.
        # term = phi_r[r] - kernel_phase[h, r] - phi_h[h]
        # But kernel_phase depends on h and r.
        # kernel_phase[h, r] = -2*pi*(x[h]*nu_x[r]/W + y[h]*nu_y[r]/H)
        
        # Reshape for broadcasting
        # phi_h: (H, W) -> (H, W, 1)
        # phi_r: (H, W) -> (1, H, W)
        # kernel_phase: (H, W) -> (H, W, 1) ? No, kernel_phase is defined for each (h,r) pair.
        # Actually, in FFT, the relationship is global.
        # The derivation in [S1] assumes a general propagation.
        # For FFT, u_hr is the element of the DFT matrix.
        # u_hr = exp(-i 2pi (x_h nu_r_x / W + y_h nu_r_y / H))
        # So phi_hr = -2pi (x_h nu_r_x / W + y_h nu_r_y / H)
        
        # Let's compute the phase difference for all h, r
        # x_h, y_h are coordinates of h
        # nu_r_x, nu_r_y are coordinates of r
        
        # Create coordinate arrays for h and r
        # x_h, y_h: (H, W, 1)
        # nu_r_x, nu_r_y: (1, H, W)
        
        x_h, y_h = np.meshgrid(np.arange(W), np.arange(H), indexing='xy')
        nu_r_x, nu_r_y = np.meshgrid(np.arange(W), np.arange(H), indexing='xy')
        
        # Expand dimensions for broadcasting
        x_h = x_h[:, :, np.newaxis] # (H, W, 1)
        y_h = y_h[:, :, np.newaxis] # (H, W, 1)
        nu_r_x = nu_r_x[np.newaxis, :, :] # (1, H, W)
        nu_r_y = nu_r_y[np.newaxis, :, :] # (1, H, W)
        
        # Kernel phase for each pair (h, r)
        # phi_hr[h, r]
        phi_hr = -2 * np.pi * (x_h * nu_r_x / W + y_h * nu_r_y / H)
        
        # phi_r[r] -> (1, H, W)
        phi_r_exp = phi_r[np.newaxis, :, :]
        
        # phi_h[h] -> (H, W, 1)
        phi_h_exp = phi_h[:, :, np.newaxis]
        
        # Phase difference: phi_r - phi_hr - phi_h
        phase_diff = phi_r_exp - phi_hr - phi_h_exp
        
        # omega_r[r] -> (1, H, W)
        omega_r_exp = omega_r[np.newaxis, :, :]
        
        # a_h = 1
        # S1[h] = sum_r omega_r[r] * cos(phase_diff[h, r])
        # S2[h] = sum_r omega_r[r] * sin(phase_diff[h, r])
        
        S1 = np.sum(omega_r_exp * np.cos(phase_diff), axis=(1, 2))
        S2 = np.sum(omega_r_exp * np.sin(phase_diff), axis=(1, 2))
        
        # Step 4: Delta phi
        # Delta phi_h = atan2(S2, S1)
        delta_phi_h = np.arctan2(S2, S1)
        
        # Step 5: Update phi_h
        phi_h = phi_h + delta_phi_h
        
    return phi_h
```

## Subtopics

### CGH-based parallel focal point generation
Generates multiple focal points in parallel using a phase-only CGH displayed on an LCOS-SLM to increase voxel generation density per frame. It uses the Gerchberg-Saxton method for 50 Fourier iterative calculations to optimize the CGH. A lens with a focal length of 200 mm is used to focus light into separated focal points. The number of voxels generated per frame $N$ is calculated as $N = P n / f_r$, where $P$ is the number of parallel focal points, $n$ is the number of voxels generated in 1 second (dependent on laser repetition frequency), and $f_r$ is the refresh rate. A pitfall is that the actual number of parallelized voxels $P$ achieved may be significantly less than estimated due to irradiated pulse energy reduction to prevent varifocal lens damage. For example, with $E_{th} = 0.004$ mJ and $D=50\%$, estimated $P=708$ and $N=35,400$ at $f_r=20$ Hz, but actual $P=4$ was achieved due to lens damage constraints. This is handled by using a varifocal lens with higher light resistance or combining with dynamic optimization for CGH diffraction efficiency [S2].

## Operating characteristics
*   **Time Complexity:** The ORA method involves iterative FFTs and summations over all pixels. Each iteration is dominated by the FFT, which is $O(N^2 \log N)$ for an $N \times N$ grid. The summation for $S_1$ and $S_2$ is $O(N^4)$ if done naively, but can be optimized. The number of iterations is not specified in [S1] but [S2] uses 50.
*   **Memory Complexity:** Requires storing the hologram phase, reconstruction field, weights, and kernel phases. $O(N^2)$ for the fields and $O(N^4)$ for the kernel phase matrix if stored explicitly. [G] The kernel phase matrix can be computed on the fly or stored as $O(N^2)$ if using FFT properties, but the explicit sum in ORA suggests $O(N^4)$ memory if not optimized.
*   **Measured Performance:** [S2] estimates 35,400 voxels per frame at 20 Hz refresh rate with 4 parallel focal points.
*   **Strengths:** Allows high-quality spot array generation with uniform intensity. Can control lateral and axial focusing positions.
*   **Limitations:** Computationally intensive. Sensitive to pulse energy limits in physical implementations. Actual parallelization may be lower than theoretical estimates due to hardware constraints.
*   **When to use:** When high-quality, uniform intensity spot arrays are required for volumetric displays using phase-only SLMs.
*   **When not to use:** When real-time performance is critical and computational resources are limited, or when the hardware cannot support the required pulse energy for parallel focal points.

## Background
The ORA method was introduced by Hayasaki et al. in 2005 [S1]. It was developed to enable spatial phase control of light for laser processing and volumetric graphics rendering. The method uses a Liquid Crystal Spatial Light Modulator (LC-SLM) as an optical phased array, where molecule directions are controlled by electrodes to modulate the phase of a reflected laser beam. It builds on the principle that the complex amplitude of the reconstruction is the Fourier transform of the CGH pattern. The method was later applied to volumetric displays using femtosecond lasers to generate voxels in a drawing space [S2].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
