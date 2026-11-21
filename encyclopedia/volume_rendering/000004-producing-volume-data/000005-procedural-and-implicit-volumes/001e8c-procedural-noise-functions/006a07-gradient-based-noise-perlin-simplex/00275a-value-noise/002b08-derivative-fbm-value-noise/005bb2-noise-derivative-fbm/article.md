# Noise Derivative FBM

Also called: FBM with noise derivatives, Tweaked Fractal Brownian Motion using noise derivatives

## Overview
Noise Derivative FBM is a procedural terrain generation technique that utilizes the analytic derivatives of value noise, rather than the noise values themselves, to modulate the accumulation of height in a Fractal Brownian Motion (FBM) loop [S1, unofficial]. It solves the problem of repetitive, uniform terrain structures common in standard FBM by accentuating steep regions and creating flat plateaux, resulting in more detailed and physically plausible landscapes [S1, unofficial]. It differs from standard FBM by using the gradient magnitude to dampen height accumulation, and from basic Derivative FBM by explicitly returning the derivative vector alongside the scalar noise value for use in lighting and erosion calculations [S1, unofficial].

## How it works
**Inputs:**
- `uTexture`: A `sampler2D` uniform holding a pre-generated grayscale noise texture (typically 256x256 pixels) [S1, unofficial].
- `p`: The input 2D coordinate vector for the terrain sample [S1, unofficial].
- `m`: A 2x2 rotation matrix used to rotate the sampling coordinate between octaves [S1, unofficial].

**Outputs:**
- A scalar height value representing the terrain elevation at `p` [S1, unofficial].

**Data Structures:**
- The noise texture is accessed via bilinear interpolation (implicit in `textureLod` with LOD 0) [S1, unofficial].
- Coordinates are normalized to `[0, 1]` by dividing integer grid coordinates by 256.0 [S1, unofficial].

**Parameters:**
- Initial scale factor `b = 2.5` [S1, unofficial].
- Initial height accumulator `a = 0.0` [S1, unofficial].
- Initial derivative accumulator `d = vec2(0.0)` [S1, unofficial].
- Coordinate scale `scl = 2.75` [S1, unofficial].
- Coordinate offset multiplier `0.06` [S1, unofficial].
- Octave count: 8 [S1, unofficial].
- Height decay factor: `0.85` per octave [S1, unofficial].
- Amplitude decay factor: `-0.4` per octave [S1, unofficial].
- Final height multiplier: `3.0` [S1, unofficial].

**Steps:**
1.  **Noise Sampling (`noised` function):**
    For a given 2D coordinate `x`:
    1.  Calculate integer part `p = floor(x)` and fractional part `f = fract(x)` [S1, unofficial].
    2.  Calculate interpolation weights `u = f * f * (3.0 - 2.0 * f)` [S1, unofficial].
    3.  Sample the four corners of the noise texture at normalized coordinates `(p + offset) / 256.0` with LOD 0 to get values `a`, `b`, `c`, `d` [S1, unofficial]:
        -   `a = textureLod(uTexture, (p + vec2(0.0, 0.0)) / 256.0, 0).x`
        -   `b = textureLod(uTexture, (p + vec2(1.0, 0.0)) / 256.0, 0).x`
        -   `c = textureLod(uTexture, (p + vec2(0.0, 1.0)) / 256.0, 0).x`
        -   `d = textureLod(uTexture, (p + vec2(1.0, 1.0)) / 256.0, 0).x`
    4.  Calculate the noise value using trilinear interpolation (bilinear in 2D):
        $$ \text{noiseValue} = a + (b - a)u_x + (c - a)u_y + (a - b - c + d)u_x u_y $$
    5.  Calculate the noise derivative vector:
        $$ \text{noiseDerivative} = 6.0 \cdot f \cdot (1.0 - f) \cdot (\text{vec2}(b - a, c - a) + (a - b - c + d) \cdot \text{vec2}(u_y, u_x)) $$
    6.  Return `vec3(noiseValue, noiseDerivative)` [S1, unofficial].

