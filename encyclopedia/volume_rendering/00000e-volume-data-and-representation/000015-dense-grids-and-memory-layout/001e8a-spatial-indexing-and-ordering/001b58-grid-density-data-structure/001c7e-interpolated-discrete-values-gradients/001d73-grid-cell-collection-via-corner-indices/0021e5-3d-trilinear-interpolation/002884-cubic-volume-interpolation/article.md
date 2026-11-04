# Cubic Volume Interpolation
Also called: Cubic Trilinear Interpolation, Trilinear Interpolation with Cubic Weights

## Overview
Cubic Volume Interpolation extends standard trilinear interpolation by using cubic mixture weights for each axis to sample a 3D texture at 8 corners defined by those weights [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D coordinate `texCoord` (float3) within the texture space, and a 3D texture `dataTex` containing the volume data [S1, unofficial].
- Output: A single interpolated float value representing the density or color at `texCoord` [S1, unofficial].

**Data Structures:**
- `dataTex`: A 3D texture object used for sampling [S1, unofficial].
- `mixX`, `mixY`, `mixZ`: Temporary `float3` variables storing the fractional offsets for the two texels and the sum of weights for normalization for each axis [S1, unofficial].

**Steps:**
1. **Calculate Cubic Mixture Weights:** For each axis (X, Y, Z), compute the cubic interpolation weights and fractional offsets using the `cubicInterpMixture` function [S1, unofficial].
2. **Sample Texture Corners:** Sample the 3D texture at the 8 corners defined by the mixture weights [S1, unofficial]. The coordinates for these samples are derived from `mixX`, `mixY`, and `mixZ` [S1, unofficial].
3. **Interpolate along X Axis:** For each of the two Z-slices (defined by `mixZ.x` and `mixZ.y`), interpolate along the X axis between the corresponding Y-slices [S1, unofficial].
   - For the first Z-slice (`mixZ.x`):
     - `vY0Z0 = lerp(vX0Y0Z0, vX1Y0Z0, mixX.z)` [S1, unofficial]
     - `vY1Z0 = lerp(vX0Y1Z0, vX1Y1Z0, mixX.z)` [S1, unofficial]
   - For the second Z-slice (`mixZ.y`):
     - `vY0Z1 = lerp(vX0Y0Z1, vX1Y0Z1, mixX.z)` [S1, unofficial]
     - `vY1Z1 = lerp(vX0Y1Z1, vX1Y1Z1, mixX.z)` [S1, unofficial]
4. **Interpolate along Y Axis:** For each Z-slice, interpolate along the Y axis between the results of the X interpolation [S1, unofficial].
   - `vZ0 = lerp(vY0Z0, vY1Y0Z0, mixY.z)` [S1, unofficial]
   - `vZ1 = lerp(vY0Z1, vY1Y1Z1, mixY.z)` [S1, unofficial]
5. **Interpolate along Z Axis:** Finally, interpolate along the Z axis between `vZ0` and `vZ1` using `mixZ.z` to produce the final value [S1, unofficial].

**Formulas:**
The cubic weights are calculated as follows, where `a` is the fractional part of the coordinate [S1, unofficial]:
$$ a2 = a \times a $$
$$ a3 = a \times a2 $$
$$ w0 = -a3 + (2 \times a2) - a $$
$$ w1 = a3 - (2 \times a2) + 1 $$
$$ w2 = -a3 + a2 + a $$
$$ w3 = a3 - a2 $$

The `cubicInterpMixture` function returns a `float3` containing [S1, unofficial]:
- First component: `i - (w0 / (w0 + w1))` (fractional offset for the first texel) [S1, unofficial]
- Second component: `i + 1 + (w3 / (w2 + w3))` (fractional offset for the second texel) [S1, unofficial]
- Third component: `w2 + w3` (sum of weights for normalization) [S1, unofficial]

**Parameters:**
- `triCubic`: A boolean flag that determines whether to use cubic interpolation (`true`) or trilinear interpolation (`false`) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Boundary conditions: The implementation checks if the texture coordinates are within the valid range `[0, dataDim - 1]` [S1, unofficial]. If out of bounds, it returns a default `outVal` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lerp(a, b, t):
    return a + t * (b - a)

def texture_sample(tex, x, y, z):
    # Standard 3D texture lookup operation
    return tex.sample(x, y, z)

def cubic_interp_mixture(coord):
    i = math.floor(coord)
    a = coord - i
    a2 = a * a
    a3 = a * a2
    w0 = -a3 + (2 * a2) - a
    w1 = a3 - (2 * a2) + 1
    w2 = -a3 + a2 + a
    w3 = a3 - a2
    offset1 = i - (w0 / (w0 + w1))
    offset2 = i + 1 + (w3 / (w2 + w3))
    weight_sum = w2 + w3
    return (offset1, offset2, weight_sum)

def cubic_interpolate(tex_coord, data_tex, data_dim, out_val):
    # Boundary check
    if (tex_coord.x < 0 or tex_coord.x > data_dim.x - 1 or
        tex_coord.y < 0 or tex_coord.y > data_dim.y - 1 or
        tex_coord.z < 0 or tex_coord.z > data_dim.z - 1):
        return out_val

    mix_x = cubic_interp_mixture(tex_coord.x)
    mix_y = cubic_interp_mixture(tex_coord.y)
    mix_z = cubic_interp_mixture(tex_coord.z)

    # Sample the 8 corners
    v_x0_y0_z0 = texture_sample(data_tex, mix_x[0], mix_y[0], mix_z[0])
    v_x0_y1_z0 = texture_sample(data_tex, mix_x[0], mix_y[1], mix_z[0])
    v_x0_y0_z1 = texture_sample(data_tex, mix_x[0], mix_y[0], mix_z[1])
    v_x0_y1_z1 = texture_sample(data_tex, mix_x[0], mix_y[1], mix_z[1])
    v_x1_y0_z0 = texture_sample(data_tex, mix_x[1], mix_y[0], mix_z[0])
    v_x1_y1_z0 = texture_sample(data_tex, mix_x[1], mix_y[1], mix_z[0])
    v_x1_y0_z1 = texture_sample(data_tex, mix_x[1], mix_y[0], mix_z[1])
    v_x1_y1_z1 = texture_sample(data_tex, mix_x[1], mix_y[1], mix_z[1])

    # Interpolate along X
    v_y0_z0 = lerp(v_x0_y0_z0, v_x1_y0_z0, mix_x[2])
    v_y1_z0 = lerp(v_x0_y1_z0, v_x1_y1_z0, mix_x[2])
    v_y0_z1 = lerp(v_x0_y0_z1, v_x1_y0_z1, mix_x[2])
    v_y1_z1 = lerp(v_x0_y1_z1, v_x1_y1_z1, mix_x[2])

    # Interpolate along Y
    v_z0 = lerp(v_y0_z0, v_y1_y0_z0, mix_y[2])
    v_z1 = lerp(v_y0_z1, v_y1_y1_z1, mix_y[2])

    # Interpolate along Z
    return lerp(v_z0, v_z1, mix_z[2])
```

## Subtopics

### Cubic Interpolation Weights
Computes cubic interpolation weights and fractional offsets for a 1D coordinate, used for higher-order volume interpolation [S1, unofficial]. It calculates the integer part `i` and fractional part `a` of the coordinate, then derives weights `w0` through `w3` using cubic polynomials [S1, unofficial]. The function returns a `float3` containing the fractional offsets for the two texels involved in the interpolation and the sum of weights for normalization [S1, unofficial]. This is a helper function for the main cubic interpolation method [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Measured Performance:** Not given in the sources [S1, unofficial].
- **Scaling:** Not given in the sources [S1, unofficial].
- **Strengths:** Not given in the sources [S1, unofficial].
- **Limitations:** Not given in the sources [S1, unofficial].
- **When to use:** Not given in the sources [S1, unofficial].
- **When not to use:** Not given in the sources [S1, unofficial].

## Background
Originating from a CUDA volume rendering implementation, this method extends standard trilinear interpolation to provide smoother results in 3D texture sampling [S1, unofficial].

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
