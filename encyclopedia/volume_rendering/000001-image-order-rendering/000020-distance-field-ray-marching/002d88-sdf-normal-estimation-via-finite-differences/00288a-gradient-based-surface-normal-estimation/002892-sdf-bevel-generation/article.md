# SDF Bevel Generation
Also called: Distance Field Bevel, SDF Surface Normal Estimation
## Overview
SDF Bevel Generation estimates 3D surface normals for lighting by sampling a Signed Distance Function (SDF) height field at four cardinal offsets, computing finite differences, and applying bevel shaping parameters [S1, unofficial]. It differs from standard gradient estimation by introducing explicit control over edge rounding and thickness, making it suitable for stylized or physically plausible beveled geometry in ray marching [S1, unofficial]. It builds on [SDF Normal Estimation via Finite Differences](4619-sdf-normal-estimation-via-finite-differences.md) [G].
## How it works
### Inputs and Outputs
- **Input**: SDF texture or volume, UV coordinates, texture dimensions (`textureWidth`, `textureHeight`), and bevel parameters (`_Bevel`, `_BevelOffset`, `_BevelWidth`, `_BevelRoundness`, `_BevelClamp`, `_GradientScale`, `_BevelAmount`) [S1, unofficial].
- **Output**: Surface normal vector `n` [S1, unofficial].

### Data Structures
- **SDF Sample Array `h`**: A vector of 4 floats storing the sampled SDF values at the cardinal offsets [S1, unofficial].
  - `h.x`: SDF at `uv - delta.xz`
  - `h.y`: SDF at `uv + delta.xz`
  - `h.z`: SDF at `uv - delta.zy`
  - `h.w`: SDF at `uv + delta.zy`

### Parameters
- `_Bevel` (float, default 0.5): Enables bevel generation when > 0 [S1, unofficial].
- `_BevelOffset` (float, default 0): Shifts the sampled height field values [S1, unofficial].
- `_BevelWidth` (float, default 0): Controls bevel thickness, clamped to min 0.01 [S1, unofficial].
- `_BevelRoundness` (float, default 0): Controls edge rounding via sine interpolation [S1, unofficial].
- `_BevelClamp` (float, default 0): Clamps bevel height to prevent over-extrusion artifacts [S1, unofficial].
- `_GradientScale` (float): Scaling factor for the final gradient magnitude [S1, unofficial].
- `_BevelAmount` (float): Multiplier for gradient scaling [S1, unofficial].

### Steps
1. **Compute Delta**: Calculate the step size in texture space: $\delta = (1.0/\text{textureWidth}, 1.0/\text{textureHeight}, 0.0)$ [S1, unofficial].
2. **Sample SDF**: Sample the SDF at four cardinal offsets relative to the current UV `uv`: $h_x = \text{SampleSDF}(uv - \delta.xz)$, $h_y = \text{SampleSDF}(uv + \delta.xz)$, $h_z = \text{SampleSDF}(uv - \delta.zy)$, $h_w = \text{SampleSDF}(uv + \delta.zy)$ [S1, unofficial]. Store in array `h = [h_x, h_y, h_z, h_w]` [S1, unofficial].
3. **Apply Offset**: Add `_BevelOffset` to all values in `h` [S1, unofficial].
4. **Normalize Width**: Divide all values in `h` by $\max(0.01, \text{_BevelWidth})$ [S1, unofficial].
5. **Apply Roundness**: If `_BevelRoundness` > 0, apply sine interpolation to smooth edges: $h_i = \text{lerp}(h_i, \sin(h_i \cdot \pi/2), \text{_BevelRoundness})$ for each $i \in \{x, y, z, w\}$ [S1, unofficial].
6. **Clamp**: Clamp the values in `h` using `_BevelClamp` to prevent over-extrusion [S1, unofficial].
7. **Scale Gradient**: Scale the height differences by $\text{_BevelAmount} \times \text{_BevelWidth} \times \text{_GradientScale} \times -2.0$ [S1, unofficial].
8. **Compute Tangents**: Calculate two tangent vectors $v_a$ and $v_b$ from the finite differences: $v_a = \text{normalize}(-1, 0, h_y - h_x)$, $v_b = \text{normalize}(0, 1, h_w - h_z)$ [S1, unofficial].
9. **Compute Normal**: Calculate the cross product of $v_a$ and $v_b$, then apply a flip factor $f$ based on face orientation: $n = (v_a \times v_b) \cdot f$, where $f = (1, 1, -1)$ for front faces and $f = (1, 1, 1)$ for back faces [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    l = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/l, v[1]/l, v[2]/l) if l > 1e-8 else (0.0, 0.0, 0.0)

