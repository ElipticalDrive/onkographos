# Point-Sampled Spectrum Model

Also called: Doppler Rendering with Fixed Bands, Fixed Band Representation, Point-sampled Doppler Rendering

## Overview
The Point-Sampled Spectrum Model is a discrete spectral representation method for volume rendering that approximates spectral distribution functions by dividing the spectrum into $N$ uniform bands via point-sampling at uniform intervals [S1]. It solves the problem of integrating emission and absorption along viewing rays in participating media with spectral variation, specifically supporting the Doppler effect [S1]. Unlike the [Piecewise Linear Spectrum Model](4709-piecewise-linear-spectrum-model.md), which uses continuous trapezoidal bands to ensure energy conservation and accurate Doppler shift representation, this model suffers from information loss and accuracy issues due to its fixed-band discretization [S1].

## How it works
**Inputs:**
- Ray integration step size $\Delta s$.
- Number of spectral bands $N$.
- Spectral range $[\lambda_{min}, \lambda_{max}]$ (typically visible light, e.g., 390 nm to 790 nm) [S1].
- Material properties: density $\rho_m(s)$, temperature $T(s)$, and pre-computed emission/absorption spectra $emissionSpect[m][b]$ and $absorptionSpect[m][b]$ for each material $m$ [S1].
- Velocity field $\nu(s)$ along the ray [S1].

**Outputs:**
- Final spectral intensity vector $bands[N]$ representing the radiance at the end of the ray [S1].
- Converted RGB color via XYZ conversion [S1].

**Data Structures:**
- `bands`: Array of size $N$. Each element stores the accumulated spectral intensity for a specific band.
- `emissionSpect[m]`: Array of size $N$ storing the emission intensity of material $m$ for each band index.
- `absorptionSpect[m]`: Array of size $N$ storing the absorption coefficient of material $m$ for each band index.

**Parameters:**
- $N$: Number of uniform bands. Typical value: $N=40$ [S1].
- $\Delta \lambda$: Band width, calculated as $(\lambda_{max} - \lambda_{min}) / N$ [S1].
- $\lambda_{min}$: Minimum wavelength (e.g., 390 nm) [S1].

**Formulas:**
1.  **Band Wavelength:** The central wavelength for band $b$ is:
    $$ \lambda_b = (\lambda_{min} + b \cdot \Delta \lambda) \cdot 10^{-9} \text{ meters} $$
    (Note: The source algorithm uses $390 + b \cdot \Delta \lambda$ assuming nm input, then converts to meters for Planck's law) [S1].

2.  **Doppler Factor:**
    $$ D = 1 + \frac{\nu}{c} $$
    where $\nu$ is the velocity and $c$ is the speed of light [S1].

3.  **Doppler-Shifted Temperature:**
    $$ T' = T \cdot D $$
    The temperature is scaled by the Doppler factor to approximate the black-body spectrum shift [S1].

4.  **Doppler-Shifted Wavelength:**
    $$ \lambda' = \lambda \cdot (1 - \frac{\nu}{c}) = \frac{\lambda}{D} $$
    (Note: The source algorithm calculates $\lambda_{shifted} = \lambda \cdot (1 - DOPPLER\_FACTOR)$ where $DOPPLER\_FACTOR = \nu/c$) [S1].

5.  **New Band Index:**
    $$ b' = \frac{\lambda' - \lambda_{min}}{\Delta \lambda} $$
    This maps the shifted wavelength to the nearest band index [S1].

6.  **Intensity Update (Linearized):**
    The traditional volume rendering integral is expanded and linearized for each band:
    $$ I(s + \Delta s, \lambda) = \sum_{m=1}^M \rho_m(s) E_m(\lambda) + I(s, \lambda) \left( 1 - \Delta s \sum_{m=1}^M \rho_m(s) A_m(\lambda) \right) $$
    where $E_m(\lambda)$ is the emitted power and $A_m(\lambda)$ is the absorption coefficient [S1].

