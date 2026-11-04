# Linear Distance Fog

Also called: Depth Cueing (Linear), Distance Fog, Light Diminishing, Linear Fog

## Overview
Linear distance fog is a depth-cueing technique that attenuates surface brightness linearly with view distance, fading colors toward black (or a specified floor color) to simulate atmospheric density or limited visibility [S1, S2, unofficial]. Unlike [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md), which models physical light absorption via an exponential decay, linear fog is a geometric approximation that is computationally cheaper and easier to implement in fixed-point or integer arithmetic [S1, unofficial]. It serves as a simpler alternative to [Fog Blending with Clouds](4123-fog-blending-with-clouds.md) or [Height-based Fog with Directional In-scatter](4400-height-based-fog-with-directional-in-scatter.md) when volumetric scattering effects are not required [S1, S2, unofficial].

## How it works
The method operates by calculating a scalar "shade" factor based on the distance from the camera to the surface, then multiplying the surface's color components by this factor.

### Inputs and Outputs
- **Inputs**:
  - `dist`: The perpendicular distance from the camera to the surface (in world units or cells).
  - `col`: The original surface color, typically in 24-bit RGB (0x00RRGGBB) or 16-bit RGB565 format.
  - `FOG_DIST`: The distance at which the surface becomes fully dark (black).
  - `FOG_MIN`: The minimum brightness floor (0–256) to prevent surfaces from vanishing completely.
- **Outputs**:
  - `fogged_col`: The attenuated color value.

### Data Structures
No complex data structures are required for the core per-pixel shading. However, for floor and ceiling rendering in raycasting engines, precomputed row tables are used to avoid per-pixel distance calculations [S1, unofficial].
- `CEIL_ROW`: An array of size $H$ (screen height) storing pre-fogged ceiling colors.
- `FLOOR_ROW`: An array of size $H$ storing pre-fogged floor colors.

### Formula
The shade factor is calculated as:
$$
shade = \max(FOG_{MIN}, 256 - \lfloor \frac{dist \times 256}{FOG_{DIST}} \rfloor)
$$
Where:
- $dist$ is the view distance.
- $FOG_{DIST}$ is the full fade distance.
- $FOG_{MIN}$ is the brightness floor (clamped between 0 and 256).
- The result `shade` is an integer in the range $[FOG_{MIN}, 256]$.

For floor and ceiling pixels in a raycaster, the distance depends only on the screen row $y$ relative to the horizon (screen height $H$):
$$
d_{row} = \frac{H / 2}{|y - H / 2|}
$$
If $|y - H / 2| < 1$, the distance is considered infinite, and the shade is clamped to $FOG_{MIN}$ [S1, unofficial].

### Parameters
- **FOG_DIST**: Distance of full fade. Typical value: 10 cells [S1, unofficial].
- **FOG_MIN**: Brightness floor. Typical value: 36 (range 0–256) [S1, unofficial].

### Steps and Pseudocode
The implementation differs slightly based on color format. Below is the standard 24-bit RGB implementation using integer arithmetic, followed by the RGB565 variant.

