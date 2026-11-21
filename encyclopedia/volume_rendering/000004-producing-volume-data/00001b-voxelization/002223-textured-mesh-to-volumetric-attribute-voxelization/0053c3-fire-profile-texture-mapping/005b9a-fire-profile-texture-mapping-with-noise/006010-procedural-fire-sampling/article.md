# Procedural Fire Sampling

Also called: sampleFire

## Overview
Procedural Fire Sampling generates a volumetric fire appearance by mapping a 3D location to a 2D fire profile texture, applying time-based vertical scrolling and turbulence-driven vertical offsets to simulate rising flames [S1, unofficial]. It differs from static volume rendering by integrating noise turbulence directly into the texture coordinate lookup to create animated, organic fire behavior without requiring pre-computed 3D noise volumes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `loc` (normalized 3D location, typically 0.0–1.0 range), `scale` (vec4 uniform controlling spatial and temporal scaling).
- Output: `vec4` representing color and alpha of the fire at the given location.
- Global Uniforms: `time` (float, elapsed time), `fireProfile` (sampler2D, the fire profile texture), `magnatude` (float constant).

**Data Structures:**
- `loc`: `vec3` representing the normalized position within the fire volume.
- `scale`: `vec4` where `xyz` scale the noise space dimensions and `w` controls the vertical scroll speed over time.
- `fireProfile`: A 2D texture where the x-axis represents radius (0.0 to 1.0) and the y-axis represents height (0.0 to 1.0).

**Steps:**
1. **Coordinate Normalization:** Convert the x and z components of `loc` from the [0.0, 1.0] range to the [-1.0, 1.0] range to center the fire profile around the origin [S1, unofficial].
   $$ loc.xz = loc.xz \times 2.0 - 1.0 $$
2. **Cylindrical Mapping:** Convert the normalized 3D location into 2D texture coordinates `st` for the fire profile. The x-coordinate of `st` is the radial distance from the center axis, and the y-coordinate is the height [S1, unofficial].
   $$ st = \left( \sqrt{ \text{dot}(loc.xz, loc.xz) }, loc.y \right) $$
3. **Time-Based Scrolling:** Adjust the y-component of `loc` by subtracting the product of `time` and the scroll speed (`scale.w`) to create an upward scrolling effect [S1, unofficial]. Then, scale the entire `loc` vector by `scale.xyz` to adjust the noise frequency [S1, unofficial].
   $$ loc.y = loc.y - time \times scale.w $$
   $$ loc = loc \times scale.xyz $$
4. **Turbulence Offset Calculation:** Calculate a vertical offset based on the square root of the height (`st.y`), a magnitude constant, and the turbulence value at the current location. The square root scaling ensures stability at the base and volatility at the top [S1, unofficial].
   $$ offset = \sqrt{st.y} \times magnatude \times turbulence(loc) $$
   Where `turbulence(loc)` is defined as the sum of absolute values of multiple octaves of noise:
   $$ turbulence(loc) = \sum_{i=0}^{3} |noise(loc \times freq_i)| \times amp_i $$
   With `freq_0 = 1.0`, `amp_0 = 1.0`, `lacunarity = 2.0`, `gain = 0.5`, and `octaves = 4`.
5. **Apply Offset:** Add the calculated offset to the y-component of the texture coordinates [S1, unofficial].
   $$ st.y = st.y + offset $$
6. **Boundary Check (Top):** If `st.y` exceeds 1.0, return black with full alpha to prevent out-of-bounds sampling artifacts [S1, unofficial].
   $$ \text{if } st.y > 1.0 \text{ return } (0, 0, 0, 1) $$
7. **Texture Sampling:** Sample the fire profile texture using the modified coordinates `st` [S1, unofficial].
   $$ result = \text{texture2D}(fireProfile, st) $$
