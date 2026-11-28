# Planck Transfer Function
## Overview
The Planck Transfer Function maps volumetric scalar fields (typically temperature and density) to optical properties by modeling black body radiation convolved with spectral filters [S2]. It solves the problem of physically plausible emission and absorption for hot gases or plasmas, distinguishing itself from generic artistic transfer functions by grounding color and intensity in physical emission models [S1, S3].
## How it works
**Inputs and Outputs:**
*   **Inputs:** A scalar field representing temperature ($T$) and a scalar field representing density or optical depth parameter ($\rho$) [S1, S3].
*   **Outputs:** Opacity ($\alpha$) and Color ($C = [r, g, b]$) for each sample point [S5, unofficial].

**Data Structures:**
The function maintains two tabulated lookup tables (LUTs) discretized into $n_{bins}$ intervals [S2].
1.  **Emission Table:** Maps temperature $T$ to spectral radiance values for Red, Green, and Blue channels [S2].
2.  **Absorption Table:** Maps density $\rho$ to absorption coefficients [S2].

**Parameters:**
*   $T_{bounds}$: Tuple $(T_{min}, T_{max})$ defining the range of tabulated emission [S2].
*   $\rho_{bounds}$: Tuple $(\rho_{min}, \rho_{max})$ defining the range of tabulated absorption [S2].
*   $n_{bins}$: Number of bins for discretization (default 256) [S2].
*   $filter_{R}, filter_{G}, filter_{B}$: Johnson filter names (default 'R', 'V', 'B') [S2].
*   $a_{norm}$: Normalization fudge factor for scattering approximation (default $1,000,000.0$) [S2].

**Method Steps:**
1.  **Black Body Calculation:** For a given temperature $T$, calculate the spectral radiance $B(\lambda, T)$ using Planck's Law:
    $$ B(\lambda, T) = \frac{2hc^2}{\lambda^5} \frac{1}{e^{\frac{hc}{\lambda k_B T}} - 1} $$
    where $h$ is Planck's constant, $c$ is the speed of light, and $k_B$ is Boltzmann's constant [S2].
2.  **Filter Convolution:** Integrate $B(\lambda, T)$ over the spectral response curves of the specified Johnson filters ($R, V, B$) to obtain raw channel intensities $I_R, I_G, I_B$ [S2].
3.  **Normalization:** Scale the intensities by $a_{norm}$ to adjust relative reddening and match the scattering approximation [S2].
    $$ I'_{c} = a_{norm} \cdot I_{c} \quad \text{for } c \in \{R, G, B\} $$
