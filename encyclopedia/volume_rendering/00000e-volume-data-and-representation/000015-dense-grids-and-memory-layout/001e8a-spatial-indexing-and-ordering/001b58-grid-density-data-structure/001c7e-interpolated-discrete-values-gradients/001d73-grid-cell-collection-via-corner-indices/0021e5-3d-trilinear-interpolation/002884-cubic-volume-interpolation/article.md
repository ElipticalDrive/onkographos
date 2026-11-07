# Cubic Volume Interpolation
Also called: Cubic Trilinear Interpolation, Trilinear Interpolation with Cubic Weights

## Overview
Cubic Volume Interpolation extends standard 3D trilinear interpolation by replacing linear mixture weights with cubic polynomial weights for each spatial axis [S1, unofficial]. It differs from its parent [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by using cubic mixture weights to determine texture sampling coordinates and interpolation factors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D texture or grid `dataTex` (typically `float`) and a query coordinate `texCoord` ($x, y, z$) in texture space [S1, unofficial].
- Output: A single interpolated scalar value [S1, unofficial].

**Data Structures:**
- `mixX`, `mixY`, `mixZ`: Temporary 3D vectors returned by `cubicInterpMixture`, where each vector contains three components: `(offset0, offset1, weight_sum)` [S1, unofficial].

**Algorithm Steps:**
1. For each axis ($x, y, z$), compute the cubic mixture weights and offsets by calling `cubicInterpMixture` on the coordinate component [S1, unofficial].
2. Sample the 8 corner values of the grid cell surrounding `texCoord` using the components of `mixX`, `mixY`, and `mixZ` as texture coordinates [S1, unofficial].
3. Interpolate along the X-axis for the two Y-positions and two Z-slices using `mixX.z` as the interpolation factor [S1, unofficial].
4. Interpolate along the Y-axis for the two Z-slices using `mixY.z` as the interpolation factor [S1, unofficial].
5. Interpolate along the Z-axis using `mixZ.z` as the interpolation factor to produce the final value [S1, unofficial].

**Formulas:**
The core weight calculation for a 1D coordinate $a$ (fractional part of the position) is:
$$
\begin{aligned}
a_2 &= a \cdot a \\
a_3 &= a \cdot a_2 \\
w_0 &= -a_3 + 2a_2 - a \\
w_1 &= a_3 - 2a_2 + 1 \\
w_2 &= -a_3 + a_2 + a \\
w_3 &= a_3 - a_2
\end{aligned}
$$
The mixture vector components are derived as:
$$
\begin{aligned}
\text{offset}_0 &= i - \frac{w_0}{w_0 + w_1} \\
\text{offset}_1 &= i + 1 + \frac{w_3}{w_2 + w_3} \\
\text{weight\_sum} &= w_2 + w_3
\end{aligned}
$$
where $i = \lfloor x \rfloor$ and $a = x - i$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

def cubic_interpolate(tex_coord: Vec3, sample_texture) -> float:
    mix_x = cubic_interp_mixture(tex_coord.x)
    mix_y = cubic_interp_mixture(tex_coord.y)
    mix_z = cubic_interp_mixture(tex_coord.z)

    v_000 = sample_texture(Vec3(mix_x.x, mix_y.x, mix_z.x))
    v_100 = sample_texture(Vec3(mix_x.y, mix_y.x, mix_z.x))
    v_010 = sample_texture(Vec3(mix_x.x, mix_y.y, mix_z.x))
    v_110 = sample_texture(Vec3(mix_x.y, mix_y.y, mix_z.x))
    v_001 = sample_texture(Vec3(mix_x.x, mix_y.x, mix_z.y))
    v_101 = sample_texture(Vec3(mix_x.y, mix_y.x, mix_z.y))
    v_011 = sample_texture(Vec3(mix_x.x, mix_y.y, mix_z.y))
    v_111 = sample_texture(Vec3(mix_x.y, mix_y.y, mix_z.y))

    v_y0z0 = lerp(v_000, v_100, mix_x.z)
    v_y1z0 = lerp(v_010, v_110, mix_x.z)
    v_y0z1 = lerp(v_001, v_101, mix_x.z)
    v_y1z1 = lerp(v_011, v_111, mix_x.z)

    v_z0 = lerp(v_y0z0, v_y1z0, mix_y.z)
    v_z1 = lerp(v_y0z1, v_y1z1, mix_y.z)

    return lerp(v_z0, v_z1, mix_z.z)

def cubic_interp_mixture(x: float) -> Vec3:
    i = math.floor(x)
    a = x - i
    a2 = a * a
    a3 = a * a2
    w0 = -a3 + (2 * a2) - a
    w1 = a3 - (2 * a2) + 1
    w2 = -a3 + a2 + a
    w3 = a3 - a2
    offset0 = i - (w0 / (w0 + w1))
    offset1 = i + 1 + (w3 / (w2 + w3))
    weight_sum = w2 + w3
    return Vec3(offset0, offset1, weight_sum)

def lerp(a: float, b: float, t: float) -> float:
    return a + t * (b - a)
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The source code checks if coordinates are outside the grid bounds (`texCoord.x < 0` or `texCoord.x > dataDim.width - 1`, etc.) and returns a default value if true [S1, unofficial].
- **Precision:** The calculation of `w0/(w0+w1)` and `w3/(w2+w3)` involves division [S1, unofficial]. Not given in the sources.

## Subtopics

### [Multi-linear Isosurface Coefficient Calculation](5607-multi-linear-isosurface-coefficient-calculation.md)
This method calculates the coefficients of the multi-variate polynomial that represents a multi-linearly interpolated level set (LS) function within a grid cell. It solves the problem of efficiently converting corner LS values into polynomial coefficients for root finding, optimizing the number of arithmetic operations compared to straightforward expansion. It is distinguished from its parent, Cubic Volume Interpolation, by its specific focus on the algebraic structure of the level set equation for intersection solving rather than general value estimation.

### Cubic Interpolation Weights
Computes cubic interpolation weights and fractional offsets for a 1D coordinate, used for higher-order volume interpolation [S1, unofficial]. It computes the integer part $i = \lfloor x \rfloor$ and fractional part $a = x - i$, then calculates powers $a_2 = a^2$ and $a_3 = a^3$ [S1, unofficial]. The weights are $w_0 = -a_3 + 2a_2 - a$, $w_1 = a_3 - 2a_2 + 1$, $w_2 = -a_3 + a_2 + a$, $w_3 = a_3 - a_2$ [S1, unofficial]. It returns a 3D vector containing the fractional offsets for the two texels involved in the interpolation and the sum of weights for normalization [S1, unofficial].

## Operating characteristics
- **Time and memory complexity:** Not given in the sources.
- **Measured performance on specific hardware or resolutions:** Not given in the sources.
- **Scaling behavior:** Not given in the sources.
- **Strengths and limitations:** Not given in the sources.
- **When to use or not use:** Not given in the sources.

## Background
Cubic Volume Interpolation is implemented in CUDA-based volume rendering pipelines, as seen in the `dtu-2010-cuda-volume-rendering` repository [S1, unofficial]. It extends the concept of trilinear interpolation by using cubic basis functions to determine the interpolation weights, aiming to improve visual smoothness in ray-casted volumes [S1, unofficial].

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
