# Probe-based Global Illumination (DDGI-like)

Also called: DDGI, DDGI implementation, Probe Grid, Probe-based Methods (DDGI Baseline)

## Overview
A probe-based global illumination technique implemented for quantitative comparison with NIV, supporting visibility-based probe rejection using ray tracing [S1]. It differs from its parent [Dynamic Diffuse Global Illumination (DDGI)](8142-dynamic-diffuse-global-illumination-ddgi.md) by replacing depth-texture-based visibility with ray-traced visibility to eliminate aliasing and reduce memory overhead, while retaining the core structure of spherical harmonics probes [S1].

## How it works
### Inputs and Outputs
- **Inputs**: Scene geometry (for ray tracing), camera position, shading point position, and shading normal [S1].
- **Outputs**: Approximated indirect irradiance at the shading point [S1].

### Data Structures
- **Probe Grid**: A regular 3D grid of probes [S1].
- **Probe Storage Layout**: Each probe stores directional indirect irradiance represented by second-order spherical harmonics (9 coefficients) [RH01]. The coefficients are quantized to half-precision after baking, resulting in 54 bytes per probe ($9 \times 3 \times 2$ bytes, representing RGB irradiance per coefficient) [S1]. Alternative quantization to 28 bytes per probe is possible but has little influence on results [S1].
- **Visibility**: Computed via ray tracing from the shading point to the probe locations [S1].