4.  **Absorption Mapping:** Calculate absorption coefficient $\alpha_{coeff}$ based on density $\rho$ within $\rho_{bounds}$ [S2].
5.  **Composite Output:** The final opacity and color for a sample are derived from the emission intensities and absorption coefficient, then added to integration channels during ray marching [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def planck_lambda(wavelength, temp):
    """Calculate Planck's Law spectral radiance."""
    h = 6.626e-34
    c = 3.0e8
    kb = 1.38e-23
    exponent = (h * c) / (wavelength * kb * temp)
    if exponent > 100:
        return 0.0
    return (2 * h * c**2) / (wavelength**5 * (np.exp(exponent) - 1))

def get_johnson_filter_response(filter_name, wavelength):
    """Return spectral response curve for a Johnson filter [G]."""
    # Standard rectangular approximation for visible spectrum [G]
    return 1.0 if 400e-9 <= wavelength <= 700e-9 else 0.0

def evaluate_planck_tf(temp, density, t_bounds, rho_bounds, anorm, filters):
    """
    Evaluate Planck Transfer Function.
    temp: scalar temperature
    density: scalar density
    t_bounds: (t_min, t_max)
    rho_bounds: (rho_min, rho_max)
    anorm: normalization factor
    filters: dict {'R': filter_func, 'G': filter_func, 'B': filter_func}
    """
    # 1. Clamp temperature to bounds
    t_clamped = np.clip(temp, t_bounds[0], t_bounds[1])
    
    # 2. Calculate Black Body Emission per channel
    wavelengths = np.linspace(400e-9, 700e-9, 100)
    d_lambda = wavelengths[1] - wavelengths[0]
    
    emission = {}
    for channel, filter_func in filters.items():
        radiance = 0.0
        for w in wavelengths:
            b_val = planck_lambda(w, t_clamped)
            f_val = filter_func(w)
            radiance += b_val * f_val * d_lambda
        # 3. Normalize
        emission[channel] = anorm * radiance
        
    # 4. Calculate Absorption from density
    rho_min, rho_max = rho_bounds
    if rho_max == rho_min:
        alpha_coeff = 0.0
    else:
        alpha_coeff = (density - rho_min) / (rho_max - rho_min)
        alpha_coeff = np.clip(alpha_coeff, 0.0, 1.0)
        
    # 5. Return Color and Opacity
    color = np.array([emission['R'], emission['G'], emission['B']])
    opacity = alpha_coeff
    
    return color, opacity
```

## Subtopics

### Black Body Radiation Model
A variant used primarily for fire visualization. It transitions between an artistic look (parameter 0) and a realistic physically-based intensity (parameter 1) [S4]. In realistic mode, it multiplies the fire intensity by the Black Body Radiation model, giving strong brightness to hot parts of the fire [S4]. Controlled by the `emBlackBody` parameter [S4].

### Blackbody Emission
A mode for volume emission that converts temperature in Kelvin to physically correct color and intensity, simulating incandescence [S5, unofficial]. ~3,000K yields deep red; higher temperatures become white/blue-white [S5, unofficial]. Contrasts with Color mode, which uses user-defined gradients for artistic deviation from physical laws [S5, unofficial].

### Planck Transfer Function Initialization
Initialized with temperature range $(T_{min}, T_{max})$, intensity range $(0.0, 1.0)$, and channel mappings [S6, unofficial]. Parameters `red`, `green`, `blue` specify field names or identifiers for RGB channels (e.g., 'I', 'R', 'U') [S6, unofficial]. Pitfall: The underlying field (e.g., RelativeDensity) is a clamped ratio of density and is not computed in log space [S6, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Not given in the sources.
*   **Memory:** Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Provides physically accurate color for hot objects; handles relative reddening via $a_{norm}$ [S2].
*   **Limitations:** The normalization is arbitrary ("fudge factor") and unit-free, potentially leading to inaccurate relative reddening if not tuned [S2]. Not suitable for compositing fire mixed with smoke without additional opacity handling [S4].

## Background
Implemented in the `yt` visualization library for astrophysics data rendering [S1, S2]. Also referenced in Voxel Shader implementations for fire effects in Maya/Phoenix to enhance realism [S4].

## Sources
[S1] 3D Visualization and Volume Rendering, https://yt-project.org/doc/visualizing/volume_rendering.html, fetched 2026-10-28
[S2] ColorTransferFunction yt.visualization.volume_rendering.transfer_functions.ColorTransferFunction, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.transfer_functions.html, fetched 2026-10-29
[S3] https://yt-project.org/docs/dev/visualizing/volume_rendering.html#volume-rendering-components, https://yt-project.org/docs/dev/visualizing/volume_rendering.html, fetched 2026-10-29
[S4] Voxel Shader - Phoenix for Maya, https://documentation.chaos.com/space/PHX4MAYA/125768009, fetched 2026-10-29
[S5] ボリュームレンダリングとは？雲から爆発まで, https://www.maxon.net/ja/article/volumetric-rendering?srsltid=AYCX9vK-JecQAKHX24X4ZdtRcP9GsOjvBnGwCqB51LzXFIekQMframwI, fetched 2026-10-30 (unofficial)
[S6] Volume rendering colormap is not directly exposed, https://github.com/yt-project/yt/issues/2804, fetched 2026-10-27 (unofficial)
