# Volume Rendering Equation

Also called: Radiative Transfer Equation Integration, SPH Volumetric Rendering, Transmittance Equation, Volume Rendering Integral, Volumetric Ray-Marching for Gas Rendering, Volumetric Rendering Model

## Overview

The Volume Rendering Equation computes the outgoing radiance at a camera pixel by integrating the emission and scattering of light along a ray passing through a participating medium, attenuated by the accumulated transmittance [S2, S6, unofficial]. It solves the problem of visualizing continuous density fields (such as gas, fog, or clouds) by avoiding the "dots with glow" artifact of discrete particle rendering, instead producing a diffuse, physically plausible gas cloud appearance [S2, S3, unofficial]. Unlike the Standard volume rendering equation which often assumes pre-computed scalar fields and simple compositing, this method explicitly formulates the radiative transfer differential equation, allowing for the construction of emission and absorption fields directly from particle distributions (e.g., SPH) or volume grids [S2, S7, unofficial].

## How it works

### Inputs and Outputs

**Inputs:**
*   **Camera Ray:** A ray $r(t) = p_0 + t \vec{d}$ parameterized by $t \in [0, 1]$, where $p_0$ is the eye position and $\vec{d}$ is the direction vector [S7, unofficial].
*   **Medium Properties:** Functions for density $\sigma(x)$, emitted color/intensity $c(x)$, and optionally absorption/scattering coefficients [S6, S7, unofficial].
*   **Particle Data (Optional):** If using SPH, inputs include particle masses $m_i$, positions $p_i$, smoothing lengths $h_i$, luminosities $L_i$, and cross-sections $\sigma_i$ [S2, unofficial].

**Outputs:**
*   **Pixel Color:** The accumulated radiance $C(D)$ or $I_{out}$ at the exit distance $D$ of the volume [S6, S2, unofficial].

### Data Structures

1.  **3D Density Grid (Texture):** A 3D texture storing pre-computed density $\rho$, emission $j$, and absorption $\kappa$ values.
    *   **Layout:** Trilinearly interpolated floating-point values.
    *   **Size:** Example resolution $128^3$ voxels (approx. 2 million cells) [S2, unofficial].
    *   **Fields:**
        *   `density`: $\rho(x)$
        *   `emission`: $j(x)$ or $c(x)$
        *   `absorption`: $\kappa(x)$ or $\sigma(x)$
2.  **Particle List (CPU/GPU Buffer):**
    *   **Fields:** `position` (vec3), `mass` (float), `luminosity` (float), `cross_section` (float), `smoothing_length` (float) [S2, unofficial].

### Formulas

**1. Radiative Transfer Differential Equation:**
The change in intensity $C(s)$ along a path $s$ is governed by:
$$ \frac{dC(s)}{ds} = \sigma(s)c(s) - \sigma(s)C(s) $$
Where:
*   $C(s)$ is the intensity at path length $s$ [S6, unofficial].
*   $\sigma(s)$ is the opacity (density times cross-section area, $\sigma = A\rho$) [S6, unofficial].
*   $c(s)$ is the emitted light intensity per unit area [S6, unofficial].

**2. Transmittance:**
The transmittance $T(s)$ from the viewer (origin) to point $s$ is:
$$ T(s) = e^{-\int_0^s \sigma(t) dt} $$
This represents the fraction of light that survives without being absorbed or scattered out of the ray [S6, unofficial].

**3. Volume Rendering Integral (Continuous Solution):**
The accumulated color $C(D)$ at distance $D$ is:
$$ C(D) = \int_0^D \sigma(s)c(s)T(s) ds $$
$$ C(D) = \int_0^D \sigma(s)c(s) e^{-\int_0^s \sigma(t) dt} ds $$
[S6, S7, unofficial]

**4. SPH Field Construction (Pre-computation):**
If deriving from particles, fields are constructed using a Gaussian kernel $W(r, h)$:
$$ \rho(p) = \sum_i m_i W(|p - p_i|, h_i) $$
$$ j(p) = \sum_i L_i W(|p - p_i|, h_i) $$
$$ \kappa(p) = \sum_i \sigma_i W(|p - p_i|, h_i) $$
Where $W(r, h)$ is the smoothing kernel with length $h$ [S2, unofficial].

