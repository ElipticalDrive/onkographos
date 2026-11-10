# Computational Phase Modulation (CGH Generation)
Also called: ORA method, Optimization with Random Phase Algorithm

## Overview
Computational Phase Modulation generates a phase-only computer-generated hologram (CGH) $\phi_h(x,y)$ displayed on a Liquid Crystal Spatial Light Modulator (LCSLM) to reconstruct a target 3D intensity distribution $I_r$ via diffraction. It solves the problem of converting a desired volumetric light field into a modifiable phase map for phase-only devices. It is distinguished from direct phase extraction methods by using an iterative optimization algorithm (ORA) to maximize the fidelity of the reconstructed intensity pattern, specifically handling the loss of amplitude information inherent in phase-only modulation [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Target intensity distribution $I_r^{(d)}(\nu_x, \nu_y)$ on the reconstruction plane; wavelength $\lambda$; SLM pixel pitch $p$; number of SLM pixels $N_{SLM}$.
- **Output:** Phase map $\phi_h(x,y)$ for the CGH.

**Data Structures:**
- **Hologram Plane:** Complex field $U_h(x,y) = a_h(x,y) \exp[i\phi_h(x,y)]$. Since the illumination is a uniform plane wave, $a_h(x,y)$ is constant across all pixels [S1].
- **Reconstruction Plane:** Complex field $U_r(\nu_x, \nu_y) = a_r(\nu_x, \nu_y) \exp[i\phi_r(\nu_x, \nu_y)]$ and intensity $I_r = |U_r|^2$ [S1].
- **Weights:** A weight map $\omega_r^{(i)}$ for each pixel $r$ in the reconstruction plane to control local intensity [S1].

**Parameters:**
- $\lambda$: Wavelength of the laser.
- $k = 2\pi/\lambda$: Wave number [S1].
- $f$: Focal length of the phase Fresnel lens (for axial control) [S1].
- $\alpha$: Constant for weight update (value not given in sources) [S1].
- $I_{max}$: Maximum allowable intensity to prevent hardware damage (implicit in pitfalls) [S2].

**Algorithm Steps:**
The method uses the Optimization with Random Phase Algorithm (ORA) [S1].

1.  **Initialization:**
    - Set iteration counter $i = 0$.
    - Initialize phase $\phi_h^{(0)}(x,y) = 0$ or random values.
    - Initialize weights $\omega_r^{(0)} = 1$ for all pixels $r$ [S1].
    - Define target intensity $I_r^{(d)}$ [S1].

2.  **Forward Propagation (Calculate Current Field):**
    - Compute the complex amplitude at the reconstruction plane $U_r^{(i)}$ from the current hologram phase $\phi_h^{(i)}$ using the Fourier Transform:
      $$ U_r^{(i)}(\nu_x, \nu_y) = \mathcal{F} \left\{ a_h \exp[i\phi_h^{(i)}(x,y)] \right\} $$
    - Calculate current intensity $I_r^{(i)} = |U_r^{(i)}|^2$ [S1].

3.  **Update Weights:**
    - Update the weight $\omega_r^{(i)}$ for each pixel $r$ based on the ratio of desired to current intensity:
      $$ \omega_r^{(i)} = \omega_r^{(i-1)} \left( \frac{I_r^{(d)}}{I_r^{(i)}} \right)^\alpha $$
    - Note: If $I_r^{(i)} = 0$, handle division by zero (e.g., keep previous weight) [G].

4.  **Calculate Phase Variation:**
    - For each pixel $h$ on the hologram plane, calculate the phase variation $\Delta\phi_h^{(i)}$ to maximize the sum of light intensities at the reconstruction plane:
      $$ S_1 = \sum_r \omega_r^{(i)} a_h \cos(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
      $$ S_2 = \sum_r \omega_r^{(i)} a_h \sin(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}) $$
      $$ \Delta\phi_h^{(i)} = \tan^{-1}\left(\frac{S_2}{S_1}\right) $$
    - Here, $\phi_{hr}$ is the phase contribution from hologram pixel $h$ to reconstruction pixel $r$, derived from the propagation kernel [S1].
    - $\phi_r^{(i)}$ is the phase of the current reconstruction field $U_r^{(i)}$ [S1].

5.  **Update Hologram Phase:**
    - Update the phase map:
      $$ \phi_h^{(i)} = \phi_h^{(i-1)} + \Delta\phi_h^{(i)} $$
    - Wrap $\phi_h^{(i)}$ to $[0, 2\pi)$ if the SLM requires it [G].

6.  **Convergence Check:**
    - Check if $I_r^{(i)}$ is sufficiently close to $I_r^{(d)}$ (e.g., mean squared error below threshold).
    - If not converged, increment $i$ and return to Step 2.

**Spatial Control Modifiers:**
- **Lateral (XY) Control:** To focus at multiple lateral positions, the CGH is designed as a superposition of blazed gratings with various azimuth angles. If $N$ spots are required, $N$ blazed gratings are superimposed in the phase calculation [S1].
- **Axial (Z) Control:** To control depth, add a phase Fresnel lens pattern to the hologram phase:
  $$ \phi_p(x, y) = \frac{k(x^2 + y^2)}{2f} $$
  The final phase becomes $\phi_h^{final} = \phi_h + \phi_p$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_cgh_ora(target_intensity, wavelength, slm_size, num_iterations, alpha=1.0):
    """
    Generates a phase-only CGH using the ORA method.
    
    Args:
        target_intensity: 2D array, desired intensity distribution I_r^(d).
        wavelength: float, lambda.
        slm_size: tuple (H, W), dimensions of the SLM.
        num_iterations: int, number of ORA iterations.
        alpha: float, weight update exponent.
        
    Returns:
        phase_map: 2D array, the optimized phase phi_h.
    """
    H, W = slm_size
    # Initialize phase to zero
    phi_h = np.zeros((H, W))
    
    # Constant amplitude (uniform illumination)
    a_h = 1.0
    
    # Initialize weights
    weights = np.ones_like(target_intensity)
    
    # Precompute propagation phases if possible, or compute on fly
    # For simplicity, we assume a helper function for forward propagation
    # which includes the Fourier Transform and phase extraction.
    
    for i in range(1, num_iterations + 1):
        # 1. Forward Propagation
        # U_r = FFT( a_h * exp(i * phi_h) )
        hologram_field = a_h * np.exp(1j * phi_h)
        reconstruction_field = np.fft.fft2(hologram_field)
        
        # Current intensity and phase
        current_intensity = np.abs(reconstruction_field)**2
        reconstruction_phase = np.angle(reconstruction_field)
        
        # Handle zero intensity to avoid division by zero in weight update
        # [G] Guard against numerical instability
        safe_intensity = np.where(current_intensity > 1e-10, current_intensity, 1e-10)
        
        # 2. Update Weights
        # omega_r^(i) = omega_r^(i-1) * (I_r^(d) / I_r^(i))^alpha
        weights = weights * (target_intensity / safe_intensity)**alpha
        
        # 3. Calculate Phase Variation Delta_phi_h
        # S1 = sum_r [ omega_r * a_h * cos(phi_r - phi_hr - phi_h) ]
        # S2 = sum_r [ omega_r * a_h * sin(phi_r - phi_hr - phi_h) ]
        
        # phi_hr is the phase from pixel h to pixel r.
        # In the Fourier domain, the phase term phi_hr is implicitly handled
        # by the FFT kernel. However, the ORA formula explicitly sums over r.
        # We must reconstruct the contribution of each h to each r.
        # u_hr = exp(i * phi_hr) is the element of the FFT matrix.
        # Let M be the FFT matrix (size H*W x H*W).
        # M[r, h] = (1/N) * exp(-i * 2pi * (x_h * nu_r + y_h * nu_r))
        # phi_hr = angle(M[r, h])
        
        # To avoid constructing the full matrix, we can iterate or use vectorized ops.
        # Here we use a vectorized approach assuming standard FFT normalization.
        
        # Get coordinates
        y, x = np.ogrid[:H, :W]
        nu_y, nu_x = np.ogrid[:H, :W] # Frequencies corresponding to output
        
        # Phase kernel for FFT: exp(-i * 2pi * (x*nu_x + y*nu_y))
        # Note: numpy fft2 uses specific normalization. The phase term phi_hr
        # corresponds to the exponent in the Fourier integral.
        # phi_hr = -2 * pi * (x * nu_x + y * nu_y) (scaled by grid spacing)
        # We need the phase contribution from h to r.
        
        # Let's compute S1 and S2 directly using the definition.
        # S1 and S2 are sums over r for each h.
        
        # Initialize S1 and S2 arrays for each h
        S1 = np.zeros((H, W))
        S2 = np.zeros((H, W))
        
        # Reshape for broadcasting
        # weights: (H, W)
        # a_h: scalar
        # reconstruction_phase: (H, W)
        # phi_h: (H, W)
        
        # We need phi_hr. 
        # The Fourier transform is: U_r = sum_h U_h * exp(-i * 2pi * (x_h*u_r + y_h*v_r))
        # So phi_hr = -2pi * (x_h*u_r + y_h*v_r)
        
        # Create coordinate grids for the phase calculation
        # Assuming unit spacing for simplicity, scaling handled by FFT
        # We need the phase of the kernel element connecting h to r.
        
        # Efficient calculation:
        # The term inside cos/sin is: phi_r - phi_hr - phi_h
        # Let theta_hr = phi_hr.
        # We can compute the matrix of phases K[r, h] = angle(FFT_kernel[r, h])
        
        # Construct the FFT kernel phases
        # K[r, h] = -2 * pi * (x[h] * nu_x[r] + y[h] * nu_y[r])
        # Note: numpy fft2 output order and frequency indices must match.
        
        # Generate frequency indices
        fy, fx = np.fft.fftfreq(H), np.fft.fftfreq(W)
        FY, FX = np.meshgrid(fy, fx, indexing='ij') # Shape (H, W)
        
        # Coordinates of SLM pixels
        Y, X = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
        
        # Phase kernel phi_hr for all pairs (r, h)
        # This is a 4D tensor or computed via broadcasting.
        # phi_hr[r, h] = -2 * pi * (X[h] * FY[r] + Y[h] * FX[r])
        # To save memory, we can compute S1 and S2 by iterating or using einsum.
        
        # Vectorized approach:
        # Expand dimensions
        # weights: (H, W, 1, 1) -> broadcast over r, h? No, weights is (H,W) for r.
        # Let's reshape weights to (H, W, 1, 1) where last two dims are dummy for h?
        # Actually, weights is indexed by r. phi_h is indexed by h.
        
        # W_r: (H, W)
        # Phi_h: (H, W)
        # Phi_r: (H, W)
        # Phi_hr: (H, W, H, W) -> Too big.
        
        # Alternative: Iterate over r to accumulate into S1_h, S2_h
        # S1[h] = sum_r [ W_r * cos(Phi_r[r] - Phi_hr[r,h] - Phi_h[h]) ]
        
        # This is computationally expensive O(N^2). 
        # For implementation, one might use the property that 
        # sum_r W_r U_r exp(i phi_hr) relates to inverse FFT.
        # However, ORA explicitly uses the gradient-like update.
        
        # Let's implement the loop for clarity, assuming N is small enough or optimized.
        # [G] In practice, this step is optimized using FFT properties or GPU.
        
        for r_y in range(H):
            for r_x in range(W):
                w_r = weights[r_y, r_x]
                phi_r = reconstruction_phase[r_y, r_x]
                
                # Phase from h to r
                # phi_hr = -2 * pi * (X * FY[r_y, r_x] + Y * FX[r_y, r_x])
                # But X, Y are arrays. 
                # phi_hr[h] = -2 * pi * (X[h] * FY[r_y, r_x] + Y[h] * FX[r_y, r_x])
                
                # Vectorize over h
                phi_hr_h = -2 * np.pi * (X * FY[r_y, r_x] + Y * FX[r_y, r_x])
                
                # Term: phi_r - phi_hr - phi_h
                # phi_r is scalar for this r. phi_h is array over h.
                term = phi_r - phi_hr_h - phi_h
                
                S1 += w_r * np.cos(term)
                S2 += w_r * np.sin(term)
                
        # 4. Update Phase
        # Delta_phi_h = atan2(S2, S1)
        delta_phi_h = np.arctan2(S2, S1)
        
        # 5. Update Phi_h
        phi_h = phi_h + delta_phi_h
        
    return phi_h
```

## Subtopics

### CGH-based parallel focal point generation
Generates multiple focal points in parallel using a phase-only CGH displayed on an LCOS-SLM to increase voxel generation density per frame. It uses the Gerchberg–Saxton method for 50 Fourier iterative calculations to optimize the CGH, which controls light to form separated focal points in a drawing space at infinite distance via a lens and varifocal lens [S2]. The number of voxels generated per frame is estimated by $N = P n / f_r$, where $P$ is the number of parallel focal points, $n$ is the number of voxels generated in 1 second (dependent on laser repetition frequency), and $f_r$ is the refresh rate [S2]. A pitfall is that the actual number of parallelized voxels $P$ may be significantly less than estimated due to irradiated pulse energy reduction to prevent varifocal lens damage; this is handled by using a varifocal lens with higher light resistance or combining with dynamic optimization for CGH diffraction efficiency [S2].

## Operating characteristics
- **Time Complexity:** Iterative. Each iteration requires a forward Fourier Transform and a phase update calculation over all pixels. Convergence depends on the complexity of the target intensity distribution [S1].
- **Memory Complexity:** Requires storing the hologram phase map, reconstruction field, weights, and intermediate sums. $O(N_{SLM})$ for the map, $O(N_{SLM}^2)$ if the full propagation matrix is stored, though optimized implementations avoid this [S1].
- **Performance:** The ORA method allows design of CGH with high quality, achieving uniform intensity spot arrays [S1]. In parallel focal point generation, estimated $N = 35,400$ voxels per frame at $f_r = 20$ Hz was achieved with $P=4$ parallel points, though estimated $P$ was 708 [S2].
- **Strengths:** Handles phase-only devices effectively; allows precise lateral and axial control of focal spots [S1].
- **Limitations:** Iterative nature can be slow; sensitive to hardware limitations like varifocal lens damage thresholds [S2].

## Background
The method was introduced by Hayasaki et al. in 2005 for laser processing and volumetric graphics rendering using a Liquid Crystal SLM (LCSLM) [S1]. The LCSLM acts as an optical phased array where molecule directions are controlled by electrodes to modulate the phase of a reflected laser beam [S1]. The ORA algorithm was developed to optimize the phase variation to obtain a CGH reconstruction composed of a spot array with uniform intensity [S1]. It builds on the principle that spatial phase control enables focusing position control along both lateral (XY) and axial (Z) directions [S1].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
