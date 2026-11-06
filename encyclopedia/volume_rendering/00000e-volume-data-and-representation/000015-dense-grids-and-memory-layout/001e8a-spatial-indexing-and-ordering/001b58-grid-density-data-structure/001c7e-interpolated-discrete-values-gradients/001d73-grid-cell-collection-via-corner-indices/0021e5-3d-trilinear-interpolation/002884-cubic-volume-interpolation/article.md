# Cubic Volume Interpolation
Also called: Cubic Trilinear Interpolation, Trilinear Interpolation with Cubic Weights

## Overview
Cubic Volume Interpolation extends standard 3D trilinear interpolation by replacing linear mixture weights with cubic polynomial weights for each spatial axis, providing smoother value estimation in dense grid volume rendering [S1, unofficial]. It differs from its parent [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by using higher-order basis functions to reduce aliasing artifacts, while avoiding the gradient computation overhead of [Trilinear Gradient Interpolation](3470-trilinear-gradient-interpolation.md) [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D texture/grid `dataTex` (typically `float`), and a query coordinate `texCoord` ($x, y, z$) in texture space.
- Output: A single interpolated scalar value.

**Data Structures:**
- `texCoord`: A 3D vector $(x, y, z)$ representing the position within the volume.
- `mixX`, `mixY`, `mixZ`: Temporary 3D vectors returned by `cubicInterpMixture`, containing fractional offsets and weight sums.

**Algorithm Steps:**
1. For each axis ($x, y, z$), compute the cubic mixture weights and offsets by calling `cubicInterpMixture` on the coordinate component. Let the results be `mixX`, `mixY`, and `mixZ`.
2. Sample the 8 corner values of the grid cell surrounding `texCoord`. The texture coordinates for these corners are derived from the components of `mixX`, `mixY`, and `mixZ`. Specifically, the texture lookup uses `mixX.x` or `mixX.y` for the X-axis, `mixY.x` or `mixY.y` for the Y-axis, and `mixZ.x` or `mixZ.y` for the Z-axis.
   - Let $V_{ijk}$ denote the value at corner indices determined by the mixture components.
3. Perform interpolation along the X-axis for the two Z-slices and two Y-positions:
   - $v_{Y0Z0} = \text{lerp}(V_{000}, V_{100}, \text{mixX}.z)$
   - $v_{Y1Z0} = \text{lerp}(V_{010}, V_{110}, \text{mixX}.z)$
   - $v_{Y0Z1} = \text{lerp}(V_{001}, V_{101}, \text{mixX}.z)$
   - $v_{Y1Z1} = \text{lerp}(V_{011}, V_{111}, \text{mixX}.z)$
   *(Note: The source code uses `mixX.z` as the interpolation factor for the X-axis lerp, which corresponds to the weight sum component returned by `cubicInterpMixture`)*.
4. Interpolate along the Y-axis for the two Z-slices:
   - $v_{Z0} = \text{lerp}(v_{Y0Z0}, v_{Y1Z0}, \text{mixY}.z)$
   - $v_{Z1} = \text{lerp}(v_{Y0Z1}, v_{Y1Z1}, \text{mixY}.z)$
5. Interpolate along the Z-axis to produce the final value:
   - $result = \text{lerp}(v_{Z0}, v_{Z1}, \text{mixZ}.z)$

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
where $i = \lfloor x \rfloor$ and $a = x - i$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_interpolate(tex_coord: Vector3f, sample_texture: Callable[[Vector3f], float]) -> float:
    """
    Performs cubic volume interpolation using cubic mixture weights.
    """
    # Step 1: Compute mixture weights for each axis
    mix_x = cubic_interp_mixture(tex_coord.x)
    mix_y = cubic_interp_mixture(tex_coord.y)
    mix_z = cubic_interp_mixture(tex_coord.z)

    # Step 2: Sample the 8 corners of the grid cell
    # The texture coordinates are constructed from the mixture components
    # mix.x and mix.y are the fractional offsets, mix.z is the weight sum
    v_000 = sample_texture(Vector3f(mix_x.x, mix_y.x, mix_z.x))
    v_100 = sample_texture(Vector3f(mix_x.y, mix_y.x, mix_z.x))
    v_010 = sample_texture(Vector3f(mix_x.x, mix_y.y, mix_z.x))
    v_110 = sample_texture(Vector3f(mix_x.y, mix_y.y, mix_z.x))
    v_001 = sample_texture(Vector3f(mix_x.x, mix_y.x, mix_z.y))
    v_101 = sample_texture(Vector3f(mix_x.y, mix_y.x, mix_z.y))
    v_011 = sample_texture(Vector3f(mix_x.x, mix_y.y, mix_z.y))
    v_111 = sample_texture(Vector3f(mix_x.y, mix_y.y, mix_z.y))

    # Step 3: Interpolate along X axis for Y=0, Z=0
    v_y0z0 = lerp(v_000, v_100, mix_x.z)
    # Interpolate along X axis for Y=1, Z=0
    v_y1z0 = lerp(v_010, v_110, mix_x.z)
    # Interpolate along X axis for Y=0, Z=1
    v_y0z1 = lerp(v_001, v_101, mix_x.z)
    # Interpolate along X axis for Y=1, Z=1
    v_y1z1 = lerp(v_011, v_111, mix_x.z)

    # Step 4: Interpolate along Y axis for Z=0
    v_z0 = lerp(v_y0z0, v_y1z0, mix_y.z)
    # Interpolate along Y axis for Z=1
    v_z1 = lerp(v_y0z1, v_y1z1, mix_y.z)

    # Step 5: Interpolate along Z axis
    return lerp(v_z0, v_z1, mix_z.z)

def cubic_interp_mixture(x: float) -> Vector3f:
    """
    Computes cubic interpolation weights and fractional offsets for a 1D coordinate.
    Returns (offset0, offset1, weight_sum).
    """
    i = floor(x)
    a = x - i
    
    a2 = a * a
    a3 = a * a2
    
    w0 = -a3 + (2 * a2) - a
    w1 = a3 - (2 * a2) + 1
    w2 = -a3 + a2 + a
    w3 = a3 - a2
    
    # Calculate fractional offsets for the two texels involved
    offset0 = i - (w0 / (w0 + w1))
    offset1 = i + 1 + (w3 / (w2 + w3))
    
    # Weight sum for normalization/interpolation factor
    weight_sum = w2 + w3
    
    return Vector3f(offset0, offset1, weight_sum)

def lerp(a: float, b: float, t: float) -> float:
    return a + t * (b - a)
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The source code implies clamping or wrapping is handled by the texture sampler (`tex3D`), but the mixture weights calculation assumes the coordinate is within valid bounds. If `x` is outside the grid, the texture lookup behavior depends on the hardware/driver.
- **Precision:** The calculation of `w0/(w0+w1)` and `w3/(w2+w3)` involves division. If `w0+w1` or `w2+w3` are near zero, precision issues may arise. However, for standard cubic interpolation weights, these sums are typically non-zero within the valid range.
- **Normalization:** The `weight_sum` component is used directly as the interpolation factor `t` in `lerp`. This assumes the weights are normalized such that the sum of the two relevant weights equals 1, or that the `lerp` function handles the scaling. In the provided source, `mixX.z` is used as `t`, implying `w2+w3` acts as the effective weight for the interpolation step.

## Subtopics

### [Multi-linear Isosurface Coefficient Calculation](5607-multi-linear-isosurface-coefficient-calculation.md)
This method calculates the coefficients of the multi-variate polynomial that represents a multi-linearly interpolated level set (LS) function within a grid cell. It solves the problem of efficiently converting corner LS values into polynomial coefficients for root finding, optimizing the number of arithmetic operations compared to straightforward expansion. It is distinguished from its parent, Cubic Volume Interpolation, by its specific focus on the algebraic structure of the level set equation for intersection solving rather than general value estimation.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, requiring 8 texture lookups and a fixed number of arithmetic operations (additions, multiplications).
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Performance:** Slower than linear trilinear interpolation due to the additional weight calculations and 8 texture samples instead of 8 (same count, but more complex addressing/weighting). However, it provides smoother results.
- **Strengths:** Reduces aliasing and provides smoother gradients compared to linear interpolation without the cost of gradient-based methods.
- **Limitations:** Higher computational cost than linear interpolation. May introduce slight overshoots (Gibbs phenomenon) near sharp discontinuities, though less severe than higher-order polynomials.
- **When to use:** When visual quality is prioritized over raw performance, and linear interpolation artifacts are noticeable.
- **When not to use:** When performance is critical and linear interpolation is sufficient, or when gradient information is explicitly required (use [Trilinear Gradient Interpolation](3470-trilinear-gradient-interpolation.md)).

## Background
Cubic Volume Interpolation is implemented in CUDA-based volume rendering pipelines, as seen in the `dtu-2010-cuda-volume-rendering` repository [S1, unofficial]. It extends the concept of trilinear interpolation by using cubic basis functions to determine the interpolation weights, aiming to improve visual smoothness in ray-casted volumes.

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