2.  **FBM Accumulation (`terrain` function):**
    1.  Initialize `p1 = p * 0.06`, `a = 0.0`, `b = 2.5`, `d = vec2(0.0)`, `scl = 2.75` [S1, unofficial].
    2.  Loop 8 times:
        a.  Call `noised(p1)` to get `n` (where `n.x` is value, `n.yz` is derivative) [S1, unofficial].
        b.  Accumulate derivatives: `d += n.yz` [S1, unofficial].
        c.  Update height: `a += b * n.x / (dot(d, d) + 1.0)` [S1, unofficial].
        d.  Update amplitude: `b *= -0.4` [S1, unofficial].
        e.  Update height scale: `a *= 0.85` [S1, unofficial].
        f.  Transform coordinate: `p1 = m * p1 * scl` [S1, unofficial].
    3.  Return `a * 3.0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def noised(u_texture, x):
    """
    Samples a grayscale noise texture and returns value and derivative.
    u_texture: 2D array of floats, size 256x256.
    x: vec2 input coordinate.
    Returns: (value, deriv_x, deriv_y)
    """
    # 1. Integer and fractional parts
    p_x = math.floor(x[0])
    p_y = math.floor(x[1])
    f_x = x[0] - p_x
    f_y = x[1] - p_y
    
    # 2. Interpolation weights
    u_x = f_x * f_x * (3.0 - 2.0 * f_x)
    u_y = f_y * f_y * (3.0 - 2.0 * f_y)
    
    # 3. Sample corners (normalized by 256.0)
    # Assuming u_texture is accessed via bilinear interpolation.
    # For exact implementation from array, bilinear interpolation is required.
    # Here we assume a helper texture_sample that does bilinear interpolation on the 256x256 grid.
    
    # Corner coordinates in grid space
    p00 = (p_x + 0.0) / 256.0
    p10 = (p_x + 1.0) / 256.0
    p01 = (p_y + 0.0) / 256.0
    p11 = (p_y + 1.0) / 256.0
    
    # Note: The source uses (p + vec2) / 256.0. 
    # p is integer part. The texture is 256x256.
    # We need bilinear interpolation of the texture at these normalized coords.
    
    a = texture_sample(u_texture, p00, p00) # vec2(0.0, 0.0) offset
    b = texture_sample(u_texture, p10, p00) # vec2(1.0, 0.0) offset
    c = texture_sample(u_texture, p00, p01) # vec2(0.0, 1.0) offset
    d = texture_sample(u_texture, p10, p01) # vec2(1.0, 1.0) offset
    
    # 4. Noise Value
    noise_value = a + (b - a) * u_x + (c - a) * u_y + (a - b - c + d) * u_x * u_y
    
    # 5. Noise Derivative
    # 6.0 * f * (1.0 - f) * (vec2(b - a, c - a) + (a - b - c + d) * u.yx)
    deriv_x = 6.0 * f_x * (1.0 - f_x) * ((b - a) + (a - b - c + d) * u_y)
    deriv_y = 6.0 * f_y * (1.0 - f_y) * ((c - a) + (a - b - c + d) * u_x)
    
    return (noise_value, deriv_x, deriv_y)

def texture_sample(tex, u, v):
    """
    Bilinear interpolation of a 256x256 texture.
    u, v are in [0, 1].
    """
    # Map to texel coordinates
    tex_u = u * 256.0
    tex_v = v * 256.0
    
    # Integer parts
    i_u = int(math.floor(tex_u))
    i_v = int(math.floor(tex_v))
    
    # Fractional parts
    f_u = tex_u - i_u
    f_v = tex_v - i_v
    
    # Wrap or clamp? Standard GLSL textureLod with LOD 0 usually wraps or clamps depending on sampler state.
    # The source implies standard wrapping or clamping. We assume clamping to [0, 255] for safety in array access.
    i_u = max(0, min(255, i_u))
    i_v = max(0, min(255, i_v))
    
    # Neighbors
    i_u1 = min(255, i_u + 1)
    i_v1 = min(255, i_v + 1)
    
    # Fetch values
    v00 = tex[i_v][i_u]
    v10 = tex[i_v][i_u1]
    v01 = tex[i_v1][i_u]
    v11 = tex[i_v1][i_u1]
    
    # Bilinear interp
    top = v00 + (v10 - v00) * f_u
    bottom = v01 + (v11 - v01) * f_u
    return top + (bottom - top) * f_v

def terrain(p, u_texture, m):
    """
    Generates terrain height using Noise Derivative FBM.
    p: vec2 input coordinate.
    m: 2x2 rotation matrix [[m00, m01], [m10, m11]].
    """
    p1 = [p[0] * 0.06, p[1] * 0.06]
    a = 0.0
    b = 2.5
    d_x = 0.0
    d_y = 0.0
    scl = 2.75
    
    for i in range(8):
        n_val, n_dx, n_dy = noised(u_texture, p1)
        
        # Accumulate derivatives
        d_x += n_dx
        d_y += n_dy
        
        # Update height
        dot_dd = d_x * d_x + d_y * d_y
        a += b * n_val / (dot_dd + 1.0)
        
        # Update amplitude
        b *= -0.4
        
        # Update height scale
        a *= 0.85
        
        # Transform coordinate
        # p1 = m * p1 * scl
        new_x = (m[0][0] * p1[0] + m[0][1] * p1[1]) * scl
        new_y = (m[1][0] * p1[0] + m[1][1] * p1[1]) * scl
        p1 = [new_x, new_y]
        
    return a * 3.0
```

