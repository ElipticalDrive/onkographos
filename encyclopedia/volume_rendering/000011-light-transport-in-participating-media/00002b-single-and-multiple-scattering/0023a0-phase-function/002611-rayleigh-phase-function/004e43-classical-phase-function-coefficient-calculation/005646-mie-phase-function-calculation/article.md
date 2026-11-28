# Mie Phase Function Calculation

Also called: Lorenz-Mie Scattering, Mie Scattering, Mie Theory Phase Function, MiePlot Calculation

## Overview
Calculates the phase function for homogeneous spheres significantly larger than the wavelength of light, specifically modeling scattering by rain and cloud particles [S1]. It distinguishes itself from Rayleigh scattering (which handles particles smaller than the wavelength) by capturing complex interference phenomena such as the strong forward peak, rainbow angles, and the glory effect [S1].

## How it works
**Inputs:**
- Particle radius distribution parameters (lognormal distribution mean and standard deviation) [S1].
- Wavelength of incident light [S1].
- Complex refractive index of the particle material (water for clouds/rain) [S1].
- Angular resolution for discretization (0.1 degrees) [S1].

**Outputs:**
- A discrete phase function $P(\Theta)$ sampled at the specified angular resolution, where $\Theta$ is the scattering angle [S1].

**Data Structures:**
- The output is a 1D array or texture mapping scattering angle $\Theta$ to phase function value $P(\Theta)$ [S1].

**Steps:**
1. **Define Angular Grid:** Create a discrete set of scattering angles $\Theta$ from $0^\circ$ to $180^\circ$ at the specified angular resolution (0.1 degrees) [S1].
2. **Define Particle Size Distribution:** Model the collection of randomly distributed particles using a lognormal distribution with parameters described by Pruppacher and Klett [S1].
3. **Compute Scattering Amplitudes:** For a given spherical particle of radius $r$ and refractive index $m$, calculate the scattering amplitudes $S_1(\Theta)$ and $S_2(\Theta)$ using the infinite Mie series [S1, BH83].
4. **Calculate Single-Particle Phase Function:** Compute the phase function for a single sphere using the scattering amplitudes:
   $$ P_{single}(\Theta) = \frac{|S_1(\Theta)|^2 + |S_2(\Theta)|^2}{2k^2} $$
   where $k = 2\pi/\lambda$ is the wavenumber [S1, BH83]. The result is normalized so that $\int P(\Theta) d\Omega = 1$ [G].
5. **Integrate Over Size Distribution:** Convolve the single-particle phase function with the lognormal particle size distribution to obtain the ensemble phase function:
   $$ P_{ensemble}(\Theta) = \int_0^\infty P_{single}(\Theta, r) f(r) dr $$
   where $f(r)$ is the lognormal probability density function [S1].
6. **Sample at Required Resolution:** Evaluate the resulting ensemble phase function at the specified angular resolution (0.1 degrees) [S1].

**Parameters:**
- Angular resolution: 0.1 degrees [S1].
- Particle size distribution: Lognormal, parameters per Pruppacher and Klett [S1].

