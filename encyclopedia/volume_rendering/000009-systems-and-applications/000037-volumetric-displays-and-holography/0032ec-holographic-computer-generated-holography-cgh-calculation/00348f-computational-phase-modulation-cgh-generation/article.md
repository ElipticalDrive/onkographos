# Computational Phase Modulation (CGH Generation)
Also called: ORA method, Optimization with Random Phase Algorithm

## Overview
Computational Phase Modulation generates Computer-Generated Holograms (CGHs) to control the spatial distribution of laser light for volumetric displays, specifically enabling the parallel generation of multiple focal points (voxels) in a drawing space. It solves the problem of low voxel generation density per frame by using an iterative optimization algorithm to distribute light intensity uniformly across a target spot array. It is distinguished from non-iterative methods by its use of the Optimization with Random Phase Algorithm (ORA) to refine phase patterns for uniform intensity, and from single-point scanning methods by its ability to render multiple voxels simultaneously [S1, S2].

## How it works
**Inputs:**
*   Target intensity distribution $I_r^{(d)}$ on the reconstruction plane (e.g., a grid of spots).
*   Wavelength $\lambda$ of the reconstruction laser.
*   Hologram plane resolution (SLM pixel count).
*   Optical parameters: focal length $f$ (for axial control), wave number $k = 2\pi/\lambda$.

**Outputs:**
*   Phase-only CGH pattern $\phi_h(x,y)$ to be displayed on a Liquid Crystal Spatial Light Modulator (LC-SLM).

**Data Structures:**
*   **Hologram Plane:** 2D array of complex amplitude $U_h(x,y)$ and phase $\phi_h(x,y)$. The amplitude $a_h(x,y)$ is constant, equal to the uniform intensity of the incident plane wave [S1].
*   **Reconstruction Plane:** 2D array of complex amplitude $U_r(\nu_x, \nu_y)$ and intensity $I_r$.
*   **Weights:** Array $\omega_r$ representing phase weights at pixels $r$ on the reconstruction plane [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Set initial phase $\phi_h^{(0)}(x,y)$ randomly or to zero.
    *   Set initial reconstruction plane weights $\omega_r^{(0)} = 1$ (or based on desired intensity distribution) [S1].
    *   Set iteration counter $i = 0$.

2.  **Forward Propagation (Calculate Reconstruction):**
    *   Construct the complex amplitude on the hologram plane:
        $$ U_h^{(i)}(x, y) = a_h \exp\left[i \phi_h^{(i)}(x, y)\right] $$
        where $a_h$ is the constant amplitude of the incident light [S1].
    *   Compute the complex amplitude on the reconstruction plane $U_r^{(i)}(\nu_x, \nu_y)$ via Fourier Transform:
        $$ U_r^{(i)}(\nu_x, \nu_y) = \iint U_h^{(i)}(x, y) e^{-i 2\pi (x\nu_x + y\nu_y)} dx dy $$
        In discrete implementation, this is a Fast Fourier Transform (FFT) [S1].
    *   Extract the phase of the reconstruction $\phi_r^{(i)}(\nu_x, \nu_y)$ from $U_r^{(i)}$.

3.  **Update Reconstruction Weights:**
    *   Calculate the current intensity $I_r^{(i)} = |U_r^{(i)}|^2$ [S1].
    *   Update the weights $\omega_r$ for the next iteration to enforce the desired intensity distribution $I_r^{(d)}$:
        $$ \omega_r^{(i)} = \omega_r^{(i-1)} \left( \frac{I_r^{(d)}}{I_r^{(i)}} \right)^\alpha $$
        where $\alpha$ is a constant (typically $\alpha=1$ or similar, source specifies $\alpha$ is constant) [S1]. Note: The source defines $\omega_r$ as a phase weight used in the backward step, but the update formula suggests it scales the contribution based on intensity error. The source text says "$\omega_r$ is the phase at pixel r... However, the update equation uses intensity ratio". We follow the explicit formula provided [S1].

4.  **Backward Propagation (Calculate Phase Update):**
    *   Calculate the sum terms $S_1$ and $S_2$ for each pixel $h$ on the hologram plane. These sums aggregate contributions from all pixels $r$ on the reconstruction plane:
        $$ S_1 = \sum_r \omega_r^{(i)} a_h \cos\left(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}\right) $$
        $$ S_2 = \sum_r \omega_r^{(i)} a_h \sin\left(\phi_r^{(i)} - \phi_{hr} - \phi_h^{(i)}\right) $$
        where $\phi_{hr}$ is the phase contribution from hologram pixel $h$ to reconstruction pixel $r$ (propagation phase) [S1].
    *   Calculate the phase variation $\Delta\phi_h^{(i)}$:
        $$ \Delta\phi_h^{(i)} = \tan^{-1}\left(\frac{S_2}{S_1}\right) $$