8. **Boundary Fade (Bottom):** If `st.y` is less than 0.1, linearly fade the result to hide slice clipping at the base of the fire [S1, unofficial].
   $$ \text{if } st.y < 0.1 \text{ result} = result \times \frac{st.y}{0.1} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sampleFire(loc: vec3, scale: vec4, time: float) -> vec4:
    # Step 1: Normalize xz to [-1.0, 1.0]
    loc.xz = loc.xz * 2.0 - 1.0
    
    # Step 2: Convert to (radius, height)
    st_x = sqrt(dot(loc.xz, loc.xz))
    st_y = loc.y
    st = vec2(st_x, st_y)
    
    # Step 3: Time-based scrolling and scaling
    loc.y = loc.y - time * scale.w
    loc = loc * scale.xyz
    
    # Step 4: Calculate turbulence offset
    # turbulence(loc) is defined as sum of 4 octaves of noise
    # For implementation, assume a noise function noise3D(vec3) is available
    sum_turbulence = 0.0
    freq = 1.0
    amp = 1.0
    for i in range(4):
        sum_turbulence += abs(noise3D(loc * freq)) * amp
        freq *= 2.0
        amp *= 0.5
    
    magnatude = 1.3
    offset = sqrt(st_y) * magnatude * sum_turbulence
    
    # Step 5: Apply offset
    st_y = st_y + offset
    
    # Step 6: Boundary check (Top)
    if st_y > 1.0:
        return vec4(0, 0, 0, 1)
    
    # Step 7: Sample texture
    # Assuming texture2D is a standard 2D texture lookup
    result = texture2D(fireProfile, vec2(st_x, st_y))
    
    # Step 8: Boundary fade (Bottom)
    if st_y < 0.1:
        result = result * (st_y / 0.1)
        
    return result
```

**Parameters:**
- `magnatude`: Float constant, typically 1.3 [S1, unofficial]. Controls the magnitude of the vertical noise offset.
- `scale`: Vec4 uniform. `xyz` components scale the noise space; `w` component controls vertical scroll speed [S1, unofficial].
- `time`: Float uniform, elapsed time in seconds [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Out-of-bounds Sampling:** If `st.y > 1.0`, direct sampling may cause artifacts. The method explicitly returns black with alpha 1 in this case [S1, unofficial].
- **Base Clipping:** If `st.y < 0.1`, the fire may appear clipped. A linear fade `result *= st.y / 0.1` is applied to smooth the transition to the base [S1, unofficial].
- **Precision:** The use of `sqrt` and noise functions requires careful handling of floating-point precision, especially near the boundaries [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: O(1) per sample, but involves a constant number of noise evaluations (4 octaves).
- Memory Complexity: O(1) additional memory, plus the memory for the `fireProfile` texture.

**Measured Performance:**
- Performance depends on the efficiency of the `noise3D` function and the texture lookup. On modern GPUs, this is generally fast due to hardware-accelerated texture sampling and noise generation [S1, unofficial].

**Scaling:**
- Scaling the fire size is achieved by adjusting the `scale` uniform, which affects the noise frequency and scroll speed [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Efficient, procedural, and animated without pre-computation. Easy to integrate with existing fire profile textures [S1, unofficial].
- Limitations: Relies on a 2D fire profile texture, which may limit the variety of fire shapes. The noise function implementation can vary in quality and performance [S1, unofficial].

**When to Use:**
- Use when a procedural, animated fire effect is needed with minimal memory overhead and no pre-computed 3D noise volumes [S1, unofficial].

**When Not to Use:**
- Avoid when highly detailed, physically accurate fire simulation is required, or when the fire shape needs to be complex and non-cylindrical [S1, unofficial].

## Background
Procedural Fire Sampling is part of the VolumetricFire library, created by yomotsu [S1, unofficial]. It is designed for real-time rendering in WebGL environments, leveraging the efficiency of texture mapping and noise functions to create convincing fire effects [S1, unofficial].

## Sources
[S1] yomotsu/VolumetricFire, https://github.com/yomotsu/VolumetricFire, fetched 2026-10-27 (unofficial)