**Algorithm Steps:**
1.  Initialize `bands` array of size $N$ to zero.
2.  While the ray is inside the volume:
    a. Fetch velocity $\nu$. Calculate $DOPPLER\_FACTOR = \nu / c$.
    b. Fetch density $\rho$ and temperature $T$.
    c. Update temperature: $T \leftarrow T \cdot (1 + DOPPLER\_FACTOR)$.
    d. **Black-body Emission Accumulation:**
       For each band $b$ from $0$ to $N-1$:
       i. Calculate wavelength $\lambda = (390 + b \cdot \Delta \lambda) \cdot 10^{-9}$.
       ii. Calculate Planck's law emission $E = PLANCKLAW(\lambda, T)$.
       iii. Accumulate: $bands[b] \leftarrow bands[b] + \rho \cdot E$.
    e. **Material Emission/Absorption Accumulation:**
       For each material $m$:
       i. Fetch density $\rho_m$.
       ii. For each band $b$ from $0$ to $N-1$:
           1. Calculate base wavelength $\lambda = (390 + b \cdot \Delta \lambda) \cdot 10^{-9}$.
           2. Calculate shifted wavelength $\lambda_{shifted} = \lambda \cdot (1 - DOPPLER\_FACTOR)$.
           3. Calculate new band index $new\_b = (\lambda_{shifted} - 390) / \Delta \lambda$.
           4. Accumulate emission: $bands[new\_b] \leftarrow bands[new\_b] + \rho_m \cdot emissionSpect[m][b]$.
           5. Accumulate absorption: $bands[new\_b] \leftarrow bands[new\_b] - \rho_m \cdot absorptionSpect[m][b]$.
3.  Convert `bands` to XYZ color space.
4.  Convert XYZ to RGB.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_sampled_spectrum_rendering(ray, volume, N=40, lambda_min=390.0, lambda_max=790.0):
    """
    Renders a ray using the Point-Sampled Spectrum Model.
    
    Args:
        ray: Object containing ray origin, direction, and intersection data.
        volume: Volume data structure providing density, velocity, temperature, and spectra.
        N: Number of spectral bands.
        lambda_min: Minimum wavelength in nm.
        lambda_max: Maximum wavelength in nm.
        
    Returns:
        rgb: Final RGB color.
    """
    delta_lambda = (lambda_max - lambda_min) / N
    bands = [0.0] * N
    
    # Ray marching loop
    while ray.inside_volume():
        # Fetch properties
        nu = volume.fetch_velocity(ray.position)
        rho = volume.fetch_density(ray.position)
        T = volume.fetch_temperature(ray.position)
        
        # Doppler calculations
        doppler_factor = nu / 299792458.0 # c in m/s
        T_shifted = T * (1.0 + doppler_factor)
        
        # Accumulate Black-body Emission
        for b in range(N):
            # Wavelength in nm for index calculation, then convert to meters for Planck's law
            lambda_nm = lambda_min + b * delta_lambda
            lambda_m = lambda_nm * 1e-9
            
            # Planck's Law: B_lambda(T) = (2*h*c^2 / lambda^5) * 1/(exp(hc/(lambda*k_B*T)) - 1)
            # Constants: h=6.626e-34, c=299792458, k_B=1.381e-23
            emission = planck_law(lambda_m, T_shifted)
            
            bands[b] += rho * emission
            
        # Accumulate Material Emission and Absorption
        for m in volume.materials():
            rho_m = volume.fetch_material_density(ray.position, m)
            
            for b in range(N):
                lambda_nm = lambda_min + b * delta_lambda
                lambda_m = lambda_nm * 1e-9
                
                # Shift wavelength for Doppler effect
                # lambda_shifted = lambda * (1 - v/c)
                lambda_shifted_m = lambda_m * (1.0 - doppler_factor)
                lambda_shifted_nm = lambda_shifted_m * 1e9
                
                # Map shifted wavelength to new band index
                new_b = (lambda_shifted_nm - lambda_min) / delta_lambda
                
                # Clamp index to valid range [0, N-1] to handle out-of-bounds shifts
                # [G] Guard against negative or out-of-bounds indices which cause artifacts
                if 0 <= new_b < N:
                    new_b_int = int(round(new_b))
                    # Ensure integer index is within bounds
                    new_b_int = max(0, min(N - 1, new_b_int))
                    
                    # Accumulate emission from material spectrum
                    bands[new_b_int] += rho_m * m.emission_spectrum[b]
                    
                    # Accumulate absorption from material spectrum
                    bands[new_b_int] -= rho_m * m.absorption_spectrum[b]
                    
        ray.step()
        
    # Convert spectrum to color
    xyz = spectrum_to_xyz(bands)
    rgb = xyz_to_rgb(xyz)
    return rgb

