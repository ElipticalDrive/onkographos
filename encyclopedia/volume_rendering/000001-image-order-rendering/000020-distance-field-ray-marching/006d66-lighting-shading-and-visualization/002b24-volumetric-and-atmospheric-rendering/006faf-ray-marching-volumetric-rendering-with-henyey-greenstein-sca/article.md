# Ray Marching Volumetric Rendering with Henyey-Greenstein Scattering

Also called: GLSL Volumetric Rendering, TouchDesigner Volumetric Rendering

## Overview
This method renders volumetric scenes by discretizing the Radiative Transfer Equation (RTE) along camera rays using fixed-step ray marching, evaluating density via a procedural fractal Signed Distance Field (SDF), and computing single-scattering light contributions using the Henyey-Greenstein phase function [S1, unofficial]. It distinguishes itself from sibling methods like [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by explicitly mixing forward and backward scattering coefficients to control anisotropy, and from [Volumetric Rendering with Spherical Harmonics](4962-volumetric-rendering-with-spherical-harmonics.md) by avoiding frequency-domain approximations in favor of direct spatial integration [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   `fragCoord`: Pixel coordinates $(u, v)$ in the output image resolution.
*   `iResolution`: Image width and height $(W, H)$.
*   `iTime`: Global animation time variable.
*   **Parameters:**
    *   `RAY_ITER`: Number of ray marching steps (default: 120) [S1, unofficial].
    *   `LIGHT_ITER`: Number of shadow sampling steps (default: 16) [S1, unofficial].
    *   `RAY_STEP_SIZE`: Distance advanced per ray step (default: 0.01) [S1, unofficial].
    *   `LIGHT_ATTEN`: Light attenuation coefficient (default: 64.0) [S1, unofficial].
    *   `FOWARD`: Forward scattering anisotropy parameter $g_f$ (default: 0.8) [S1, unofficial].
    *   `BACKWARD`: Backward scattering anisotropy parameter $g_b$ (default: -0.2) [S1, unofficial].

**Outputs:**
*   `fragColor`: Final RGBA color for the pixel.

### Data Structures
No complex data structures are used; the method relies on scalar variables and 3D vectors. The density field is defined implicitly by the `sdf` function.

### Algorithm Steps

1.  **Initialization**:
    *   Map pixel coordinates to normalized device coordinates: $\mathbf{uv} = \left(\frac{u}{W} - 0.5\right) \cdot 2.0, \left(\frac{v}{H} - 0.5\right) \cdot 2.0$.
    *   Set camera origin $\mathbf{o} = (0.0, 0.0, -2.0)$.
    *   Compute ray direction $\mathbf{d} = \text{normalize}((\mathbf{uv}, 0.0) - \mathbf{o})$.
    *   Initialize accumulated color $\mathbf{C} = (0.0, 0.0, 0.0)$ and transmittance $T = 1.0$.
    *   Set current ray position $\mathbf{p} = \mathbf{o} + \mathbf{d}$ (initial step offset) [S1, unofficial].
    *   Compute dynamic light position $\mathbf{l}$ using a Lissajous curve (see below).

2.  **Ray Marching Loop** (iterate $i$ from 0 to `RAY_ITER`):
    *   Advance position: $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{d} \cdot \text{RAY\_STEP\_SIZE}$.
    *   Evaluate density $\rho = \text{sampleDensity}(\mathbf{p})$.
    *   **Pitfall**: If $\rho \le 0.0$, skip to the next iteration (empty space) [S1, unofficial].
    *   **Early Termination**: If $T < 0.01$, break the loop [S1, unofficial].

3.  **Lighting and Scattering**:
    *   Compute light direction vector $\mathbf{l}_{dir} = \mathbf{l} - \mathbf{p}$.
    *   Compute light distance $D_l = \|\mathbf{l}_{dir}\|$.
    *   Normalize light direction: $\mathbf{l}_{dir} \leftarrow \mathbf{l}_{dir} / D_l$.
    *   Compute light step size: $\Delta t_l = D_l / \text{LIGHT\_ITER}$.
    *   Compute dot product for phase function: $\mu = \text{dot}(\mathbf{d}, -\mathbf{l}_{dir})$.
    *   Compute scattering weight $S = \text{getScattering}(\mu)$ (see Phase Function below).

4.  **Shadow Density Accumulation**:
    *   Initialize shadow density $\rho_{shadow} = 0.0$.
    *   Set light ray position $\mathbf{p}_l = \mathbf{p}$.
    *   Iterate $j$ from 0 to `LIGHT_ITER`:
        *   $\rho_{shadow} \leftarrow \rho_{shadow} + \text{sampleDensity}(\mathbf{p}_l) \cdot \Delta t_l$.
        *   $\mathbf{p}_l \leftarrow \mathbf{p}_l + \mathbf{l}_{dir} \cdot \Delta t_l$.

5.  **Color and Transmittance Update**:
    *   Compute attenuated external light: $\mathbf{L}_{ext} = \exp(-\rho_{shadow} \cdot \text{LIGHT\_ATTEN}) \cdot S$.
    *   Update accumulated color: $\mathbf{C} \leftarrow \mathbf{C} + T \cdot \mathbf{L}_{ext} \cdot \rho$.
    *   Update transmittance: $T \leftarrow T \cdot \exp(-\rho \cdot \text{RAY\_STEP\_SIZE} \cdot \text{LIGHT\_ATTEN})$.

6.  **Post-Processing**:
    *   Apply gamma correction: $\mathbf{C} \leftarrow \mathbf{C}^{(1/2.2)}$.
    *   Output $\mathbf{C}$ as `fragColor`.

### Formulas

**Henyey-Greenstein Phase Function:**
$$ P(\mu, g) = \frac{1 - g^2}{(1 + g^2 - 2g\mu)^{1.5}} $$
Where $\mu = \cos \theta = \text{dot}(\mathbf{d}, -\mathbf{l}_{dir})$ and $g$ is the anisotropy parameter [S1, unofficial].

**Scattering Mix:**
$$ S(\mu) = 0.5 \cdot P(\mu, g_f) + 0.5 \cdot P(\mu, g_b) $$
Where $g_f = \text{FOWARD}$ and $g_b = \text{BACKWARD}$ [S1, unofficial].

**Density Sampling:**
$$ \rho(\mathbf{p}) = 10.0 \cdot \max(-\text{sdf}(\mathbf{p}), 0.0)^{1.3} $$
Where `sdf` is the procedural fractal distance field [S1, unofficial].

**Light Position (Lissajous Curve):**
$$ \mathbf{l}(t) = \begin{bmatrix} A \sin(a t + \delta) \\ B \sin(b t) \\ C \sin(c t) \end{bmatrix} $$
With $A=1.5, B=1.2, C=1.1, a=3.1, b=2.2, c=4.3, \delta=0.2$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def rotate_vector(z, axis, angle):
    """Rotate vector z around axis by angle."""
    s = math.sin(angle)
    c = math.cos(angle)
    ax, ay, az = axis
    # Rotation matrix components
    c1 = 1.0 - c
    m00 = c + ax * ax * c1
    m01 = ax * ay * c1 - az * s
    m02 = ax * az * c1 + ay * s
    m10 = ay * ax * c1 + az * s
    m11 = c + ay * ay * c1
    m12 = ay * az * c1 - ax * s
    m20 = az * ax * c1 - ay * s
    m21 = az * ay * c1 + ax * s
    m22 = c + az * az * c1
    
    # Apply rotation
    new_x = m00 * z[0] + m01 * z[1] + m02 * z[2]
    new_y = m10 * z[0] + m11 * z[1] + m12 * z[2]
    new_z = m20 * z[0] + m21 * z[1] + m22 * z[2]
    return np.array([new_x, new_y, new_z])

def fractal_sdf(p, time):
    """Compute procedural fractal SDF."""
    current_p = p.copy()
    for i in range(8):
        # Rotate around X and Y axes
        current_p = rotate_vector(current_p, np.array([1.0, 0.0, 0.0]), time * 0.2)
        current_p = rotate_vector(current_p, np.array([0.0, 1.0, 0.0]), time * 0.1)
        
        # Reflective symmetry checks
        if current_p[0] + current_p[1] < 0.0:
            current_p[0], current_p[1] = -current_p[1], -current_p[0]
        if current_p[1] + current_p[2] < 0.0:
            current_p[1], current_p[2] = -current_p[2], -current_p[1]
        if current_p[2] + current_p[0] < 0.0:
            current_p[2], current_p[0] = -current_p[0], -current_p[2]
            
        # Scale down and translate
        current_p -= 0.06
        
    return np.linalg.norm(current_p) - 0.15

def sample_density(p, time):
    """Convert SDF to density."""
    d = fractal_sdf(p, time)
    return max(-d, 0.0) ** 1.3 * 10.0

def henyey_greenstein(mu, g):
    """Henyey-Greenstein phase function."""
    gg = g * g
    denom = 1.0 + gg - 2.0 * g * mu
    if denom <= 0:
        return 0.0
    return (1.0 - gg) / (denom ** 1.5)

def get_scattering(mu, g_forward, g_backward):
    """Mix forward and backward scattering."""
    return 0.5 * henyey_greenstein(mu, g_forward) + 0.5 * henyey_greenstein(mu, g_backward)

def light_pos_lissajous(t):
    """Compute light position using Lissajous curve."""
    A, B, C = 1.5, 1.2, 1.1
    a, b, c = 3.1, 2.2, 4.3
    delta = 0.2
    x = A * math.sin(a * t + delta)
    y = B * math.sin(b * t)
    z = C * math.sin(c * t)
    return np.array([x, y, z])

def render_pixel(u, v, resolution, time, params):
    """Render a single pixel."""
    W, H = resolution
    RAY_ITER = params['RAY_ITER']
    LIGHT_ITER = params['LIGHT_ITER']
    RAY_STEP_SIZE = params['RAY_STEP_SIZE']
    LIGHT_ATTEN = params['LIGHT_ATTEN']
    FOWARD = params['FOWARD']
    BACKWARD = params['BACKWARD']
    
    # UV mapping
    uv = np.array([(u / W - 0.5) * 2.0, (v / H - 0.5) * 2.0, 0.0])
    
    # Camera setup
    cam_pos = np.array([0.0, 0.0, -2.0])
    ray_dir = uv - cam_pos
    ray_dir = ray_dir / np.linalg.norm(ray_dir)
    
    # Light setup
    light_pos = light_pos_lissajous(time * 0.2)
    
    # Initialization
    col = np.array([0.0, 0.0, 0.0])
    transmittance = 1.0
    ray_pos = cam_pos + ray_dir
    
    for i in range(RAY_ITER):
        # Advance ray
        ray_pos += ray_dir * RAY_STEP_SIZE
        
        # Sample density
        density = sample_density(ray_pos, time)
        
        # Skip empty space
        if density <= 0.0:
            continue
            
        # Early termination
        if transmittance < 0.01:
            break
            
        # Lighting calculation
        light_dir_vec = light_pos - ray_pos
        light_dist = np.linalg.norm(light_dir_vec)
        light_dir = light_dir_vec / light_dist
        light_step = light_dist / LIGHT_ITER
        
        # Phase function input
        mu = np.dot(ray_dir, -light_dir)
        scattering = get_scattering(mu, FOWARD, BACKWARD)
        
        # Shadow density accumulation
        shadow_density = 0.0
        light_ray_pos = ray_pos
        for j in range(LIGHT_ITER):
            shadow_density += sample_density(light_ray_pos, time) * light_step
            light_ray_pos += light_dir * light_step
            
        # Color update
        external_light = np.array([math.exp(-shadow_density * LIGHT_ATTEN) * scattering] * 3)
        col += transmittance * external_light * density
        
        # Transmittance update
        transmittance *= math.exp(-density * RAY_STEP_SIZE * LIGHT_ATTEN)
        
    # Gamma correction
    col = col ** (1.0 / 2.2)
    return col
```

### Edge Cases and Pitfalls
*   **Empty Space**: If `sampleDensity` returns $\le 0.0$, the iteration must be skipped to avoid division by zero or invalid scattering calculations in subsequent steps, although the primary loop guard handles this [S1, unofficial].
*   **Transmittance Threshold**: Breaking the loop when $T < 0.01$ is a standard optimization but may cause artifacts in very dense volumes if the threshold is too high [S1, unofficial].
*   **Singularity**: The Henyey-Greenstein denominator $(1 + g^2 - 2g\mu)$ approaches zero if $g \approx 1$ and $\mu \approx 1$ (perfect forward scattering). The implementation should guard against division by zero or negative denominators [S1, unofficial].

## Subtopics

### Procedural Density Generation
Uses a fractal SDF with rotational symmetry and reflection to create complex volumetric shapes without texture maps. Chosen when memory bandwidth is limited or dynamic deformation is required [S1, unofficial].

### Lissajous Light Motion
The light source moves along a parametric curve. Chosen to demonstrate dynamic lighting effects in real-time ray marching [S1, unofficial].

## Operating characteristics

*   **Time Complexity**: $O(R \cdot L)$ per pixel, where $R$ is `RAY_ITER` and $L$ is `LIGHT_ITER`. With typical values ($120 \times 16$), this is computationally expensive for high-resolution images [S1, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory per pixel, as the density field is procedural and no voxel grids are stored [S1, unofficial].
*   **Performance**: Suitable for real-time rendering on GPUs (GLSL/WebGPU) due to the lack of data structure traversal overhead, but limited by the nested loop for shadow density [S1, unofficial].
*   **Strengths**: Simple implementation, no pre-computation required, supports dynamic geometry via the SDF [S1, unofficial].
*   **Limitations**: Single-scattering approximation only; no multiple scattering or global illumination effects [S1, unofficial].

## Background

This method originates from GLSL shader programming tutorials, specifically popularized by the "Okdalto" blog post detailing a "rigorous" version of volumetric rendering in GLSL [S1, unofficial]. It is frequently implemented in creative coding environments like TouchDesigner to demonstrate real-time volumetric effects using procedural noise and SDFs [S1, unofficial]. The use of the Henyey-Greenstein phase function is standard in atmospheric and particulate rendering to model anisotropic scattering [S1, unofficial].

## Sources

[S1] GLSL Volumetric Rendering (The Rigorous, Harder Version) · Okdalto, https://okdalto.github.io/en/glsl-volumetric-rendering-rigorous/, fetched 2026-10-30 (unofficial)