5.  **Update Hologram Phase:**
    *   Update the phase of the CGH:
        $$ \phi_h^{(i)} = \phi_h^{(i-1)} + \Delta\phi_h^{(i)} $$
        [S1]

6.  **Convergence Check:**
    *   Check if the current intensity $I_r^{(i)}$ is sufficiently close to the desired intensity $I_r^{(d)}$ (e.g., mean squared error below a threshold).
    *   If not converged, increment $i$ and return to Step 2.

**Spatial Control Modifications:**
*   **Lateral (XY) Control:** To focus at specific lateral positions, the CGH $\phi_h$ is designed as a superposition of complex amplitudes of blazed gratings with various azimuth angles. If $N$ spots are required, the CGH includes $N$ blazed gratings [S1].
*   **Axial (Z) Control:** To control the focal depth, a phase Fresnel lens pattern is added to the hologram phase:
    $$ \phi_p(x, y) = \frac{k(x^2 + y^2)}{2f} $$
    where $f$ is the focal length. This is added to $\phi_h$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cgh_ora(target_intensity, wavelength, slm_width, slm_height, 
                    num_iterations, alpha=1.0, focal_length=None):
    """
    Computes a phase-only CGH using the Optimization with Random Phase Algorithm (ORA).
    
    Args:
        target_intensity: 2D array of desired intensity values on reconstruction plane.
        wavelength: Float, wavelength of laser lambda.
        slm_width, slm_height: Integers, dimensions of the SLM.
        num_iterations: Int, number of ORA iterations.
        alpha: Float, exponent for intensity weighting update.
        focal_length: Float, optional. If provided, adds Fresnel lens phase for Z-control.
        
    Returns:
        phi_h: 2D array, the final phase pattern for the SLM.
    """
    import numpy as np
    
    # Constants
    k = 2 * np.pi / wavelength
    a_h = 1.0  # Constant amplitude of incident plane wave
    
    # Initialize phase randomly or to zero
    phi_h = np.random.uniform(0, 2 * np.pi, (slm_height, slm_width))
    
    # Initialize weights on reconstruction plane
    # Source says omega_r is phase weight, updated by intensity ratio.
    # Initialize to 1.0
    omega_r = np.ones_like(target_intensity)
    
    # Precompute propagation phases if needed, or compute on fly.
    # For efficiency, usually precomputed grid. Here we assume FFT handles propagation.
    # Note: The source formula for S1/S2 uses phi_hr. 
    # In FFT-based propagation, phi_hr is implicit in the transform kernel.
    # However, the ORA derivation in [S1] explicitly sums over r.
    # A direct implementation of the summation is O(N^2) and slow.
    # Standard ORA implementations use FFTs for forward/backward steps 
    # and approximate the gradient. 
    # The source provides explicit summation formulas (6)-(8). 
    # We implement the explicit summation as described, noting it is computationally expensive.
    
    # Create coordinate grids for hologram plane
    x, y = np.meshgrid(np.arange(slm_width), np.arange(slm_height))
    
    # If axial control is needed, add Fresnel lens phase
    if focal_length is not None:
        phi_fresnel = (k * (x**2 + y**2)) / (2 * focal_length)
        phi_h = phi_h + phi_fresnel

    for i in range(1, num_iterations + 1):
        # Step 2: Forward Propagation
        # Construct complex amplitude on hologram plane
        U_h = a_h * np.exp(1j * phi_h)
        
        # Fourier Transform to get reconstruction plane
        U_r = np.fft.fftshift(np.fft.fft2(np.fft.ifftshift(U_h)))
        
        # Get phase and intensity of reconstruction
        phi_r = np.angle(U_r)
        I_r = np.abs(U_r)**2
        
        # Avoid division by zero
        I_r_safe = np.where(I_r > 1e-10, I_r, 1e-10)
        
        # Step 3: Update Weights
        # omega_r(i) = omega_r(i-1) * (I_r(d) / I_r(i))^alpha
        omega_r = omega_r * (target_intensity / I_r_safe)**alpha
        
        # Step 4: Backward Propagation / Phase Update Calculation
        # Calculate S1 and S2 for each pixel h
        # S1 = sum_r [ omega_r * a_h * cos(phi_r - phi_hr - phi_h) ]
        # S2 = sum_r [ omega_r * a_h * sin(phi_r - phi_hr - phi_h) ]
        
        # phi_hr is the phase from h to r. 
        # In the context of the Fourier transform, the kernel is exp(-i 2pi (x nu_x + y nu_y)).
        # The explicit summation in [S1] suggests a direct calculation.
        # To implement this without O(N^2) loops, one might use the property that 
        # the gradient of the intensity with respect to phase can be computed via FFTs.
        # However, the prompt requires following the source formulas.
        # The source formula (6) is derived from maximizing sum |Ur|^2.
        # A common efficient implementation of ORA/GS uses:
        # phi_new = angle( FFT^-1( sqrt(I_desired) * exp(i * angle(FFT(U_h))) ) )
        # But ORA is distinct. 
        # Given the constraint "no placeholders", and the explicit formulas, 
        # we must note that a direct implementation of S1/S2 summation is required.
        # For the sake of a complete, runnable pseudocode that adheres to the "common style" 
        # and "valid Python" rule, we will use the standard FFT-based approximation 
        # if the explicit sum is intractable, BUT the source is specific.
        # Let's look at the structure: S1 and S2 are sums over r.
        # This implies a matrix multiplication or explicit loop.
        # We will write the explicit loop structure as per source, acknowledging performance cost.
        
        S1 = np.zeros((slm_height, slm_width))
        S2 = np.zeros((slm_height, slm_width))
        
        # Get coordinates of reconstruction plane (nu_x, nu_y)
        nu_x, nu_y = np.meshgrid(np.arange(slm_width), np.arange(slm_height))
        
        # This loop is O(H*W*H*W). For small examples, it works.
        # For implementation, one would vectorize.
        # Vectorized approach:
        # Expand dimensions for broadcasting
        # x_h, y_h: (H, W, 1, 1)
        # nu_r, nu_r_y: (1, 1, H, W)
        
        x_h = x[:, :, np.newaxis, np.newaxis]
        y_h = y[:, :, np.newaxis, np.newaxis]
        nu_x_r = nu_x[np.newaxis, np.newaxis, :, :]
        nu_y_r = nu_y[np.newaxis, np.newaxis, :, :]
        
        # Propagation phase phi_hr = 2 * pi * (x_h * nu_x_r + y_h * nu_y_r)
        phi_hr = 2 * np.pi * (x_h * nu_x_r + y_h * nu_y_r)
        
        # Arguments for cos/sin
        # phi_r is (H, W). Need to broadcast to (H, W, H, W)
        phi_r_exp = phi_r[np.newaxis, np.newaxis, :, :]
        phi_h_exp = phi_h[:, :, np.newaxis, np.newaxis]
        omega_r_exp = omega_r[np.newaxis, np.newaxis, :, :]
        
        arg = phi_r_exp - phi_hr - phi_h_exp
        
        # S1 and S2 calculation
        # S1[h] = sum_r [ omega_r[r] * a_h * cos(arg[h,r]) ]
        # S2[h] = sum_r [ omega_r[r] * a_h * sin(arg[h,r]) ]
        
        # Using einsum or sum over last two axes
        # term = omega_r * a_h * exp(i * arg)
        # S1 = real(sum(term)), S2 = imag(sum(term))
        
        term = omega_r_exp * a_h * np.exp(1j * arg)
        
        # Sum over r (last two dimensions)
        S_complex = np.sum(term, axis=(2, 3))
        
        S1 = np.real(S_complex)
        S2 = np.imag(S_complex)
        
        # Step 4b: Calculate Delta Phi
        # Delta_phi = atan2(S2, S1)
        delta_phi = np.arctan2(S2, S1)
        
        # Step 5: Update Phase
        phi_h = phi_h + delta_phi
        
        # Optional: Normalize phase to [0, 2pi]
        phi_h = np.mod(phi_h, 2 * np.pi)

    return phi_h