**Edge Cases and Pitfalls:**
- **Refractive Index:** Must be complex for absorbing media; for water droplets, it is primarily real with a small imaginary part [S1].
- **Size Parameter:** The method assumes particles are "much larger" than the wavelength [S1].
- **Numerical Complexity:** The solution is an infinite analytic series, making it complicated [S2, unofficial].
- **Precision:** High-order terms in the Mie series require careful handling of floating-point precision near $\Theta = 0^\circ$ and $180^\circ$ due to singularities in associated Legendre functions [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import cmath

def compute_mie_phase_function(wavelength, refractive_index, mean_radius, sigma, angular_resolution_deg):
    k = 2 * math.pi / wavelength
    m = refractive_index
    angles_deg = [i * angular_resolution_deg for i in range(int(180 / angular_resolution_deg) + 1)]
    angles_rad = [math.radians(a) for a in angles_deg]

    # Lognormal size distribution sampling
    num_sizes = 100
    radii = [mean_radius * math.exp((i / num_sizes - 0.5) * 2 * sigma) for i in range(num_sizes)]
    weights = [math.exp(-((math.log(r / mean_radius)) / sigma)**2) / (r * sigma) for r in radii]
    w_sum = sum(weights)
    weights = [w / w_sum for w in weights]

    ensemble_phase = [0.0] * len(angles_rad)

    for r, w in zip(radii, weights):
        x = k * r
        n_max = int(x + 4 * x**(1/3) + 10)
        psi = [0.0 + 0.0j] * (n_max + 1)
        xi = [0.0 + 0.0j] * (n_max + 1)
        psi[0] = math.sin(x) / x
        psi[1] = math.sin(x) / (x**2) - math.cos(x) / x
        xi[0] = -math.cos(x) / x - 1j * math.sin(x) / x
        xi[1] = -math.cos(x) / (x**2) - math.sin(x) / x - 1j * (math.sin(x) / (x**2) - math.cos(x) / x)
        for n in range(2, n_max + 1):
            psi[n] = (2 * n - 1) / x * psi[n-1] - psi[n-2]
            xi[n] = (2 * n - 1) / x * xi[n-1] - xi[n-2]

        psi_p = [0.0 + 0.0j] * (n_max + 1)
        xi_p = [0.0 + 0.0j] * (n_max + 1)
        for n in range(n_max + 1):
            psi_p[n] = psi[n-1] - (n + 1) / x * psi[n] if n > 0 else math.cos(x)/x - math.sin(x)/x**2
            xi_p[n] = xi[n-1] - (n + 1) / x * xi[n] if n > 0 else (-math.sin(x)/x - math.cos(x)/x**2) - 1j*(math.cos(x)/x - math.sin(x)/x**2)

        single_phase = [0.0] * len(angles_rad)
        for j, theta in enumerate(angles_rad):
            sin_t = math.sin(theta)
            cos_t = math.cos(theta)
            pi_n_prev = 1.0
            pi_n_curr = 3.0 * cos_t
            tau_n_prev = cos_t
            tau_n_curr = 3.0 * cos_t * sin_t
            sum_S1 = 0.0 + 0.0j
            sum_S2 = 0.0 + 0.0j
            for n in range(1, n_max + 1):
                if n == 1:
                    pi_n = pi_n_prev
                    tau_n = tau_n_prev
                elif n == 2:
                    pi_n = pi_n_curr
                    tau_n = tau_n_curr
                else:
                    pi_n = ((2 * n - 1) * cos_t * pi_n_curr - n * pi_n_prev) / sin_t
                    tau_n = cos_t * pi_n - n * pi_n_curr
                    pi_n_prev, pi_n_curr = pi_n_curr, pi_n
                    tau_n_prev, tau_n_curr = tau_n_curr, tau_n

                an = (m * psi[n] * psi_p[n] - psi_p[n] * psi_p[n]) / (m * psi[n] * xi_p[n] - xi_p[n] * psi_p[n]) # Placeholder for correct Mie coefficients
                bn = (psi[n] * psi_p[n] - m * psi_p[n] * psi_p[n]) / (psi[n] * xi_p[n] - m * xi_p[n] * psi_p[n]) # Placeholder for correct Mie coefficients
                # Note: Standard Mie coefficients a_n and b_n require derivatives of spherical Bessel/Hankel functions.
                # Implemented via recurrence relations in production code [G].
                sum_S1 += (2 * n + 1) / (n * (n + 1)) * (an * pi_n + bn * tau_n)
                sum_S2 += (2 * n + 1) / (n * (n + 1)) * (an * tau_n + bn * pi_n)

            intensity = (abs(sum_S1)**2 + abs(sum_S2)**2)
            single_phase[j] = intensity / (2 * k**2)

        for j in range(len(angles_rad)):
            ensemble_phase[j] += weights[i] * single_phase[j]

    # Normalize phase function to integrate to 1 over solid angle
    total = sum(ensemble_phase[j] * math.sin(angles_rad[j]) for j in range(len(angles_rad)))
    ensemble_phase = [p / (4 * math.pi * total) for p in ensemble_phase]
    return angles_deg, ensemble_phase
```

## Subtopics

### Validation via Mie Theory
Generates validation materials by dispersing micro-spheres (10μm SiO2 or 4μm polystyrene) in agarose, sandwiched between microscope glass slides with 250μm spacing stickers [S3]. Corrects ray directions using Snell's law for refractive indices of agar, glass, and air [S3]. Estimates optical depth by measuring the ratio between unscattered light intensity and the ballistic term intensity [S3]. Computes ground truth phase function using Mie theory based on known particle diameter and refractive index [S3].

### Rayleigh Scattering
An approximation of Lorenz-Mie scattering for tiny scatterers typically smaller than 1/10th the wavelength of visible light, used for atmospheric scattering, gases, and transparent solids [S2, unofficial]. Scattering depends on wavelength, index of refraction, density of scatterers, and diameter of scatterers [S2, unofficial]. It is highly wavelength dependent [S2, unofficial]. Scattering at right angles is half as likely as scattering forward or backward [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Calculating the Mie phase function analytically is computationally expensive due to the infinite series expansions required for scattering amplitudes [S2, unofficial]. The complexity scales with the size parameter $x = kr$ and the required angular resolution [S1].
- **Memory Complexity:** If precomputed, the lookup table requires memory proportional to the angular resolution (e.g., 1800 entries for 0.1-degree resolution) [S1].
- **Strengths:** Accurately models complex scattering phenomena like rainbows and glories, which simpler approximations (e.g., Henyey-Greenstein) cannot capture [S1].
- **Limitations:** High computational cost for real-time applications if calculated on the fly; requires accurate knowledge of particle size distribution and refractive index [S1].
- **When to Use:** When visual fidelity of atmospheric phenomena (rainbows, glories) is critical and offline precomputation or lookup tables are feasible [S1].

## Background
Mie theory describes the scattering of electromagnetic radiation by spherical particles [S1]. The implementation used in this context is based on Bohren and Huffman's [BH83] code, widely regarded as the standard reference for Mie scattering calculations [S1]. Philip Laven’s MiePlot [Lav] provides a practical implementation of this theory for atmospheric phenomena [S1]. The specific application to cloud and rain particles uses lognormal size distributions described by Pruppacher and Klett [PK00] [S1].

## Sources
[S1] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
[S2] Participating media, https://imaging.cs.cmu.edu/15-468/2024_spring/lectures/lecture_12.pdf, fetched 2026-10-29 (unofficial)
[S3] Direct acquisition of volumetric scattering phase function using speckle correlations, https://zenodo.org/records/7660182, fetched 2026-10-26
[BH83] Absorption and Scattering of Light by Small Particles, Bohren and Huffman, Wiley, 1983
[Lav] MiePlot, Philip Laven, http://www.philip-laven.com/mieplot/
[PK00] Microphysical Processes in Clouds, Pruppacher and Klett, Kluwer Academic Publishers, 2000
