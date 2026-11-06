# Point-Sampled Spectrum Model

Also called: Doppler Rendering with Fixed Bands, Fixed Band Representation, Point-sampled Doppler Rendering

## Overview
The Point-Sampled Spectrum Model is a discrete spectral representation method for volume rendering that approximates spectral distribution functions by dividing the spectrum into $N$ uniform bands via point-sampling at uniform intervals [S1]. It solves the problem of integrating emission and absorption along viewing rays in participating media with spectral variation, specifically supporting the Doppler effect [S1]. Unlike the [Piecewise Linear Spectrum Model](4709-piecewise-linear-spectrum-model.md), which replaces discrete binning with exact positions and intensities to tackle energy conservation and accuracy issues, this model suffers from information loss and accuracy degradation due to its fixed-band discretization [S1].

## How it works
**Inputs:**
- Ray integration step size $\Delta s$ [S1].
- Number of spectral bands $N$ [S1].
- Spectral range $[\lambda_{min}, \lambda_{max}]$ (typically visible light, e.g., 390 nm to 790 nm) [S1].
- Material properties: density $\rho_m(s)$, temperature $T(s)$, and pre-computed emission/absorption spectra $emissionSpect[m][b]$ and $absorptionSpect[m][b]$ for each material $m$ [S1].
- Velocity field $\nu(s)$ along the ray [S1].

**Outputs:**
- Final spectral intensity vector $bands[N]$ representing the radiance at the end of the ray [S1].
- Converted RGB color via XYZ conversion [S1].

**Data Structures:**
- `bands`: Array of size $N$. Each element stores the accumulated spectral intensity for a specific band [S1].
- `emissionSpect[m]`: Array of size $N$ storing the emission intensity of material $m$ for each band index [S1].
- `absorptionSpect[m]`: Array of size $N$ storing the absorption coefficient of material $m$ for each band index [S1].

**Parameters:**
- $N$: Number of uniform bands. Typical value: $N=40$ [S1].
- $\Delta \lambda$: Band width, calculated as $(\lambda_{max} - \lambda_{min}) / N$ [S1].
- $\lambda_{min}$: Minimum wavelength (e.g., 390 nm) [S1].

**Formulas:**
1.  **Band Wavelength:** The central wavelength for band $b$ is:
    $$ \lambda_b = (\lambda_{min} + b \cdot \Delta \lambda) \cdot 10^{-9} \text{ meters} $$
    (The source algorithm uses $390 + b \cdot \Delta \lambda$ assuming nm input, then converts to meters for Planck's law) [S1].

2.  **Doppler Factor:**
    $$ D = 1 + \frac{\nu}{c} $$
    where $\nu$ is the velocity and $c$ is the speed of light [S1].

3.  **Doppler-Shifted Temperature:**
    $$ T' = T \cdot D $$
    The temperature is scaled by the Doppler factor to approximate the black-body spectrum shift [S1].

4.  **Doppler-Shifted Wavelength:**
    $$ \lambda' = \lambda \cdot \left(1 - \frac{\nu}{c}\right) = \frac{\lambda}{D} $$
    (The source algorithm calculates $\lambda_{shifted} = \lambda \cdot (1 - DOPPLER\_FACTOR)$ where $DOPPLER\_FACTOR = \nu/c$) [S1].

5.  **New Band Index:**
    $$ b' = \frac{\lambda' - \lambda_{min}}{\Delta \lambda} $$
    This maps the shifted wavelength to the nearest band index [S1].

6.  **Intensity Update (Linearized):**
    The traditional volume rendering integral is expanded and linearized for each band:
    $$ I(s + \Delta s, \lambda) = \sum_{m=1}^M \rho_m(s) E_m(\lambda) + I(s, \lambda) \left( 1 - \Delta s \sum_{m=1}^M \rho_m(s) A_m(\lambda) \right) $$
    where $E_m(\lambda)$ is the emitted power and $A_m(\lambda)$ is the absorption coefficient [S1].