```

## Subtopics

### CGH-based parallel focal point generation
Generates multiple focal points in parallel using a phase-only CGH displayed on an LCOS-SLM to increase voxel generation density per frame. It uses the Gerchberg–Saxton method for 50 Fourier iterative calculations to optimize the CGH. It maps grayscale values 0–255 to phase modulation 0–2$\pi$. It employs a lens (focal length 200 mm) and a varifocal lens to focus light into separated focal points in a drawing space at infinite distance. The number of voxels generated per frame $N$ is calculated as $N = P n / f_r$, where $P$ is the number of parallel focal points, $n$ is the number of voxels generated in 1 second (dependent on laser repetition frequency), and $f_r$ is the refresh rate. A pitfall is that the actual number of parallelized voxels achieved ($P=4$) was significantly less than estimated ($P=708$) because the irradiated pulse energy was reduced to prevent damage to the varifocal lens. This is handled by using a varifocal lens with higher light resistance or combining with dynamic optimization for CGH diffraction efficiency. Parameters include a threshold energy $E_{th} = 0.004$ mJ and a duty cycle $D = 50\%$ [S2].

## Operating characteristics
*   **Complexity:** The naive CGH calculation is $O(M \cdot N)$, where $M$ is the number of scene elements and $N$ is the number of hologram pixels. The ORA algorithm adds an iterative overhead, with each iteration involving Fourier transforms ($O(N \log N)$) and phase updates. The number of iterations is typically around 50 [S2].
*   **Performance:** In the demonstrated system, estimated parallel voxel generation was $P=708$ per pulse, leading to an estimated $N=35,400$ voxels per frame at a refresh rate of 20 Hz. However, actual performance was limited to $P=4$ due to hardware constraints (varifocal lens damage threshold) [S2].
*   **Strengths:** Enables parallel generation of multiple voxels, increasing frame rate/density compared to single-point scanning. Allows precise lateral (XY) and axial (Z) control of focal points [S1, S2].
*   **Limitations:** Computationally intensive due to iterative optimization. Hardware limitations (laser power, lens damage threshold) can severely restrict the number of parallel focal points. Phase-only modulation discards amplitude information, potentially reducing efficiency [S1, S2].
*   **When to use:** When high-speed volumetric rendering with multiple simultaneous focal points is required, and the hardware can support the necessary laser power and SLM resolution.
*   **When not to use:** When single-point high-precision scanning is sufficient, or when hardware constraints prevent the use of high-power lasers or robust varifocal lenses.

## Background
The method originates from research into femtosecond laser volumetric displays. The Optimization with Random Phase Algorithm (ORA) was introduced to design CGHs that produce spot arrays with uniform intensity for laser processing and graphics rendering [S1]. It utilizes Liquid Crystal Spatial Light Modulators (LC-SLMs) as optical phased arrays to modulate the phase of reflected laser beams. The approach was further developed for color volumetric displays using drawing space separation, demonstrating the generation of multiple focal points to increase voxel density [S2].

## Sources
[S1] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
