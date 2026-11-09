# Absorbing Particle Rendering

Also called: Direct Volume Rendering (Absorption Only)

## Overview
Absorbing Particle Rendering simulates a participating medium composed of perfectly black particles that absorb all intercepted light without scattering or emitting [S3]. It solves for light intensity attenuation through a medium, resembling a cloud of black smoke or traditional X-ray imagery [S1, unofficial]. Unlike the Density-Emitter Model, it ignores emission, and unlike the Prism Model, it treats particles as discrete absorbers rather than continuous prisms [S3153-prism-model.md].

## How it works
**Inputs:**
- Initial light intensity $I_0$ at the entry point of the volume.
- Extinction coefficient $k$ of the medium.
- Distance $x$ traveled through the medium.

**Outputs:**
- Transmitted light intensity $I(x)$ at the exit point.

**Data Structures:**
The method relies on scalar parameters derived from the medium's physical properties:
- $A$: Projected area of a single spherical particle ($A = \pi r^2$, where $r$ is the particle radius) [S3].
- $N$: Number of particles per unit volume (number density) [S3].
- $k$: Extinction coefficient [S3].
- $\alpha$: Opacity, defined such that transparency is $1 - \alpha$ [S3].

**Formulas:**
The change in light intensity $I$ over a differential distance $dx$ is governed by the differential equation:
$$ \frac{dI}{dx} = -k I $$
where $k$ is the extinction coefficient [S3].

The solution for intensity after traveling distance $x$ is:
$$ I(x) = I_0 e^{-k x} $$
where $I_0$ is the initial light intensity [S3].

The extinction coefficient $k$ is derived from the particle properties. For a unit volume approximated as a small cylindrical slab with base area $A$ and thickness $dx$, the intensity absorbed is determined by the total projected area of particles within that volume. The extinction coefficient is given by:
$$ k = N A $$
where $N$ is the number of particles per unit volume and $A$ is the area of each particle [S3].

**Parameters:**
- $r$: Radius of the spherical particles. Typical values depend on the scene scale; for example, discrete media simulations may use geometric mean radii $\bar{r}_g$ around 800 units in specific benchmark contexts [S2].
- $\eta_i$: The imaginary part of the complex refractive index. In contexts involving scattering modifications, increasing $\eta_i$ reduces the intensity of scattering, but in the pure absorption-only model, particles are assumed perfectly black [S2][S3].

**Steps:**
1. Calculate the projected area of a single particle: $A = \pi r^2$.
2. Determine the number density $N$ (particles per unit volume) [S3].
3. Compute the extinction coefficient: $k = N \cdot A$ [S3].
4. For a viewing ray traversing a segment of length $x$ through the medium, compute the transmitted intensity: $I(x) = I_0 e^{-k x}$ [S3].
5. If implementing discretely, accumulate opacity $\alpha = 1 - e^{-k x}$ for compositing [S3].

**Edge Cases and Pitfalls:**
- **Precision:** For large optical depths ($k x \gg 1$), $I(x)$ approaches 0. Standard floating-point precision is sufficient, but underflow to zero may occur for very dense media.
- **Boundary Conditions:** The model assumes a homogeneous medium with constant $k$. If $N$ or $r$ varies spatially, $k$ must be integrated numerically along the ray.
- **Degenerate Input:** If $N=0$, $k=0$ and $I(x) = I_0$ (no attenuation). If $r=0$, $A=0$ and no attenuation occurs.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def absorbing_particle_rendering(I0, r, N, x):
    """
    Calculate transmitted light intensity through an absorbing particle medium.
    
    Args:
        I0 (float): Initial light intensity at the start of the volume.
        r (float): Radius of the spherical particles.
        N (float): Number of particles per unit volume.
        x (float): Distance traveled through the medium.
        
    Returns:
        float: Transmitted light intensity I(x).
    """
    # Step 1: Calculate projected area of a single particle
    A = 3.141592653589793 * r * r
    
    # Step 2: Calculate extinction coefficient
    k = N * A
    
    # Step 3: Calculate transmitted intensity using Beer-Lambert law
    # I(x) = I0 * exp(-k * x)
    import math
    I_x = I0 * math.exp(-k * x)
    
    return I_x
```

## Subtopics

### X-ray simulation via ray integration
Simulates attenuation using similar integration principles but often accounts for energy-dependent absorption coefficients rather than discrete particle geometry [1445-x-ray-simulation-via-ray-integration.md].

### Prism Model
Approximates the medium as a continuous distribution of light-absorbing prisms rather than discrete spherical particles, offering different computational trade-offs [3153-prism-model.md].

### Absorption Attenuation
A general term for the physical phenomenon; this method is a specific implementation using discrete particle assumptions [7104-absorption-attenuation.md].

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray segment if $k$ is constant. If $k$ varies, it requires numerical integration along the ray, typically $O(N_{steps})$ where $N_{steps}$ is the number of integration steps.
- **Memory Complexity:** $O(1)$ if parameters are precomputed. $O(N)$ if storing explicit particle positions for sparse sampling.
- **Performance:** Rendering time is minimal as it involves only exponential calculations. Benchmarks for discrete media show lowest rendering times at resolutions where there is roughly one particle per voxel (e.g., $60 \times 60 \times 60$ for $2 \times 10^5$ particles) [S2].
- **Strengths:** Computationally efficient; physically intuitive for dense, non-scattering media like smoke or X-ray attenuation.
- **Limitations:** Cannot represent scattering effects (glory, halos); assumes spherical particles; ignores emission.
- **When to use:** Simulating black smoke, fog, or X-ray projections where scattering is negligible.
- **When not to use:** When realistic lighting with shadows, caustics, or color bleeding from scattering is required.

## Background
The absorption-only model is a foundational concept in Direct Volume Rendering, historically used to simulate media that only block light [S1, unofficial]. It is derived from the Beer-Lambert law applied to discrete particulate media. The model assumes perfectly black particles that absorb light perfectly [S3]. It serves as a baseline for more complex models that add emission or scattering [S1, unofficial].

## Sources
[S1] Direct Volume Rendering Data, https://www.scribd.com/document/1050958395/Direct-Volume-Rendering-Data, fetched 2026-10-25 (unofficial)
[S2] Rendering discrete participating media using geometrical optics approximation, https://doi.org/10.1007/s41095-021-0253-5, fetched 2026-10-26
[S3] Direct_Volume_Rendering.pdf, https://github.com/pyvista/pyvista/files/9562451/Direct_Volume_Rendering.pdf, fetched 2026-10-29