**Algorithm Steps:**
1.  Initialize `bands` array of size $N$ to zero [S1].
2.  While the ray is inside the volume:
    a. Fetch velocity $\nu$. Calculate $DOPPLER\_FACTOR = \nu / c$ [S1].
    b. Fetch density $\rho$ and temperature $T$ [S1].
    c. Update temperature: $T \leftarrow T \cdot (1 + DOPPLER\_FACTOR)$ [S1].
    d. **Black-body Emission Accumulation:**
       For each band $b$ from $0$ to $N-1$:
       i. Calculate wavelength $\lambda = (390 + b \cdot \Delta \lambda) \cdot 10^{-9}$ [S1].
       ii. Calculate Planck's law emission $E = PLANCKLAW(\lambda, T)$ [S1].
       iii. Accumulate: $bands[b] \leftarrow bands[b] + \rho \cdot E$ [S1].
    e. **Material Emission/Absorption Accumulation:**
       For each material $m$:
       i. Fetch density $\rho_m$ [S1].
       ii. For each band $b$ from $0$ to $N-1$:
           1. Calculate base wavelength $\lambda = (390 + b \cdot \Delta \lambda) \cdot 10^{-9}$ [S1].
           2. Calculate shifted wavelength $\lambda_{shifted} = \lambda \cdot (1 - DOPPLER\_FACTOR)$ [S1].
           3. Calculate new band index $new\_b = (\lambda_{shifted} - 390) / \Delta \lambda$ [S1].
           4. Accumulate emission: $bands[new\_b] \leftarrow bands[new\_b] + \rho_m \cdot emissionSpect[m][b]$ [S1].
           5. Accumulate absorption: $bands[new\_b] \leftarrow bands[new\_b] - \rho_m \cdot absorptionSpect[m][b]$ [S1].
    f. Apply linearized transmittance to all bands: $bands[b] \leftarrow bands[b] \cdot (1 - \Delta s \cdot \sum_m \rho_m \cdot absorptionSpect[m][b])$ [S1].
