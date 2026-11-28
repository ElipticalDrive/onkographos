# Exponential Color Transfer Function
Also called: Exponential Opacity, Exponential Opacity Transfer, Simple Color Map
## Overview
The Exponential Color Transfer Function maps scalar volume intensity values to RGBA color and opacity, using an exponential function for alpha and linear interpolation for color [S1, unofficial][S2, unofficial]. It is a hardcoded transfer function that computes color and opacity directly from scalar intensity without lookup tables [S1, unofficial][S2, unofficial].
## How it works
**Inputs and Outputs:**
- Input: scalar intensity $I$ normalized to $[0, 1]$.
- Output: RGBA color vector $\mathbf{c}$.

**Data Structures:**
- No lookup table is stored; the function is hardcoded as a static computation [S1, unofficial][S2, unofficial].
- Constants: `high` = `vec3(1.0, 1.0, 1.0)` (white) [S2, unofficial]. `low` = `vec3(0.0, 0.0, 0.0)` (black) [S2, unofficial].

**Steps:**
1. Compute opacity $\alpha$ using the exponential formula.
2. Compute color $\mathbf{c}_{rgb}$ via linear interpolation between `low` and `high` weighted by $I$.
3. Return $\mathbf{c} = (\mathbf{c}_{rgb}, \alpha)$.
4. In the raymarching loop, sample intensity, apply the transfer function, and composite the result using front-to-back alpha blending.

**Formulas:**
$$ \alpha = \frac{e^{I} - 1}{e^1 - 1} $$
where $I$ is the input intensity [S1, unofficial][S2, unofficial].
$$ \mathbf{c}_{rgb} = I \cdot \text{high} + (1.0 - I) \cdot \text{low} $$
Since `low` is black and `high` is white, this simplifies to $\mathbf{c}_{rgb} = I \cdot \text{high}$ [S1, unofficial][S2, unofficial].
Front-to-back blending: $\mathbf{C}_{out} = \alpha_c \mathbf{C}_c + (1 - \alpha_c) \alpha_{acc} \mathbf{C}_{acc}$ and $\alpha_{acc} = \alpha_c + (1 - \alpha_c) \alpha_{acc}$ [S1, unofficial].

**Parameters:**
- The exponential base is fixed at $e$ [S1, unofficial][S2, unofficial].
- The normalization factor is $e^1 - 1$ [S1, unofficial][S2, unofficial].
- Color endpoints are fixed at black and white [S2, unofficial].

**Edge Cases:**
- If $I=0$, $\alpha = 0$ and color is black [S1, unofficial][S2, unofficial].
- If $I=1$, $\alpha = 1$ and color is white [S1, unofficial][S2, unofficial].
- Precision: For $I$ very close to 0, $e^I - 1$ may suffer from floating-point cancellation [G]. Use `math.expm1(I)` if available for better precision.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def exponential_color_transfer(intensity: float) -> tuple:
    high = (1.0, 1.0, 1.0)
    low = (0.0, 0.0, 0.0)
    exp_1 = math.exp(1.0)
    alpha = (math.exp(intensity) - 1.0) / (exp_1 - 1.0)
    r = intensity * high[0] + (1.0 - intensity) * low[0]
    g = intensity * high[1] + (1.0 - intensity) * low[1]
    b = intensity * high[2] + (1.0 - intensity) * low[2]
    return (r, g, b, alpha)

def raycast_volume(volume_data, ray_origin, ray_direction, step_length, background_color=(0.0, 0.0, 0.0), gamma=2.2):
    dir_len = math.sqrt(ray_direction[0]**2 + ray_direction[1]**2 + ray_direction[2]**2)
    step_vector = (ray_direction[0]/dir_len * step_length,
                   ray_direction[1]/dir_len * step_length,
                   ray_direction[2]/dir_len * step_length)
    ray_length = dir_len
    position = ray_origin
    accumulated_color = list(background_color)
    accumulated_alpha = 0.0

    while ray_length > 0:
        intensity = volume_data[position]
        c_rgb, c_a = exponential_color_transfer(intensity)
        accumulated_color[0] = c_a * c_rgb[0] + (1 - c_a) * accumulated_alpha * accumulated_color[0]
        accumulated_color[1] = c_a * c_rgb[1] + (1 - c_a) * accumulated_alpha * accumulated_color[1]
        accumulated_color[2] = c_a * c_rgb[2] + (1 - c_a) * accumulated_alpha * accumulated_color[2]
        accumulated_alpha = c_a + (1 - c_a) * accumulated_alpha
        ray_length -= step_length
        position = (position[0] + step_vector[0], position[1] + step_vector[1], position[2] + step_vector[2])

    bg = background_color
    final_r = accumulated_alpha * accumulated_color[0] + (1 - accumulated_alpha) * bg[0]
    final_g = accumulated_alpha * accumulated_color[1] + (1 - accumulated_alpha) * bg[1]
    final_b = accumulated_alpha * accumulated_color[2] + (1 - accumulated_alpha) * bg[2]
    final_a = 1.0

    final_r = final_r ** (1.0 / gamma)
    final_g = final_g ** (1.0 / gamma)
    final_b = final_b ** (1.0 / gamma)

    return (final_r, final_g, final_b, final_a)
```
## Subtopics

### [Tent Alpha Transfer Function](3031-tent-alpha-transfer-function.md)
Uses a piecewise linear tent function for alpha, allowing for selective visibility of specific intensity ranges. Choose when you need to highlight a specific band of values rather than a monotonic fade-in.

### [2D Transfer Function Volume Rendering](1295-2d-transfer-function-volume-rendering.md)
Maps two scalar values (e.g., density and gradient magnitude) to color/opacity. Choose when scalar data alone is insufficient to distinguish features.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample [S1, unofficial][S2, unofficial].
- **Memory Complexity:** $O(1)$ [S1, unofficial][S2, unofficial].
- **Strengths:** No memory overhead for tables [S1, unofficial][S2, unofficial].
- **Limitations:** Limited flexibility; only one parameter (intensity) is used; opacity curve is fixed and cannot be adjusted for different datasets [S1, unofficial][S2, unofficial].
- **When to use:** Simple visualization of scalar data [S1, unofficial][S2, unofficial].
- **When not to use:** When specific intensity ranges need to be highlighted or when multi-dimensional data classification is required [S1, unofficial][S2, unofficial].

## Background
The method originates from simple GPU-based volume raycasting implementations, such as those by Martino Pilia and Jimpelton, which prioritize simplicity and performance over complex classification [S1, unofficial][S2, unofficial].

## Sources
[S1] jimpelton/subvol, https://github.com/jimpelton/subvol, fetched 2026-10-26 (unofficial)
[S2] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