**Edge Cases and Pitfalls:**
-   **Precision:** The derivative calculation involves `6.0 * f * (1.0 - f)`. At `f=0` or `f=1`, the derivative is zero. Floating point precision errors near boundaries may cause slight discontinuities if not handled carefully, though bilinear interpolation smooths the value [S1, unofficial].
-   **Boundary Conditions:** The texture sampling assumes coordinates are normalized to `[0, 1]`. If `p` results in coordinates outside the texture bounds, wrapping or clamping behavior must be defined (standard GLSL `sampler2D` behavior applies) [S1, unofficial].
-   **Degenerate Input:** If `p` is constant, the terrain will be flat. If `m` is singular, the coordinate transformation may collapse dimensions [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(O)$ where $O$ is the number of octaves (8 in the source). Each octave requires one noise texture lookup with bilinear interpolation and constant arithmetic operations [S1, unofficial].
-   **Memory Complexity:** $O(1)$ for the algorithm state, plus $O(N^2)$ for the noise texture storage (256x256 floats) [S1, unofficial].
-   **Performance:** The method is computationally more expensive than standard FBM due to the derivative calculation and the additional texture sampling overhead (though still constant per octave) [S1, unofficial].
-   **Strengths:** Produces terrains with steeper regions and flat plateaux, reducing visual repetition and increasing detail compared to standard FBM [S1, unofficial].
-   **Limitations:** Requires a pre-generated noise texture. The derivative accumulation can lead to numerical instability if not clamped, though the `dot(d,d) + 1.0` denominator mitigates this [S1, unofficial].
-   **When to use:** When realistic, varied terrain with distinct steep/flat features is desired, and performance allows for derivative calculations [S1, unofficial].
-   **When not to use:** When simple, uniform noise is sufficient, or when memory bandwidth is extremely constrained (due to texture sampling) [S1, unofficial].

## Background
The technique was popularized in the context of raymarching and procedural terrain generation by Maxime Heckel in his blog post "Painting with Math: A Gentle Study of Raymarching" [S1, unofficial]. It builds upon the concept of Fractal Brownian Motion (FBM) and value noise, extending it by leveraging the analytic gradient of the noise function to influence the height accumulation [S1, unofficial].

## Sources
[S1] Painting with Math: A Gentle Study of Raymarching, https://blog.maximeheckel.com/posts/painting-with-math-a-gentle-study-of-raymarching/, fetched 2026-10-27 (unofficial)