def lerp(a, b, t):
    return a + (b - a) * t

def clamp(v, mn, mx):
    return max(mn, min(mx, v))

def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def sdf_bevel_generation(uv, texture_width, texture_height, sdf_sample, params, is_front):
    bevel_offset = params.get('_BevelOffset', 0.0)
    bevel_width = params.get('_BevelWidth', 0.0)
    bevel_roundness = params.get('_BevelRoundness', 0.0)
    bevel_clamp = params.get('_BevelClamp', 0.0)
    gradient_scale = params.get('_GradientScale', 1.0)
    bevel_amount = params.get('_BevelAmount', 1.0)

    delta_x = 1.0 / texture_width
    delta_y = 1.0 / texture_height

    h_x = sdf_sample(uv[0] - delta_x, uv[1])
    h_y = sdf_sample(uv[0] + delta_x, uv[1])
    h_z = sdf_sample(uv[0], uv[1] - delta_y)
    h_w = sdf_sample(uv[0], uv[1] + delta_y)
    h = [h_x, h_y, h_z, h_w]

    h = [val + bevel_offset for val in h]

    effective_width = max(0.01, bevel_width)
    h = [val / effective_width for val in h]

    if bevel_roundness > 0:
        h = [lerp(val, math.sin(val * math.pi / 2.0), bevel_roundness) for val in h]

    if bevel_clamp > 0:
        h = [clamp(val, -bevel_clamp, bevel_clamp) for val in h]

    scale_factor = bevel_amount * bevel_width * gradient_scale * -2.0
    h = [val * scale_factor for val in h]

    dy_dx = h[1] - h[0]
    dy_dy = h[3] - h[2]

    va = normalize((-1.0, 0.0, dy_dx))
    vb = normalize((0.0, 1.0, dy_dy))

    f = (1.0, 1.0, -1.0) if is_front else (1.0, 1.0, 1.0)

    cross_prod = cross(va, vb)
    normal = (cross_prod[0]*f[0], cross_prod[1]*f[1], cross_prod[2]*f[2])

    return normal
```

### Edge Cases and Pitfalls
- **Zero Width**: `_BevelWidth` is clamped to a minimum of 0.01 to prevent division by zero [S1, unofficial].
- **Backface Orientation**: The normal must be flipped for backfaces to ensure correct lighting orientation [S1, unofficial].
- **Precision**: Floating-point precision issues may arise with very small `_BevelWidth` values, mitigated by the clamping [G].
- **Degenerate Input**: If the SDF is constant, the gradient will be zero, resulting in an undefined normal [G].

## Operating characteristics
- **Time Complexity**: O(1) per pixel, as it involves a fixed number of texture samples and arithmetic operations [G].
- **Memory Complexity**: O(1) auxiliary memory [G].
- **Strengths**: Provides explicit control over edge appearance (bevel width and roundness), which is useful for stylized rendering [G].
- **Limitations**: Relies on texture sampling, which may introduce aliasing if the texture resolution is low. The quality depends on the accuracy of the SDF texture [G].
- **When to Use**: When stylized beveled edges are desired in ray-marched SDF rendering [G].
- **When Not to Use**: When high-precision, smooth normals are required without stylization, or when the SDF is not available as a texture [G].

## Background
This method originates from the PNANOVDB implementation, which provides C and GLSL primitives for vector operations and coordinate handling in distance-field rendering [S1, unofficial]. It is part of a suite of techniques for normal estimation in distance-field based rendering [G].

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