**5. Discrete Ray Marching Approximation:**
For implementation, the integral is approximated by summing contributions over $N$ steps:
$$ C_{out} = \sum_{i=0}^{N-1} \sigma_i c_i \Delta t_i \prod_{j=0}^{i-1} e^{-\sigma_j \Delta t_j} $$
Where $\Delta t_i$ is the step size, and $\sigma_i, c_i$ are sampled at step $i$ [S2, S6, S7, unofficial].

### Parameters

*   **Samples per Ray:** Typically 200 samples for fixed-step marching [S2, unofficial].
*   **Grid Resolution:** $128^3$ voxels is a baseline for real-time performance [S2, unofficial].
*   **Opacity Range:** $\sigma \in [0, 1]$ [S6, unofficial].
*   **Scope Limitations (v1):** Single greyscale channel for radiative transfer; color derived from underlying particles [S2, unofficial]. No anisotropic scattering (Mie/Rayleigh) [S2, unofficial]. No self-consistent radiation pressure [S2, unofficial].

### Steps

1.  **Construct Density Field (Phase A):**
    *   For each particle $i$, compute its contribution to the 3D grid using the Gaussian kernel $W$ [S2, unofficial].
    *   Accumulate mass for density $\rho$, luminosity for emission $j$, and cross-section for absorption $\kappa$ [S2, unofficial].
    *   Bake the resulting fields into a 3D texture [S2, unofficial].

2.  **Ray Marching (Phase B):**
    *   For each pixel, cast a ray from the camera into the volume [S3, unofficial].
    *   Initialize accumulated transmittance $T = 1.0$ and accumulated color $C_{acc} = 0.0$ [S6, unofficial].
    *   Iterate along the ray from $t=0$ to $t=D$ with step size $\Delta t$:
        *   Sample $\sigma(t)$ and $c(t)$ from the 3D texture at current position [S2, unofficial].
        *   Compute local transmittance: $T_{local} = e^{-\sigma(t) \Delta t}$ [S6, unofficial].
        *   Compute emitted contribution: $C_{emit} = \sigma(t) c(t) \Delta t$ [S6, unofficial].
        *   Update accumulated color: $C_{acc} += C_{emit} \times T$ [S6, unofficial].
        *   Update transmittance: $T *= T_{local}$ [S6, unofficial].
        *   If $T < \epsilon$ (threshold), break loop [G].
    *   Composite with background: $C_{final} = C_{acc} + T \times C_{bg}$ [S3, unofficial].