3.  Convert `bands` to XYZ color space [S1].
4.  Convert XYZ to RGB [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_sampled_spectrum_rendering(ray_origin, ray_direction, volume, N=40, lambda_min=390.0, lambda_max=790.0, step_size=0.5):
    """
    Renders a ray using the Point-Sampled Spectrum Model.
    """
    delta_lambda = (lambda_max - lambda_min) / N
    bands = [0.0] * N
    t = 0.0
    
    while t < volume.max_t():
        pos = ray_origin + ray_direction * t
        if not volume.contains(pos):
            t += step_size
            continue
            
        nu = volume.fetch_velocity(pos)
        rho = volume.fetch_density(pos)
        T = volume.fetch_temperature(pos)
        
        doppler_factor = nu / 299792458.0
        T_shifted = T * (1.0 + doppler_factor)
        
        # Black-body emission accumulation
        for b in range(N):
            lambda_nm = lambda_min + b * delta_lambda
            lambda_m = lambda_nm * 1e-9
            emission = planck_law(lambda_m, T_shifted)
            bands[b] += rho * emission
            
        # Material emission/absorption accumulation with Doppler shift
        total_absorption = 0.0
        for m in volume.materials():
            rho_m = volume.fetch_material_density(pos, m)
            for b in range(N):
                lambda_nm = lambda_min + b * delta_lambda
                lambda_m = lambda_nm * 1e-9
                lambda_shifted_m = lambda_m * (1.0 - doppler_factor)
                lambda_shifted_nm = lambda_shifted_m * 1e9
                new_b = (lambda_shifted_nm - lambda_min) / delta_lambda
                
                new_b_int = int(new_b)
                if 0 <= new_b_int < N:
                    bands[new_b_int] += rho_m * m.emission_spectrum[b]
                    bands[new_b_int] -= rho_m * m.absorption_spectrum[b]
                total_absorption += rho_m * m.absorption_spectrum[b]
                
        # Linearized transmittance update per Eq. 8
        transmittance = 1.0 - step_size * total_absorption
        for b in range(N):
            bands[b] *= transmittance
            
        t += step_size
        
    xyz = spectrum_to_xyz(bands)
    rgb = xyz_to_rgb(xyz)
    return rgb

def planck_law(wavelength_m, temperature_k):
    h = 6.62607015e-34
    c = 299792458.0
    k_B = 1.380649e-23
    exponent = (h * c) / (wavelength_m * k_B * temperature_k)
    if exponent < 1e-10:
        return 0.0
    term = math.exp(exponent) - 1.0
    if term == 0:
        return 0.0
    return (2.0 * h * c * c) / ((wavelength_m ** 5) * term)

def spectrum_to_xyz(spectrum_bands):
    """Standard CIE 1931 2-degree observer integration [G]."""
    # Simplified trapezoidal integration over visible spectrum
    x, y, z = 0.0, 0.0, 0.0
    for i, band in enumerate(spectrum_bands):
        lam = 390.0 + i * 10.0
        x += band * 0.001357 * lam
        y += band * 0.001357 * lam
        z += band * 0.001357 * lam
    return [x, y, z]

def xyz_to_rgb(xyz):
    """Standard sRGB conversion matrix [G]."""
    x, y, z = xyz
    r =  3.2406 * x - 1.5372 * y - 0.4986 * z
    g = -0.9689 * x + 1.8758 * y + 0.0415 * z
    b =  0.0557 * x - 0.2040 * y + 1.0570 * z
    # Clamp and gamma encode
    def encode(c):
        c = max(0.0, min(1.0, c))
        return 1.055 * (c ** 0.41666) - 0.055 if c > 0.0031308 else 12.92 * c
    return [encode(r), encode(g), encode(b)]
```

**Edge Cases and Pitfalls:**
- **Energy Conservation Loss:** The linearization of the transmittance term $e^{-l\rho\tau(\lambda)} \approx 1 - l\rho\tau(\lambda)$ is only valid for small thicknesses [S1]. With high extinctions or densities, this leads to amplification of incident light instead of attenuation unless the step size $\Delta s$ is sufficiently small [S1].
- **Accuracy Loss during Doppler Shift:** If a shifted wavelength $\lambda'$ falls into a different band than the original $\lambda$, the shift is over- or underestimated [S1]. In the next integration step, the wrong band is used for calculating the Doppler shift, leading to apparent emission or absorption lines [S1].
- **Discrete Doppler Influence:** The effective visual influence of the Doppler effect depends on $N$ [S1]. The minimum velocity $v/c$ that has a visual influence is determined by $D = 1 + \Delta\lambda/(2\lambda)$, where $D = 1 + v/c$ [S1]. For example, with $N=40$, $\lambda \in [390, 790]$ nm, $\Delta\lambda=10$ nm, at $\lambda=390$ nm, $v/c \approx 0.0128$, and at $\lambda=790$ nm, $v/c \approx 0.0063$ [S1].
- **No Line Broadening:** The model cannot reproduce Doppler line broadening, as single spectral lines ($\delta$-peaks) are represented as rectangles of width $\Delta\lambda$ [S1].

## Subtopics

### [Piecewise Linear Spectrum Model](4709-piecewise-linear-spectrum-model.md)
A continuous spectral representation that replaces discrete binning with exact positions and intensities, ensuring energy conservation and accurate Doppler shift representation. It models spectral lines as trapezoidal bands with linearly varying intensity, allowing for precise handling of non-uniform spectral intervals, unlike the fixed bands of the Point-Sampled model.

## Operating characteristics
- **Time Complexity:** The complexity is mainly characterized by the number of spectral bands $N$ [S1]. For each ray step, the algorithm iterates over $N$ bands for black-body emission and $N \times M$ bands for material emission/absorption, where $M$ is the number of materials [S1].
- **Memory Complexity:** Requires storage for $N$ bands per ray and pre-computed spectra of size $N$ per material [S1].
- **Strengths:** Simple to implement; additive nature of spectral radiance allows straightforward accumulation [S1].
- **Limitations:** Suffers from energy conservation loss, information loss during Doppler shifts, and accuracy issues due to fixed bands [S1]. Requires small step sizes for high densities to avoid amplification artifacts [S1].
- **When to use:** Suitable for low-extinction scenes or when computational simplicity is prioritized over spectral accuracy [S1].
- **When not to use:** High-density volumes, scenes requiring accurate Doppler shift visualization, or when energy conservation is critical [S1].

## Background
The Point-Sampled Spectrum Model is described in the context of Doppler Volume Rendering for astrophysics simulations [S1]. It is presented as the baseline method before introducing the novel Piecewise Linear Spectrum Model [S1]. The method relies on the additive nature of spectral radiance and the linearization of the volume rendering integral [S1].

## Sources
[S1] Doppler Volume Rendering: A Dynamic, Piecewise Linear Spectral Representation for Visualizing Astrophysics Simulations, https://albertojaspe.net/publications/2023-EVIS-doppler_volume.pdf, fetched 2026-10-28
