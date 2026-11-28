# Procedural Cloud Modeling via Level Set and FBM

Also called: Analytical Cloud Structure Method, Pyroclastic Noise-based Cloud Modeling

## Overview
This method provides an analytical form for calculating cloud density at any point in space, enabling the procedural generation of various cloud types (e.g., cumulus, stratus) and cloud layers for flight simulators without relying on particle systems or voxelization [S1]. It distinguishes itself by using a level set boundary defined by a hollow spherical primitive, deformed by a virtual space point displaced via Fractal Brownian Motion (FBM), allowing for real-time controllability and variability of cloud geometry and density [S1]. It builds on [Weather & Cloud Parameterization](3617-weather-cloud-parameterization.md) for global physical and visual parameters.

## How it works
**Inputs:**
*   Point in space $x \in \mathbb{R}^3$.
*   Cloud layer parameters:
    *   $A$: Height of the cloud layer above sea level.
    *   $T$: Thickness of the cloud layer.
    *   $C$: Coverage degree (sparsity) of the cloud layer.
    *   $P$: Spatial template characterizing the average spatial size of a cloud (high-frequency for cumulus/wavy) [S1].
    *   $\rho_0$: Base density, determining cloud color and overall contour clarity [S1].
    *   $F$: Flattening of the base, observed primarily in cumulus clouds [S1].
    *   $H$: Height amplitude, determining the sharpness of height differences in the cloud layer and responsible for gaps in the layer [S1].
*   Turbulence parameters:
    *   $O$: Number of octaves.
    *   $L$: Lacunarity.
    *   $G$: Gain.
*   Level set bounds: $R_{inner}$ and $R_{outer}$ defining the hollow spherical primitive [S1].

**Outputs:**
*   Density $\rho(x)$ at point $x$. If $\rho(x) \le 0$, the cloud is considered non-existent at that point [S1].

**Data Structures:**
*   The method is stateless and analytical; no persistent data structures are required for the density calculation itself. Parameters are passed as uniforms or structure members.

**Steps:**

1.  **Define Level Set Bounds:**
    Establish the radial bounds $[R_{inner}, R_{outer}]$ for the hollow spherical primitive. The density is non-zero only if the point's distance from the center lies within these bounds [S1].

2.  **Calculate FBM Turbulence:**
    Compute the Fractal Brownian Motion value $fbm$ at position $x$ using Perlin noise $N(x)$:
    $$
    fbm = N(x) \cdot G^O
    $$
    where $N(x)$ is the Perlin noise function [S1]. Note: The source formula is presented as $fbm = N(x) * G^O$, implying standard FBM structure using Perlin noise, gain $G$, and octaves $O$ [S1].

3.  **Define Virtual Space Point:**
    Displace the position $x$ by the turbulence scaled by height amplitude $H$ to create a virtual space point $p(x)$:
    $$
    p(x) = x + fbm \cdot H
    $$
    [S1]

4.  **Define Axial Surface:**
    Construct a sphere with a radius equal to the intermediate value of the bounding radii to define the "axial" surface $c(x)$. The source text provides a garbled formula: `c(x): = + . | |`. Conceptually, this represents the mean radius surface. Let $R_{mid} = \frac{R_{inner} + R_{outer}}{2}$. The axial surface is defined by the distance from the origin to this mean radius, modulated by flattening $F$ if necessary to adjust the level set boundary relative to the limits [S1].
    *Implementation Note:* The exact formula for $c(x)$ is not clearly recoverable from the garbled text `= + . | |`. However, it serves as the reference surface for distance calculation. For implementation, $c(x)$ is typically the magnitude $|x|$ adjusted by $F$. We assume $c(x) = |x| \cdot (1 - F)$ or similar, but strictly following the source, it is the "axial surface" defined by the intermediate radius. Let us define the distance to the axial surface as $d_{axial}(x) = | |x| - R_{mid} |$. The parameter $F$ modifies this level.

5.  **Calculate Distance to Axial Surface:**
    Determine the distance $d(x)$ from the point $x$ to the axial surface using the virtual point $p(x)$:
    $$
    d(x) = | p(x) - c(x) |
    $$
    *Correction based on source:* The source states: "define distance $d(x)$ from point $x$ to 'axial' surface: $d(x) = | p(x) - c(x) |$". This implies $c(x)$ is a vector or point, but earlier it says "axial surface $c(x)$". In level set methods, $c(x)$ is often the signed distance function or the implicit surface equation. Given $p(x)$ is a point, $c(x)$ must be a point on the surface closest to $p(x)$, or $c(x)$ is the center. The source text `|! −` suggests $| p(x) - \text{center} |$. Let us assume $c(x)$ refers to the center of the sphere (origin) or the radial projection. The most consistent interpretation of "distance from $x$ to axial surface" via virtual point $p(x)$ is:
    $$
    d(x) = | |p(x)| - R_{mid} |
    $$
    However, the source explicitly writes $d(x) = | p(x) - c(x) |$. If $c(x)$ is the axial surface, it is a scalar field in level set terms. Let us stick to the literal variable names: $d(x)$ is the distance derived from $p(x)$ and the axial surface definition.