#### Standard 24-bit RGB (Wolfenstein 3D style)
This method uses integer arithmetic to avoid floating-point costs [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fog_shade(dist, fog_dist=10, fog_min=36):
    # Calculate linear attenuation
    # Integer division: (dist * 256) // fog_dist
    attenuation = (dist * 256) // fog_dist
    shade = 256 - attenuation
    # Clamp to brightness floor
    if shade < fog_min:
        shade = fog_min
    return shade

def shade_color_24bit(col, shade):
    # Unpack RGB components from 0x00RRGGBB
    r = (col & 0xFF)
    g = (col >> 8) & 0xFF
    b = (col >> 16) & 0xFF
    
    # Multiply by shade (0-256) and shift right by 8 (divide by 256)
    r_out = (r * shade) >> 8
    g_out = (g * shade) >> 8
    b_out = (b * shade) >> 8
    
    # Clamp outputs to 0-255 (though shade logic usually keeps them valid)
    r_out = max(0, min(255, r_out))
    g_out = max(0, min(255, g_out))
    b_out = max(0, min(255, b_out))
    
    # Repack into 0x00RRGGBB
    return (0xFF000000 | (b_out << 16) | (g_out << 8) | r_out)

def apply_fog_24bit(dist, col, fog_dist=10, fog_min=36):
    shade = fog_shade(dist, fog_dist, fog_min)
    return shade_color_24bit(col, shade)
```

#### RGB565 Variant (ESP8266/LCD style)
This variant unpacks 16-bit RGB565, applies a linear factor, and repacks. Note the specific multipliers (8, 4, 8) used to adjust for display driver gamma/intensity [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_fog_rgb565(color_16bit, dist, max_dist=12.0):
    # Unpack RGB565
    # R: 5 bits (10-14), G: 6 bits (5-10), B: 5 bits (0-4)
    r = (color_16bit >> 11) & 0x1F
    g = (color_16bit >> 5) & 0x3F
    b = color_16bit & 0x1F
    
    # Calculate factor
    if dist > max_dist:
        return 0x0000 # Black
    
    factor = 1.0 - (dist / max_dist)
    
    # Apply factor and scale for display driver
    # Note: Multipliers 8, 4, 8 are specific to the hardware driver [S2, unofficial]
    r_out = int(r * factor * 8)
    g_out = int(g * factor * 4)
    b_out = int(b * factor * 8)
    
    # Clamp to valid RGB565 ranges
    r_out = max(0, min(31, r_out))
    g_out = max(0, min(63, g_out))
    b_out = max(0, min(31, b_out))
    
    # Repack RGB565
    return (r_out << 11) | (g_out << 5) | b_out
```

### Edge Cases and Pitfalls
1. **Integer Overflow**: In 24-bit implementations, multiplying an 8-bit color component by a shade value up to 256 can result in a 16-bit intermediate value. Ensure the arithmetic supports this before shifting [S1, unofficial].
2. **Color Distortion in RGB565**: Multiplying the raw 16-bit integer directly by the distance factor causes color channels to bleed into each other ("swimming" colors). Bitwise unpacking is mandatory [S2, unofficial].
3. **Floor/Ceiling Singularity**: At the horizon line ($y = H/2$), the distance formula $d_{row}$ approaches infinity. The code must handle $|y - H/2| < 1$ explicitly to avoid division by zero or extreme values [S1, unofficial].
4. **Side Darkening**: In raycasting, wall shading often includes a directional component (e.g., side walls are darker). This can be folded into the fog shade calculation: `final_shade = (shade * side_factor) >> 8` [S1, unofficial].

## Subtopics

### Linear Distance Fog — Depth Cueing (Linear), Distance Fog, Light Diminishing, Linear Fog
This is the standard implementation found in Wolfenstein 3D-style engines. It uses 24-bit color, integer arithmetic, and a brightness floor to ensure readability. It is chosen when performance is critical and the engine uses fixed-point math. It differs from the RGB565 variant by its color depth and arithmetic method [S1, unofficial].

### Linear Distance Fog — RGB565 Lantern Effect
This variant is optimized for 16-bit LCD displays (e.g., ESP8266 projects). It uses floating-point factors and specific scaling multipliers (8, 4, 8) to compensate for display gamma. It is chosen for embedded systems with limited color depth and specific hardware drivers. It differs from the standard variant by its color format and the necessity of bitwise unpacking to prevent distortion [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per pixel. The calculation involves a few multiplications, shifts, and comparisons.
- **Memory Complexity**: $O(H)$ for floor/ceiling row tables if precomputed, where $H$ is screen height. Otherwise, $O(1)$ [S1, unofficial].
- **Performance**: Extremely fast. The 24-bit integer version requires no floating-point unit, making it suitable for CPUs without FPU support [S1, unofficial].
- **Strengths**: Simple to implement; computationally inexpensive; provides immediate depth perception [S1, unofficial].
- **Limitations**: Does not model physical light scattering; linear fade can look unnatural compared to exponential decay; brightness floor may make distant objects too visible or too dim depending on tuning [S1, S2, unofficial].
- **When to use**: Real-time raycasting engines, embedded graphics, or when a simple depth cue is sufficient without volumetric effects.
- **When not to use**: When physically accurate atmospheric scattering is required (use [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md)) or when fog interacts with light sources (use [Height-based Fog with Directional In-scatter](4400-height-based-fog-with-directional-in-scatter.md)) [S1, S2, unofficial].

## Background
Linear distance fog, also known as "light diminishing," was introduced in later engines of the Wolfenstein 3D family to improve atmosphere, as the original game lacked depth cues beyond object size [S1, unofficial]. It was a cheap way to add depth perception to 2.5D raycasted environments. A similar technique was later adapted for embedded systems using RGB565 displays, where it was referred to as a "lantern effect" to simulate flashlight visibility [S2, unofficial].

## Sources
[S1] A minimal Wolfenstein 3D raycasting engine, https://4rknova.com/blog/2026/07/06/wolfenstein-3d-raycasting, fetched 2026-10-24 (unofficial)
[S2] Как Я Заставил Esp8266 Рисовать 3d, Стрелять И Летать: Полный ..., https://habr.com/ru/articles/1097380/, fetched 2026-10-26 (unofficial)
