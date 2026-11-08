# Volume Single Scattering (Uniform Ray Marching with Point Light Background)


## Overview
This method computes single scattering radiance in a participating medium using uniform ray marching, where the background radiance is determined by a point light model rather than a generic surface object color [S1, unofficial]. It differs from its parent [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by using deterministic uniform sampling along the ray instead of Monte Carlo integration, and from siblings like [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md) by explicitly decoupling the background illumination calculation into a specific point light function [S1, unofficial].

## How it works
**Inputs:**
- `ray`: Origin $O$ and direction $D$.
- `volume`: Homogeneous volume with extinction coefficient $\sigma_t$ (or density $\rho$ and extinction cross-section).
- `objects`: List of scene objects (triangles, spheres).
- `light`: Point light source definition (position, intensity).
- `t_MAX`: Maximum ray distance (intersection with scene boundary or far plane).
- `N`: Number of uniform steps in the marching loop.

**Outputs:**
- `Li`: Final radiance at the pixel.

**Data Structures:**
- `Sphere`: Defined by radius $r$, center $C$, color/albedo, exponent (for specular if applicable), and radiance [S1, unofficial].
- `Ray`: Origin $O$, direction $D$ (normalized).
- `Vector3`: 3D floating-point vector.

**Formulas:**
1.  **Transmittance (Homogeneous):**
    $$ T(t) = e^{-\sigma_t t} $$
    Where $\sigma_t$ is the total extinction coefficient and $t$ is the distance traveled [S1, unofficial].

2.  **Background Radiance (Point Light):**
    Computed via `getPixelColorPointLight(objects, id, t_MAX, ray, light)`. This function traces a ray from the light to the scene or calculates direct illumination from the point light at the hit point, ignoring volume scattering for the background itself [S1, unofficial].

3.  **Emission from Light (Le):**
    Computed via `getPixelColorPointLight(objects, -1, t_march, ray, light)`. This calculates the radiance contributed by the point light at the current marching position $t_{march}$ [S1, unofficial].

**Steps:**
1.  **Initialize Background:**
    Calculate the background radiance $L_{bg}$ by tracing the primary ray to its maximum extent $t_{MAX}$ using the point light model:
    $$ L_{bg} = \text{getPixelColorPointLight}(\text{objects}, \text{id}, t_{MAX}, \text{ray}, \text{light}) $$
    [S1, unofficial]

2.  **Calculate Initial Transmittance:**
    Compute the transmittance from the eye (ray origin) to the hit point at $t_{MAX}$:
    $$ T_{eye} = e^{-\sigma_t t_{MAX}} $$
    [S1, unofficial]

3.  **Initialize Radiance:**
    Set the accumulated radiance $L_i$ to the attenuated background radiance:
    $$ L_i = L_{bg} \cdot T_{eye} $$
    [S1, unofficial]

4.  **Uniform Ray Marching Loop:**
    For $i = 0$ to $N-1$:
    a.  **Calculate Step Distance:**
        $$ t_{march} = \frac{i + 0.5}{N} \cdot t_{MAX} $$
        (Midpoint of the uniform interval) [G]
    
    b.  **Calculate Transmittance from Eye:**
        Compute transmittance from the eye to the current marching position:
        $$ T_{eye\_step} = e^{-\sigma_t t_{march}} $$
        [S1, unofficial]
    
    c.  **Calculate Light Contribution (Le):**
        Compute the radiance emitted by the point light at the current position $P = O + t_{march} D$:
        $$ L_e = \text{getPixelColorPointLight}(\text{objects}, -1, t_{march}, \text{ray}, \text{light}) $$
        [S1, unofficial]
    
    d.  **Calculate Transmittance to Light:**
        Compute transmittance from the current position $P$ to the light source. Let $d_{light}$ be the distance from $P$ to the light position $L_{pos}$:
        $$ T_{light\_step} = e^{-\sigma_t d_{light}} $$
        [S1, unofficial]
    
    e.  **Accumulate Scattering:**
        Add the scattered light to $L_i$. The contribution is the light emission attenuated by the transmittance from the eye to the point, and the light emission itself is attenuated by the transmittance from the point to the light (if the light is treated as a source of radiance entering the volume). Note: Standard single scattering formulation is $L_s = \int T(0, t) \sigma_s \Phi(t) dt$. Here, $\Phi(t)$ is the light intensity at $t$.
        $$ L_i += T_{eye\_step} \cdot \sigma_s \cdot L_e \cdot T_{light\_step} \cdot \Delta t $$
        Where $\sigma_s$ is the scattering coefficient and $\Delta t = t_{MAX} / N$. [G]
        *Note: The source notes mention `transmittanceFromLight` but do not explicitly show the multiplication with $\sigma_s$ or $\Delta t$. However, standard volume rendering requires these. If the source implies $L_e$ already includes geometric attenuation, the formula simplifies. Given the alias "Point Light Background", it is likely $L_e$ is the incident radiance from the light.*

**Edge Cases and Pitfalls:**
-   **Precision:** Exponential transmittance can underflow for large $\sigma_t t$. Use `exp(-max(sigma_t * t, 50.0))` or similar clamping if necessary [G].
-   **Boundary:** If $t_{MAX}$ is not correctly determined (e.g., ray misses all objects), $L_{bg}$ should be 0 or environment color.
-   **Light Inside Volume:** If the light is inside the volume, $d_{light}$ can be 0, leading to $T_{light\_step}=1$. Ensure $L_e$ handles singularity at light position.

## Subtopics

### [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md)
Uses stochastic sampling instead of uniform steps. Choose when variance reduction techniques are needed or when the light distribution is complex.

### [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md)
Handles in-scattering from ambient light and out-scattering. Choose when ambient illumination is significant.

### [Single Scattering Approximation with Precomputed Terms](4188-single-scattering-approximation-with-precomputed-terms.md)
Precomputes scattering terms for real-time performance. Choose for interactive applications.

### [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md)
Uses closed-form solutions for simple geometries. Choose for high accuracy in simple scenes.

### [Volume Single Scattering (Monte Carlo with Sphere Light)](4549-volume-single-scattering-monte-carlo-with-sphere-light.md)
Uses sphere lights and Monte Carlo. Choose for soft shadows from area lights.

### [BSSRDF Approximation with Dipole and Single Scattering](4750-bssrdf-approximation-with-dipole-and-single-scattering.md)
Combines subsurface scattering with single scattering. Choose for skin-like materials.

### [Physical Atmosphere Color Calculation](4820-physical-atmosphere-color-calculation.md)
Specialized for atmospheric scattering. Choose for sky and horizon rendering.

## Operating characteristics
-   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of marching steps. Each step requires a transmittance calculation and a light evaluation.
-   **Memory Complexity:** $O(1)$ auxiliary memory per pixel.
-   **Performance:** Fast for small $N$. Accuracy improves with $N$. The point light evaluation can be expensive if it involves ray tracing against complex geometry.
-   **Strengths:** Simple to implement, deterministic, no noise.
-   **Limitations:** Aliasing artifacts if $N$ is too low. Does not handle multiple scattering. Background calculation assumes point light model, which may not match complex scene lighting.
-   **When to Use:** Real-time volume rendering with simple lighting, educational purposes, or as a baseline for comparison.
-   **When Not to Use:** High-fidelity offline rendering requiring multiple scattering, or scenes with complex area lights where point light approximation is insufficient.

## Background
This method is a variant of uniform ray marching described in the repository `wangyuehi/VolumeSingleScattering` [S1, unofficial]. It is part of a family of single scattering approximations that simplify the rendering equation by neglecting multiple scattering events. The use of a point light background simplifies the calculation of the boundary condition for the radiative transfer equation.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