6.  **Calculate Deviation:**
    Compute the deviation $D(x)$ of the level set boundary:
    $$
    D(x) = d(x) \cdot (1 + fbm \cdot L + G)
    $$
    The source formula is garbled: `D(x) = '()1 + & " +,`. Contextually, it links deviation to distance $d(x)$, FBM, lacunarity $L$, and gain $G$ [S1].

7.  **Calculate Final Density:**
    Compute the density $\rho(x)$:
    $$
    \rho(x) = \rho_0 \cdot D(x) - d(x)
    $$
    This is calculated only when $d(x)$ is within the level set bounds $(0, +)$, meaning the point is within the hollow sphere [S1]. This creates spatial inhomogeneity in density.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_cloud_density(x, params):
    """
    Calculate cloud density at point x using the Analytical Cloud Structure Method.
    
    Args:
        x: vec3, position in space.
        params: dict containing:
            R_inner: float, inner radius of hollow sphere.
            R_outer: float, outer radius of hollow sphere.
            rho_0: float, base density.
            F: float, flattening of the base.
            H: float, height amplitude.
            O: int, number of octaves.
            L: float, lacunarity.
            G: float, gain.
            C: float, coverage degree (used to adjust bounds or threshold).
            
    Returns:
        float, density rho(x).
    """
    # 1. Define Level Set Bounds
    R_mid = (params['R_inner'] + params['R_outer']) / 2.0
    
    # Check if point is within the hollow sphere bounds (simplified check)
    dist_origin = length(x)
    if dist_origin < params['R_inner'] or dist_origin > params['R_outer']:
        return 0.0
        
    # 2. Calculate FBM Turbulence
    # N(x) is Perlin noise
    noise_val = perlin_noise(x)
    fbm = noise_val * (params['G'] ** params['O'])
    
    # 3. Define Virtual Space Point
    # p(x) = x + fbm * H
    # Note: H is a scalar amplitude. fbm is scalar.
    p_x = x + (fbm * params['H'])
    
    # 4 & 5. Distance to Axial Surface
    # The source defines d(x) = | p(x) - c(x) | where c(x) is the axial surface.
    # Conceptually, c(x) is the radial projection onto the mean radius sphere.
    # c(x) = (p_x / |p_x|) * R_mid  (if p_x is not zero)
    # Then d(x) = | p_x - c(x) |
    
    p_x_len = length(p_x)
    if p_x_len == 0:
        d_x = R_mid # Distance from center to surface
    else:
        # Project p_x onto the sphere of radius R_mid
        c_x = normalize(p_x) * R_mid
        d_x = length(p_x - c_x)
        
    # Apply flattening F to the axial surface definition?
    # Source: "Changing parameter flattening F, it is possible to change the level of 'axial' surface"
    # This might modify R_mid or the distance calculation.
    # Assuming F modifies the effective radius or distance scaling.
    # Without explicit formula, we apply F as a modifier to the distance or radius.
    # Let's assume F scales the deviation impact.
    
    # 6. Calculate Deviation D(x)
    # D(x) = d(x) * (1 + fbm * L + G)
    D_x = d_x * (1.0 + fbm * params['L'] + params['G'])
    
    # 7. Calculate Final Density
    # rho(x) = rho_0 * D(x) - d(x)
    rho = params['rho_0'] * D_x - d_x
    
    # Ensure density is non-negative if required by renderer, 
    # though source says rho <= 0 means non-existent.
    return rho
```

**Edge Cases and Pitfalls:**
*   **High-Resolution Maps:** High-resolution density maps may be excessive. To add high-precision structural details during visualization, a fractal transformation is applied at render time rather than pre-computing [S1].
*   **Garbled Formulas:** The source text contains garbled formulas for $c(x)$ and $D(x)$. The implementation above uses the most logical geometric interpretation of "distance to axial surface" and "deviation" based on the surrounding text.
*   **Bounds:** Density is strictly zero outside $[R_{inner}, R_{outer}]$ [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per point evaluation, dominated by the Perlin noise calculation.
*   **Memory Complexity:** $O(1)$, as no voxel grid or texture maps are required for the core density function [S1].
*   **Performance:** Suitable for real-time rendering in flight simulators [S1].
*   **Strengths:** Controllability, variability, and plausibility without particle systems or voxelization [S1]. Allows simulation of complex weather conditions with a simple set of parameters [S1].
*   **Limitations:** Relies on analytical approximations; may lack the fine-grained detail of high-resolution voxel maps without post-processing fractal transformations [S1].

## Background
*   **Origin:** Proposed by V.V. Podlesny in 2013 [S1].
*   **Paper:** "A method for procedural modeling of clouds of various types for a flight simulator" [S1].
*   **History:** Developed for virtual reality systems of flight simulators to provide instructors with a tool for simulating various cloud types and layers [S1].
*   **Usage:** Applied in aviation training simulators for real-time cloud rendering [S1].

## Sources
[S1] A method for procedural modeling of clouds of various types for a flight simulator, https://doi.org/10.15622/sp.28.11, fetched 2026-10-26
