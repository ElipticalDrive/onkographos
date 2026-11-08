# Doppler Volume Rendering

Also called: Piecewise Linear Spectral Representation, Spectral Volume Rendering with Doppler Effect

## Overview
Doppler Volume Rendering is a direct volume ray casting technique that simulates the Doppler effect on light emitted or absorbed by volumetric matter moving according to a 3D velocity field [S1]. Unlike standard volume rendering which maps scalar values to static RGB colors, this method computes spectral radiance at each sample, shifting wavelengths based on the relative motion between the emitter and the observer to produce physically accurate color shifts [S1]. It distinguishes itself from traditional spectral volume rendering by employing a dynamic piecewise linear spectral representation, which avoids the energy conservation errors and spectral broadening artifacts inherent in point-sampled (bin-based) spectral discretization when velocities vary along the ray [S1].

## How it works
The method extends the standard emission-absorption model by integrating spectral radiance rather than scalar opacity/color [S1]. It supports two emission models: thermal radiation (blackbody) and non-thermal radiation (spectral lines from atomic transitions) [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Scalar volume data $V(\mathbf{x})$ representing density $\rho$ [S1].
    *   Vector volume data $\mathbf{u}(\mathbf{x})$ representing the 3D velocity field [S1].
    *   Temperature volume data $T(\mathbf{x})$ for thermal emission [S1].
    *   Material properties defining spectral lines (e.g., Hydrogen Balmer lines) or blackbody parameters [S1].
    *   Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
*   **Outputs:**
    *   Final spectral radiance per pixel, which is mapped to RGB for display [S1].

### Data Structures
*   **Piecewise Linear Spectral Representation:** Instead of storing a value per frequency bin, the spectrum is represented as a series of linear segments defined by vertices $(\lambda_i, L_i)$, where $\lambda$ is wavelength and $L$ is spectral radiance [S1]. This allows for exact shifting of spectral features without binning artifacts [S1].
*   **Velocity Field:** A 3D vector grid $\mathbf{u}$ aligned with the scalar volume grid [S1].

### Steps

1.  **Ray Generation and Discretization:** For each pixel, generate a ray $r(t) = \mathbf{o} + t\mathbf{d}$ intersecting the volume bounding box $[t_{start}, t_{end}]$ [S1]. Discretize the segment into $N$ samples with equidistant step $\Delta t = (t_{end} - t_{start}) / N$ [S1].
2.  **Sample Processing:** For each sample point $\mathbf{p}_i$ at distance $t_i$:
    a.  **Interpolation:** Trilinearly interpolate the density $\rho_i$, velocity $\mathbf{u}_i$, and temperature $T_i$ from the volume grids [G].
    b.  **Emission Calculation:** Compute the emitted spectral radiance $L_{emit}(\lambda)$ at $\mathbf{p}_i$ [S1].
        *   **Thermal Emission:** Use Planck's radiation law [S1]:
            $$ B_\lambda(T, \lambda) = \frac{2hc^2}{\lambda^5} \left[ \exp\left(\frac{hc}{\lambda k_B T}\right) - 1 \right]^{-1} $$
            where $h$ is Planck's constant, $k_B$ is Boltzmann's constant, $c$ is the speed of light, and $T$ is the interpolated temperature [S1].
        *   **Non-Thermal Emission:** Define spectral lines as piecewise linear functions centered at specific rest wavelengths $\lambda_0$ (e.g., Hydrogen Balmer lines) [S1].
    c.  **Doppler Shift:** Calculate the observed wavelength $\lambda'$ for a ray of rest wavelength $\lambda$. The shift depends on the relative velocity along the line of sight [S1]. The relativistic Doppler factor for a receding emitter is $\sqrt{\frac{1+v_r/c}{1-v_r/c}}$, where $v_r$ is the radial velocity component [G]. The piecewise linear spectrum is shifted by transforming all vertex wavelengths $\lambda_j \to \lambda'_j$ [S1].
    d.  **Absorption/Attenuation:** Compute the optical depth for the current segment. The attenuation follows the Lambert–Bouguer law [S1]:
        $$ I(\lambda) = I_0(\lambda) e^{-\Delta t \cdot \rho_i \cdot \tau(\lambda)} $$
        where $\rho_i$ is the density, $\Delta t$ is the step size, and $\tau(\lambda)$ is the extinction coefficient at wavelength $\lambda$ [S1]. For spectral lines, $\tau(\lambda)$ is high at line centers and low elsewhere [S1].
3.  **Compositing:** Integrate the shifted and attenuated spectral radiance from front to back [S1]. Let $L'_{accum}(\lambda)$ be the accumulated spectral radiance and $T'_{accum}(\lambda)$ be the accumulated transmittance (initially $0$ and $1$, respectively) [S1]. For sample $i$:
    $$ L'_{accum}(\lambda) = L'_{accum}(\lambda) + T'_{accum}(\lambda) \cdot L_{shifted, emit}(\lambda) \cdot (1 - e^{-\Delta t \cdot \rho_i \cdot \tau(\lambda)}) $$
    $$ T'_{accum}(\lambda) = T'_{accum}(\lambda) \cdot e^{-\Delta t \cdot \rho_i \cdot \tau(\lambda)} $$
    Here, $L_{shifted, emit}(\lambda)$ is the emission spectrum at the sample, shifted to the observer's frame [S1].
4.  **Color Mapping:** After integrating along the entire ray, the final spectral radiance is mapped to RGB values for display [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Physical constants
H = 6.62607015e-34  # Planck's constant (J·s)
KB = 1.380649e-23   # Boltzmann's constant (J/K)
C = 299792458.0     # Speed of light (m/s)

def planck_lambda(T, wavelength):
    """Planck's radiation law per unit wavelength."""
    if wavelength <= 0 or T <= 0:
        return 0.0
    exponent = (H * C) / (wavelength * KB * T)
    return (2 * H * C**2 / wavelength**5) * (1.0 / (math.exp(exponent) - 1))

def ray_box_intersection(origin, direction, box_min, box_max):
    """Returns (t_start, t_end) or None if no intersection."""
    t_min = (box_min[0] - origin[0]) / direction[0]
    t_max = (box_max[0] - origin[0]) / direction[0]
    if t_min > t_max: t_min, t_max = t_max, t_min
    
    t_min_y = (box_min[1] - origin[1]) / direction[1]
    t_max_y = (box_max[1] - origin[1]) / direction[1]
    if t_min_y > t_max_y: t_min_y, t_max_y = t_max_y, t_min_y
    
    if t_min > t_max_y or t_min_y > t_max: return None
    t_min = max(t_min, t_min_y)
    t_max = min(t_max, t_max_y)
    
    t_min_z = (box_min[2] - origin[2]) / direction[2]
    t_max_z = (box_max[2] - origin[2]) / direction[2]
    if t_min_z > t_max_z: t_min_z, t_max_z = t_max_z, t_min_z
    
    if t_min > t_max_z or t_min_z > t_max: return None
    t_start = max(t_min, t_min_z)
    t_end = min(t_max, t_max_z)
    
    if t_start > t_end: return None
    return (t_start, t_end)

def trilinear_interpolate(grid, x, y, z):
    """Standard trilinear interpolation [G]."""
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    ix, iy, iz = max(0, min(ix, grid.dim-2)), max(0, min(iy, grid.dim-2)), max(0, min(iz, grid.dim-2))
    
    def lerp(v0, v1, t): return v0 + t * (v1 - v0)
    c00 = lerp(grid[ix, iy, iz], grid[ix+1, iy, iz], fx)
    c01 = lerp(grid[ix, iy+1, iz], grid[ix+1, iy+1, iz], fx)
    c10 = lerp(grid[ix, iy, iz+1], grid[ix+1, iy, iz+1], fx)
    c11 = lerp(grid[ix, iy+1, iz+1], grid[ix+1, iy+1, iz+1], fx)
    return lerp(lerp(c00, c01, fy), lerp(c10, c11, fy), fz)

def render_doppler_volume(ray_origin, ray_direction, density_grid, velocity_grid, temp_grid, 
                          spectral_lines, box_min, box_max, num_samples):
    """
    Renders a single ray using Doppler Volume Rendering.
    spectral_lines: List of (rest_wavelength, peak_extinction, width) tuples.
    """
    bounds = ray_box_intersection(ray_origin, ray_direction, box_min, box_max)
    if bounds is None:
        return (0.0, 0.0, 0.0, 0.0)
    
    t_start, t_end = bounds
    dt = (t_end - t_start) / num_samples
    
    # Track wavelengths for RGB mapping and spectral lines
    wavelengths = list(range(380, 780, 5)) + [line[0] for line in spectral_lines]
    wavelengths = [w * 1e-9 for w in wavelengths] # Convert to meters
    
    L_accum = {lam: 0.0 for lam in wavelengths}
    T_accum = {lam: 1.0 for lam in wavelengths}
    
    for i in range(num_samples):
        t = t_start + i * dt
        p = (ray_origin[0] + t * ray_direction[0],
             ray_origin[1] + t * ray_direction[1],
             ray_origin[2] + t * ray_direction[2])
        
        rho = trilinear_interpolate(density_grid, p[0], p[1], p[2])
        temp = trilinear_interpolate(temp_grid, p[0], p[1], p[2])
        vel = (trilinear_interpolate(velocity_grid, p[0], p[1], p[2]),
               trilinear_interpolate(velocity_grid, p[0]+1, p[1], p[2]),
               trilinear_interpolate(velocity_grid, p[0], p[1]+1, p[2])) # Simplified vector interp [G]
        
        # Radial velocity (positive if receding)
        v_radial = vel[0]*ray_direction[0] + vel[1]*ray_direction[1] + vel[2]*ray_direction[2]
        beta = v_radial / C
        
        for lam in wavelengths:
            # Inverse Doppler: find rest wavelength that shifts into 'lam'
            # lambda_rest = lambda_obs * sqrt((1-beta)/(1+beta))
            shift_factor = math.sqrt((1 - beta) / (1 + beta))
            lam_rest = lam * shift_factor
            
            # Emission at rest wavelength
            L_emit = planck_lambda(temp, lam_rest)
            for line_rest, peak, width in spectral_lines:
                dist = lam_rest - line_rest
                if abs(dist) < width * 3:
                    L_emit += peak * math.exp(-(dist**2) / (2 * width**2))
            
            # Extinction at observed wavelength
            tau_base = rho * dt * 0.1
            for line_rest, peak, width in spectral_lines:
                lam_line_obs = line_rest / shift_factor
                dist = lam - lam_line_obs
                if abs(dist) < width * 3:
                    tau_base += peak * math.exp(-(dist**2) / (2 * width**2)) * rho * dt
            
            transmittance_seg = math.exp(-tau_base)
            
            # Front-to-back spectral compositing
            L_accum[lam] += T_accum[lam] * L_emit * (1.0 - transmittance_seg)
            T_accum[lam] *= transmittance_seg
            
    # Map accumulated spectrum to RGB [G]
    R = sum(L_accum[lam] for lam in wavelengths if 380 <= lam*1e9 < 440)
    G = sum(L_accum[lam] for lam in wavelengths if 440 <= lam*1e9 < 580)
    B = sum(L_accum[lam] for lam in wavelengths if 580 <= lam*1e9 <= 780)
    
    return (R, G, B, 1.0 - T_accum[wavelengths[0]])
```

### Parameters and Edge Cases
*   **Spectral Resolution:** The piecewise linear representation requires sufficient vertices to capture narrow spectral lines. Point-sampled methods suffer from spectral broadening errors due to varying velocities along ray segments, which this method avoids by dynamically computing and combining piecewise linear segments [S1].
*   **Thermal vs. Non-Thermal:** The method must correctly combine or switch between Planck's law and spectral line definitions based on the material type [S1].
*   **Velocity Field Alignment:** The velocity field must be aligned with the scalar volume grid to ensure accurate Doppler computation [S1].

## Operating characteristics
*   **Performance:** Designed for interactive frame rates on GPUs [S1]. The dynamic piecewise linear representation avoids the need for large spectral bins, reducing memory bandwidth compared to full spectral volume rendering [S1].
*   **Strengths:** Physically accurate color representation for moving media; handles both thermal and line emission; avoids energy conservation errors of bin-based methods [S1].
*   **Limitations:** Requires velocity and temperature fields; computationally more expensive than standard RGB volume rendering; complex transfer function design for spectral data [S1].

## Background
Doppler Volume Rendering was introduced by Alberto Jaspe et al. in 2023 [S1]. It builds on spectral volume rendering introduced by Noordmans et al., which used point-sampled spectra. The authors identified that point-sampled methods suffer from energy conservation inaccuracies and spectral broadening when applied to the Doppler effect in astrophysics simulations [S1]. The method was developed to visualize galaxy formation simulations, where gas clouds move at high velocities, causing significant Doppler shifts in spectral lines like Hydrogen Balmer lines and Carbon lines [S1].

## Sources
[S1] Doppler Volume Rendering: A Dynamic, Piecewise Linear Spectral Representation for Visualizing Astrophysics Simulations, https://albertojaspe.net/publications/2023-EVIS-doppler_volume.pdf, fetched 2026-10-28
