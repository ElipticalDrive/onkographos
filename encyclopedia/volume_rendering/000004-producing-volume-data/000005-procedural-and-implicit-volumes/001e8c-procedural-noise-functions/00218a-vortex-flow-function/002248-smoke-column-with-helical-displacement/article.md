# Smoke Column with Helical Displacement


## Overview
This method generates a single column of rising smoke by deforming a vertical cylinder, simulating dispersion, turbulence, and swirling motion [S1, unofficial]. It builds on the [Vortex Flow Function](2538-vortex-flow-function.md) by explicitly combining vertical helical displacement for bending and swirling with height-dependent density decay and turbulence scaling to simulate the physical dispersion of a smoke plume [S1, unofficial].

## How it works
**Inputs:** Point position $P$, cylinder axis direction (unit vector), cylinder base point $P_{base}$, column height $H$, time parameter $t$, turbulence scale, density decay scale [G].
**Outputs:** Density value $\rho$ at point $P$ [G].
**Data Structures:** No complex data structures required; the method is procedural and stateless [G].

**Steps:**
1. **Normalize Height:** Calculate the normalized height $h$ of the point relative to the cylinder base and top. Clamp $h$ to $[0, 1]$. If $h < 0$ or $h > 1$, the point is outside the column bounds and returns zero density [G].
2. **Helical Displacement:** Displace the point using a vertical helix to simulate bending and swirling before calculating distance from the cylinder center [S1, unofficial]. Compute an angular offset based on height and time, then rotate the horizontal coordinates around the cylinder axis [G].
3. **Radial Distance:** Compute the radial distance $r$ of the displaced point from the cylinder axis [G].
4. **Turbulence:** Increase turbulence as a function of height to simulate air currents, making the column start smooth and become turbulent [S1, unofficial]. Add a noise-based perturbation scaled by height to the radial distance or displacement angle [G].
5. **Density:** Decrease density as a function of height to simulate dispersion [S1, unofficial]. Combine a radial falloff profile with the height decay to compute the final density [G].

**Parameters:**
- Turbulence scale: Controls the magnitude of height-dependent turbulence. Typical value: $1.0$ to $3.0$ [G].
- Density decay scale: Controls the rate of height-dependent density falloff. Typical value: $1.0$ to $2.0$ [G].
- Time $t$: Animation frame or elapsed time. Typical value: $0.0$ to $10.0$ [G].

**Edge Cases and Pitfalls:**
- Points outside the height range $[0, H]$ return zero density [G].
- Floating-point precision at extreme heights may affect noise calculations; use normalized coordinates for noise input [G].
- If $H=0$, the column has no height; handle by returning zero density [G].
- Ensure the noise function is continuous to avoid visible seams in the smoke column [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def smoke_column_density(P, axis, P_base, H, t, turb_scale, density_decay_scale):
    """
    Calculate density of a smoke column with helical displacement.
    P, axis, P_base are 3D vectors (x, y, z).
    """
    # Step 1: Normalize Height
    # Normalize axis to ensure correct projection [G]
    axis_len = math.sqrt(axis[0]**2 + axis[1]**2 + axis[2]**2)
    ux, uy, uz = axis[0]/axis_len, axis[1]/axis_len, axis[2]/axis_len
    
    vec_x, vec_y, vec_z = P[0]-P_base[0], P[1]-P_base[1], P[2]-P_base[2]
    h = (vec_x*ux + vec_y*uy + vec_z*uz) / H
    if h < 0.0 or h > 1.0:
        return 0.0

    # Step 2: Helical Displacement
    theta = h * 2.0 * math.pi + t * 0.5
    x, y = vec_x, vec_y
    x_prime = x * math.cos(theta) - y * math.sin(theta)
    y_prime = x * math.sin(theta) + y * math.cos(theta)

    # Step 3: Radial Distance
    r = math.sqrt(x_prime**2 + y_prime**2)

    # Step 4: Turbulence
    turb_factor = 1.0 + h * turb_scale
    # Simple deterministic pseudo-noise for implementation [G]
    noise_val = math.sin(r * 10.0 + h * 20.0 + t * 5.0) * 0.1
    r_turbulent = r + noise_val * turb_factor

    # Step 5: Density
    rho_radial = math.exp(-(r_turbulent**2) / 0.5)
    rho_height = 1.0 - math.pow(h, density_decay_scale)
    return rho_radial * rho_height
```

## Subtopics

### [Procedural Flame Density & Color Shader](3033-procedural-flame-density-color-shader.md)
This method computes the density and color of a procedural flame volume by deforming a base helical flow path with height-scaled turbulence and applying spatial attenuation ramps. It sets itself apart from the smoke column by introducing a multi-stage attenuation process (horizontal and vertical), explicit vertical fading, minimum density thresholds, and a direct density-to-color mapping that shifts from yellow to red based on density values. Choose this variant when a realistic fire appearance with complex shaping via sine waves and lookup tables is required over pure atmospheric smoke. See [Procedural Flame Density & Color Shader](3033-procedural-flame-density-color-shader.md).

### Synapse Volume Construction
This variant constructs volumetric emission objects (synapses) connecting an inner sphere to an outer sphere, simulating light-emitting plasma [S2, unofficial]. It sets itself apart from the standard smoke column by using a straight-line base geometry with seven layered randomization factors: circular endpoint offset, sinusoidal curve amplitude offset, width offset, Gaussian color gradients (Electric Blue to Neon Pink), exponential endpoint expansion for surface diffusion, and surface illumination on the outer sphere [S2, unofficial]. Choose this variant for artistic plasma or high-voltage discharge simulations rather than atmospheric fluid dynamics [S2, unofficial]. It typically uses 82 instances of the synapse object [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves basic arithmetic and noise evaluation [G].
- **Memory Complexity:** $O(1)$, no data structures required [G].
- **Measured Performance:** Not given in the sources [G].
- **Scaling:** Scales linearly with the number of samples; turbulence noise evaluation can be optimized with precomputed noise grids or fast noise algorithms [G].
- **Strengths:** Simple to implement, computationally inexpensive, produces visually convincing smoke columns with minimal parameters [G].
- **Limitations:** Lacks physical accuracy (e.g., no fluid dynamics simulation), turbulence is procedural and may not match real-world fluid behavior, limited to cylindrical geometries [G].
- **When to Use:** Real-time rendering, background elements, or when performance is critical and physical accuracy is not required [G].
- **When Not to Use:** Scientific visualization requiring accurate fluid dynamics, or scenes with complex interacting smoke plumes [G].

## Background
This method originates from procedural modeling techniques for gases and fluids [S1, unofficial]. It builds on the concept of deforming solid spaces over time to simulate gas motion, specifically using helical displacement to add swirling and bending to a vertical cylinder [S1, unofficial]. The approach is similar to steam rising procedures but adapted for smoke with increased turbulence and density decay [S1, unofficial].

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
[S2] Home, https://graphics.cg.uni-saarland.de/competition/cg1-2018/original/groupR_web/, fetched 2026-10-28 (unofficial)