### Steps
1. **Ray Traced Visibility**: Compute visibility using ray tracing rather than depth textures [S1].
2. **Cosine Fall-off**: Apply a cosine fall-off [Hoo16] such that probes behind the sampled normal do not contribute to trilinear interpolation [S1].
3. **Weight Clamping**: Clamp interpolation weights to a small number ($10^{-6}$) to avoid no probes contributing to a shading point [S1].
4. **Re-normalization**: Re-normalize interpolation weights after applying the heuristics (cosine fall-off and clamping) [S1].
5. **Trilinear Interpolation**: Perform trilinear interpolation of the SH coefficients from the surrounding probes in the grid based on the normalized weights [S1].
6. **SH Evaluation**: Evaluate the interpolated SH coefficients to compute the indirect irradiance at the shading point [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_probe_illumination(shading_point, shading_normal, probe_grid):
    """
    Computes indirect irradiance using a probe grid with ray-traced visibility.
    
    Args:
        shading_point: tuple(float, float, float), world position of the shading point.
        shading_normal: tuple(float, float, float), normalized surface normal.
        probe_grid: dict containing 'origin', 'spacing', 'dims', and 'probes' (list of 3D arrays).
        
    Returns:
        tuple(float, float, float), the computed indirect irradiance (RGB).
    """
    origin = probe_grid['origin']
    spacing = probe_grid['spacing']
    dims = probe_grid['dims']
    probes = probe_grid['probes']
    
    # 1. Convert world position to grid coordinates
    grid_coords = [(shading_point[i] - origin[i]) / spacing[i] for i in range(3)]
    
    # 2. Determine the 8 surrounding integer grid cells and trilinear weights
    lower = [math.floor(c) for c in grid_coords]
    frac = [grid_coords[i] - lower[i] for i in range(3)]
    
    neighbors = []
    weights = []
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                idx = [lower[i] + (dx if i == 0 else 0) + (dy if i == 1 else 0) + (dz if i == 2 else 0) for i in range(3)]
                # Clamp indices to grid bounds
                idx = [max(0, min(dims[i] - 1, idx[i])) for i in range(3)]
                w = 1.0
                for i in range(3):
                    w *= (frac[i] if (idx[i] == lower[i] + 1) else (1.0 - frac[i]))
                neighbors.append((idx, w))
    
    # 3. Apply visibility, cosine fall-off, and compute final weights
    final_weights = []
    for idx, base_w in neighbors:
        # Direction from shading point to probe center
        probe_pos = [origin[i] + (idx[i] + 0.5) * spacing[i] for i in range(3)]
        direction = [probe_pos[i] - shading_point[i] for i in range(3)]
        length = math.sqrt(sum(d*d for d in direction))
        direction = [d / length for d in direction]
        
        # Ray trace visibility
        is_visible = not ray_trace(shading_point, direction, length)
        
        # Cosine fall-off
        cos_theta = sum(shading_normal[i] * direction[i] for i in range(3))
        weight = base_w * (cos_theta if (is_visible and cos_theta > 0) else 0.0)
        final_weights.append(weight)
        
    # 4. Clamping
    min_weight = 1e-6
    final_weights = [max(w, min_weight) for w in final_weights]
    
    # 5. Re-normalization
    weight_sum = sum(final_weights)
    normalized_weights = [w / weight_sum for w in final_weights]
    
    # 6. Trilinear Interpolation of SH Coefficients
    # SH layout: 9 coefficients * 3 RGB channels
    sh_interp = [0.0] * 27
    for i, (idx, _) in enumerate(neighbors):
        probe_sh = probes[idx[0]][idx[1]][idx[2]]
        for k in range(27):
            sh_interp[k] += normalized_weights[i] * probe_sh[k]
            
    # 7. SH Evaluation
    irradiance = evaluate_sh(sh_interp, shading_normal)
    return irradiance

def ray_trace(origin, direction, max_distance):
    """Returns True if geometry is hit, False if path is clear."""
    # Standard ray-scene intersection test [G]
    return False  # Placeholder for scene-specific intersection logic

def evaluate_sh(sh_coeffs, normal):
    """Evaluates second-order SH basis functions at the normal [G]."""
    # Standard SH irradiance projection for l=2 (9 coefficients, 3 channels)
    # Y0 = 1/(2*sqrt(pi))
    # Y1 = sqrt(3/(4*pi)) * n
    # Y2 = sqrt(15/(4*pi)) * (n_x*n_y, n_y*n_z, n_z*(2*n_z - n_x - n_y), n_x*n_z, n_x*n_x - n_y*n_y)
    # Irradiance = sum(sh_k * Y_k(normal))
    pi = math.pi
    y = [0.0] * 9
    nx, ny, nz = normal
    y[0] = 1.0 / (2.0 * math.sqrt(pi))
    y[1] = math.sqrt(3.0 / (4.0 * pi)) * nx
    y[2] = math.sqrt(3.0 / (4.0 * pi)) * ny
    y[3] = math.sqrt(3.0 / (4.0 * pi)) * nz
    y[4] = math.sqrt(15.0 / (4.0 * pi)) * nx * ny
    y[5] = math.sqrt(15.0 / (4.0 * pi)) * ny * nz
    y[6] = math.sqrt(15.0 / (4.0 * pi)) * nz * (2.0 * nz - nx - ny)
    y[7] = math.sqrt(15.0 / (4.0 * pi)) * nx * nz
    y[8] = math.sqrt(15.0 / (4.0 * pi)) * (nx * nx - ny * ny)
    
    irradiance = [0.0, 0.0, 0.0]
    for c in range(3):
        for k in range(9):
            irradiance[c] += sh_coeffs[k * 3 + c] * y[k]
    return tuple(irradiance)
```

### Parameters
- **Spherical Harmonics Order**: Second-order (9 coefficients) [RH01].
- **Quantization**: Half-precision (16-bit) for SH coefficients. Further quantization to 28 bytes per probe is possible but has little influence on results [S1].
- **Clamping Threshold**: $10^{-6}$ for interpolation weights [S1].
- **Grid Size**: Example: $22^3$ probe grid for the Sponza scene, consuming 35.5 MB in standard DDGI [S1].

### Edge Cases and Pitfalls
- **Light Leaks**: Occur if ray-traced visibility is not employed and cosine fall-off is insufficient or incorrectly applied [S1].
- **Contact Shadows**: Probe-based methods fail to capture contact shadows entirely due to the low resolution of the irradiance volume [S1].
- **Probe Placement Errors**: Erroneous probe placement (e.g., inside geometry) combined with runtime updates can produce artifacts like dark smudges on surfaces [S1].
- **Memory Overhead**: Standard DDGI uses 1168 bytes per probe, with 1024 bytes dedicated to depth buffers. Using ray-traced visibility reduces this to 54 bytes per probe for SH data [S1].

## Operating characteristics
- **Time Complexity**: Query time is dominated by simple trilinear interpolation of SH coefficients, which is highly competitive [S1]. Ray-traced visibility adds significant runtime cost compared to depth-texture lookups [S1].
- **Memory Complexity**: Scales cubically with spatial discretization [S1]. For a $22^3$ grid, it consumes 35.5 MB (standard DDGI) or significantly less if depth buffers are removed (54 bytes/probe) [S1].
- **Performance**: Simple probe-based methods have a performance edge over neural implicit methods (like NIV) if optimized, due to the simplicity of trilinear interpolation [S1]. NIV requires input encoding evaluation and matrix multiplications, incurring a cost of around 1 ms on an RTX 4090 [S1].
- **Strengths**: Simple implementation, predictable memory usage, fast query times if visibility is cheap [S1].
- **Limitations**: Poor representation of high-frequency details (contact shadows), light leaks without careful visibility handling, cubic memory scaling [S1].
- **When to Use**: When memory budget is fixed and scene geometry is static or slowly changing, and high-frequency shadow details are not critical [S1].
- **When Not to Use**: When contact shadows are essential, or when adaptive resolution is needed to handle complex lighting details efficiently [S1].

## Background
- **Origin**: Based on the Dynamic Diffuse Global Illumination (DDGI) method [MGNM19].
- **Development**: Implemented as a baseline for comparison with Neural Irradiance Volumes (NIV) [S1]. The specific variant described uses ray-traced visibility instead of depth textures to improve quality and reduce memory overhead [S1].
- **Usage**: Used in real-time rendering systems for global illumination approximation, particularly in games and interactive applications where path tracing is too expensive [S1].

## Sources
[S1] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
