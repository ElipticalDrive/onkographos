# Linear Distance Fog

Also called: Depth Cueing (Linear), Distance Fog, Light Diminishing, Linear Fog

## Overview
Linear distance fog is a depth-cueing technique that attenuates surface brightness linearly with view distance, fading colors toward black with a brightness floor to simulate atmospheric density or limited visibility [S1, S2, unofficial]. Unlike [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md), which models physical light absorption via exponential decay, linear fog is a geometric approximation implemented with integer or fixed-point arithmetic for performance [S1, unofficial].

## How it works
**Inputs**: `dist` (perpendicular distance from camera to surface in cells or world units), `col` (surface color in 24-bit RGB or 16-bit RGB565), `FOG_DIST` (full fade distance), `FOG_MIN` (brightness floor).
**Outputs**: `fogged_col` (attenuated color).
**Data Structures**: Precomputed row tables for floor/ceiling in raycasters: `CEIL_ROW` and `FLOOR_ROW` arrays of size $H$ (screen height) [S1, unofficial].

**Formula**:
$$
shade = \max(FOG_{MIN}, 256 - \lfloor \frac{dist \times 256}{FOG_{DIST}} \rfloor)
$$
Where $dist$ is the view distance, $FOG_{DIST}$ is the full fade distance, and $FOG_{MIN}$ is the brightness floor clamped between 0 and 256 [S1, unofficial]. The result `shade` is an integer in the range $[FOG_{MIN}, 256]$ [S1, unofficial].

For floor and ceiling pixels in a raycaster, the distance depends only on the screen row $y$ relative to the horizon (screen height $H$):
$$
d_{row} = \frac{H / 2}{|y - H / 2|}
$$
If $|y - H / 2| < 1$, the distance is considered infinite and the shade is clamped to $FOG_{MIN}$ [S1, unofficial].

**Parameters**: $FOG_{DIST} = 10$ cells, $FOG_{MIN} = 36$ [S1, unofficial].

**Steps and Pseudocode**:
The implementation uses integer arithmetic to avoid floating-point costs [S1, unofficial]. Wall shading folds directional side-darkening into the per-column fog shade [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fog_shade(dist, fog_dist=10, fog_min=36):
    attenuation = (dist * 256) // fog_dist
    shade = 256 - attenuation
    if shade < fog_min:
        shade = fog_min
    return shade

def shade_color_24bit(col, shade):
    r = ((col & 0xff) * shade) >> 8
    g = (((col >> 8) & 0xff) * shade) >> 8
    b = (((col >> 16) & 0xff) * shade) >> 8
    return 0xff000000 | (b << 16) | (g << 8) | r

def apply_fog_24bit(dist, col, fog_dist=10, fog_min=36, side=0):
    shade = fog_shade(dist, fog_dist, fog_min)
    # Fold y-side directional darkening into the per-column factor [S1, unofficial]
    if side == 1:
        shade = (shade * 140) >> 8
    return shade_color_24bit(col, shade)

# Precompute floor/ceiling row tables once at load [S1, unofficial]
CEIL_ROW = [0] * FB_H
FLOOR_ROW = [0] * FB_H
for y in range(FB_H):
    dy = abs(y - FB_H / 2)
    shade = FOG_MIN if dy < 1 else fog_shade((FB_H / 2) / dy)
    CEIL_ROW[y] = shade_color_24bit(CEIL_COLOR, shade)
    FLOOR_ROW[y] = shade_color_24bit(FLOOR_COLOR, shade)
```

**RGB565 Variant**:
This variant unpacks 16-bit RGB565, applies a linear factor, and repacks [S2, unofficial]. Bitwise unpacking is mandatory because multiplying the raw 16-bit integer directly causes color channels to bleed into each other ("swimming" colors) [S2, unofficial]. The multipliers (8, 4, 8) adjust for the specific display driver's gamma or intensity [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_fog_rgb565(color_16bit, dist, max_dist=12.0):
    r = (color_16bit >> 11) & 0x1F
    g = (color_16bit >> 5) & 0x3F
    b = color_16bit & 0x1F
    if dist > max_dist:
        return 0
    factor = 1.0 - (dist / max_dist)
    # Multipliers 8, 4, 8 compensate for display driver intensity [S2, unofficial]
    r_out = int(r * factor * 8)
    g_out = int(g * factor * 4)
    b_out = int(b * factor * 8)
    return (r_out << 11) | (g_out << 5) | b_out
```

**Edge Cases and Pitfalls**:
1. **Integer Overflow**: Multiplying an 8-bit color component by a shade value up to 256 produces a 16-bit intermediate value; arithmetic must support this before shifting [S1, unofficial].
2. **Color Distortion in RGB565**: Direct multiplication of the packed 16-bit integer causes channel bleeding; bitwise unpacking is required [S2, unofficial].
3. **Floor/Ceiling Singularity**: At the horizon line ($y = H/2$), $d_{row}$ approaches infinity; the code must handle $|y - H/2| < 1$ explicitly to avoid division by zero [S1, unofficial].
4. **Side Darkening**: Wall shading includes a directional component that folds into the fog shade via `(shade * 140) >> 8` for side walls [S1, unofficial].

## Subtopics

### Linear Distance Fog — Depth Cueing (Linear), Distance Fog, Light Diminishing, Linear Fog
Standard 24-bit implementation using integer arithmetic and a brightness floor to ensure readability [S1, unofficial]. Chosen for performance-critical raycasting engines using fixed-point math [S1, unofficial].

### Linear Distance Fog — RGB565 Lantern Effect
Variant optimized for 16-bit LCD displays using floating-point factors and hardware-specific scaling multipliers (8, 4, 8) to compensate for display driver intensity [S2, unofficial]. Chosen for embedded systems with limited color depth [S2, unofficial]. Requires bitwise unpacking to prevent color distortion [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per pixel [G].
- **Memory Complexity**: $O(H)$ for precomputed floor/ceiling row tables, otherwise $O(1)$ [S1, unofficial].
- **Measured Performance**: Extremely fast; the 24-bit integer version requires no floating-point unit, making it suitable for CPUs without FPU support [S1, unofficial]. On embedded hardware, it runs efficiently on ESP8266-driven 16-bit LCDs [S2, unofficial].
- **Scaling**: Not given in the sources.
- **Strengths**: Computationally inexpensive and cheap enough to add atmosphere to raycasted environments [S1, unofficial].
- **Limitations**: Does not model physical light scattering; linear fade is a geometric approximation rather than a physical model [S1, unofficial].
- **When to use**: Real-time raycasting or embedded graphics where performance is critical [S1, S2, unofficial].
- **When not to use**: When physically accurate atmospheric scattering or volumetric light interaction is required [G].

## Background
Origin: Introduced in later engines of the Wolfenstein 3D family to improve atmosphere, as the original game shipped with no depth cue except object size [S1, unofficial]. Adapted for ESP8266 embedded projects as a "lantern effect" to simulate flashlight visibility on RGB565 displays [S2, unofficial].

## Sources
[S1] A minimal Wolfenstein 3D raycasting engine, https://4rknova.com/blog/2026/07/06/wolfenstein-3d-raycasting, fetched 2026-10-24 (unofficial)
[S2] Как Я Заставил Esp8266 Рисовать 3d, Стрелять И Летать: Полный ..., https://habr.com/ru/articles/1097380/, fetched 2026-10-26 (unofficial)