def planck_law(wavelength_m, temperature_k):
    """
    Calculates spectral radiance using Planck's Law.
    """
    h = 6.62607015e-34  # Planck constant
    c = 299792458.0     # Speed of light
    k_B = 1.380649e-23  # Boltzmann constant
    
    exponent = (h * c) / (wavelength_m * k_B * temperature_k)
    if exponent < 1e-10: # Avoid division by zero or overflow in exp
        return 0.0
        
    term = math.exp(exponent) - 1.0
    if term == 0:
        return 0.0
        
    numerator = 2.0 * h * c * c
    denominator = (wavelength_m ** 5) * term
    
    return numerator / denominator
```

**Edge Cases and Pitfalls:**
- **Energy Conservation Loss:** The linearization of the transmittance term $e^{-l\rho\tau(\lambda)} \approx 1 - l\rho\tau(\lambda)$ is only valid for small thicknesses. With high extinctions or densities, this leads to amplification of incident light instead of attenuation unless the step size $\Delta s$ is sufficiently small [S1].
- **Accuracy Loss during Doppler Shift:** If a shifted wavelength $\lambda'$ falls into a different band than the original $\lambda$, the shift is over- or underestimated. In the next integration step, the wrong band is used for calculating the Doppler shift, leading to apparent emission or absorption lines [S1].
- **Discrete Doppler Influence:** The effective visual influence of the Doppler effect depends on $N$. The minimum velocity $v/c$ that has a visual influence is determined by $D = 1 + \Delta\lambda/(2\lambda)$, where $D = 1 + v/c$ [S1]. For example, with $N=40$, $\lambda \in [390, 790]$ nm, $\Delta\lambda=10$ nm, at $\lambda=390$ nm, $v/c \approx 0.0128$, and at $\lambda=790$ nm, $v/c \approx 0.0063$ [S1].
- **No Line Broadening:** The model cannot reproduce Doppler line broadening, as single spectral lines ($\delta$-peaks) are represented as rectangles of width $\Delta\lambda$ [S1].

## Subtopics

### [Piecewise Linear Spectrum Model](4709-piecewise-linear-spectrum-model.md)
A continuous spectral representation that replaces discrete binning with exact positions and intensities, ensuring energy conservation and accurate Doppler shift representation. It models spectral lines as trapezoidal bands with linearly varying intensity, allowing for precise handling of non-uniform spectral intervals, unlike the fixed bands of the Point-Sampled model.

## Operating characteristics
- **Time Complexity:** The complexity is mainly characterized by the number of spectral bands $N$. For each ray step, the algorithm iterates over $N$ bands for black-body emission and $N \times M$ bands for material emission/absorption, where $M$ is the number of materials [S1].
- **Memory Complexity:** Requires storage for $N$ bands per ray and pre-computed spectra of size $N$ per material [S1].
- **Strengths:** Simple to implement; additive nature of spectral radiance allows straightforward accumulation [S1].
- **Limitations:** Suffers from energy conservation loss, information loss during Doppler shifts, and accuracy issues due to fixed bands [S1]. Requires small step sizes for high densities to avoid amplification artifacts [S1].
- **When to use:** Suitable for low-extinction scenes or when computational simplicity is prioritized over spectral accuracy.
- **When not to use:** High-density volumes, scenes requiring accurate Doppler shift visualization, or when energy conservation is critical [S1].

## Background
The Point-Sampled Spectrum Model is described in the context of Doppler Volume Rendering for astrophysics simulations [S1]. It is presented as the baseline method before introducing the novel Piecewise Linear Spectrum Model [S1]. The method relies on the additive nature of spectral radiance and the linearization of the volume rendering integral [S1].

## Sources
[S1] Doppler Volume Rendering: A Dynamic, Piecewise Linear Spectral Representation for Visualizing Astrophysics Simulations, https://albertojaspe.net/publications/2023-EVIS-doppler_volume.pdf, fetched 2026-10-28