3.  **Optimization (Phase C):**
    *   Use mipmapping on the 3D texture for hierarchical ray marching [S2, unofficial].
    *   Use adaptive step size: increase samples in regions of high density gradient [S2, unofficial].
    *   Re-bake particle grid every $N$ frames or every frame depending on particle motion [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_render_ray(ray_origin, ray_direction, volume_grid, num_samples):
    """
    Renders a single ray through a volume grid using the transmittance equation.
    
    Args:
        ray_origin: vec3, camera position
        ray_direction: vec3, normalized ray direction
        volume_grid: 3D texture object with methods sample_density(pos) and sample_emission(pos)
        num_samples: int, number of steps along the ray
        
    Returns:
        vec3, final pixel color
    """
    # Assume volume bounds are known or ray is clipped to volume
    # For simplicity, assume fixed length D or ray-marching until exit
    step_size = 1.0 / num_samples  # Normalized step size
    accumulated_color = vec3(0.0, 0.0, 0.0)
    transmittance = 1.0
    
    for i in range(num_samples):
        # Current position along ray
        t = i * step_size
        pos = ray_origin + ray_direction * t
        
        # Sample medium properties
        # Note: In practice, check if pos is inside volume bounds
        density = volume_grid.sample_density(pos)
        emission = volume_grid.sample_emission(pos)
        
        # Opacity sigma (density * cross-section, assumed normalized or scaled)
        # Using sigma directly from grid if pre-scaled, or density * constant
        sigma = density 
        
        # Local transmittance for this step: exp(-sigma * delta_s)
        # delta_s is the physical step size, here normalized to 1.0 for simplicity
        # In real implementation, delta_s must match the units of sigma
        delta_s = 1.0 / num_samples 
        local_transmittance = exp(-sigma * delta_s)
        
        # Contribution of emitted light at this point, attenuated by previous transmittance
        # Formula: C += sigma * c * T_prev * delta_s
        # Note: Some formulations absorb delta_s into sigma or c. 
        # Based on [S6, unofficial]: dC/ds = sigma*c - sigma*C. Discrete: C_new = C_old + (sigma*c - sigma*C)*ds
        # However, standard front-to-back compositing [S3, S7, unofficial] uses:
        # alpha = 1 - exp(-sigma * ds)
        # color += alpha * emission * transmittance
        # transmittance *= exp(-sigma * ds)
        
        # Using the standard discrete approximation for front-to-back compositing
        alpha = 1.0 - local_transmittance
        
        accumulated_color += alpha * emission * transmittance
        transmittance *= local_transmittance
        
        # Early exit if fully opaque
        if transmittance < 0.001:
            break
            
    # Composite with background (assumed black or passed in)
    background_color = vec3(0.0, 0.0, 0.0) 
    final_color = accumulated_color + transmittance * background_color
    
    return final_color
```

### Edge Cases and Pitfalls

*   **Sampling Artifacts:** Finite samples cause banding/striping. Solution: Use dithering or jittered sampling [S2, unofficial].
*   **Homogeneous Regions:** In regions with zero gradient, shading intensity may be unreliable. Solution: Base voxel opacity on the gradient; homogeneous regions get zero opacity [S3, unofficial].
*   **Coordinate System:** Derivation assumes origin at viewer's eye ($0$ to $D$). Literature may use back-of-volume origin. Ensure consistency in integration limits [S6, unofficial].
*   **Performance:** Naive ray-marching is $O(\text{pixels} \times \text{samples} \times \text{particles})$. Without spatial indexing (3D grid), it is infeasible [S2, unofficial].
*   **Precision:** Accumulating small transmittance values can lead to precision loss. Use early exit thresholds [G].

## Operating characteristics

*   **Time Complexity:** $O(P \cdot S \cdot G)$ where $P$ is pixels, $S$ is samples per ray, and $G$ is cost of grid lookup (constant with texture). Without grid, $O(P \cdot S \cdot N)$ where $N$ is particles [S2, unofficial].
*   **Memory Complexity:** $O(V)$ for the 3D volume grid, where $V$ is number of voxels (e.g., $128^3 \approx 2$M cells) [S2, unofficial].
*   **Measured Performance:** On a recent laptop GPU, achieving 30 fps at $1280 \times 720$ resolution with 1000 particles and 200 samples per ray requires pre-baking into a 3D grid and GPU shader implementation [S2, unofficial].
*   **Strengths:** Produces physically accurate "diffuse gas" appearance; handles complex density structures; supports real extinction and emission [S2, S7, unofficial].
*   **Limitations:** Computationally expensive for real-time without optimizations (mipmapping, adaptive steps); does not model anisotropic scattering in basic v1 implementations [S2, unofficial].
*   **When to Use:** Real-time cosmology renders, gas clouds, fog, smoke where continuous density is required [S2, S7, unofficial].
*   **When Not to Use:** When discrete particle halos are stylistically desired; when anisotropic scattering (Mie/Rayleigh) is critical and not approximated [S2, unofficial].

## Background

The Volume Rendering Equation is rooted in the Radiative Transfer Equation (RTE) from astrophysics and atmospheric physics [S2, S6, unofficial]. In computer graphics, it was formalized by Kajiya in 1986 with the Rendering Equation, which generalized the problem to include scattering [S1, unofficial]. Kajiya and Von Herzen later presented algorithms for tracing volume densities (clouds, fog) in 1984, developing light scattering equations and an approximate solution suitable for graphics [S4, unofficial]. The specific front-to-back compositing approach for ray casting was popularized by Marc Levoy in the context of medical imaging (CT scans), where it avoids the artifacts of surface extraction [S3, unofficial]. The SPH-specific volumetric rendering approach described in the notes is a modern application of these principles to smoothed-particle hydrodynamics data, aiming to replicate cosmology visualizations like Illustris-TNG [S2, unofficial].

## Sources

[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Volumetric ray-marching — proper cosmology-quality gas rendering, https://github.com/billymahmood/universesimulator/issues/17, fetched 2026-10-26 (unofficial)
[S3] Volume Rendering With Ray Casting - WPI, https://web.cs.wpi.edu/~matt/courses/cs563/talks/powwie/p1/ray-cast.htm, fetched 2026-10-26 (unofficial)
[S4] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S5] Volumetric Lighting (God Rays) — How It Works - Moonjump, https://moonjump.com/game-dev-mechanics-volumetric-lighting-god-rays-how-it-works/, fetched 2026-10-28 (unofficial)
[S6] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
[S7] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
