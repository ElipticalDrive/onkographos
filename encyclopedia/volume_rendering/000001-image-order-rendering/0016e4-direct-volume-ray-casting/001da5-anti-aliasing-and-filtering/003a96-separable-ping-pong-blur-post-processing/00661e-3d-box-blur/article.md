# 3D Box Blur
Also called: blurFragShader

## Overview
A post-processing blur filter that samples a 3D volume at the 8 corners of a small box centered on the current coordinate and averages them [S1, unofficial]. It serves as a lightweight anti-aliasing and smoothing step in direct volume ray casting pipelines, distinct from its siblings by using a uniform 8-tap kernel rather than weighted or exponential decay functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D texture (`intensityVol`) containing volume data, and uniform parameters defining the sampling geometry.
*   **Output:** A blurred color value (`FragColor`) for the current fragment.

**Data Structures and Uniforms:**
The method relies on the following GLSL uniforms:
*   `highp sampler3D intensityVol`: The 3D volume texture to sample.
*   `float coordZ`: The Z-coordinate of the current sample point in volume space.
*   `float dX`: The half-width of the box in the X dimension.
*   `float dY`: The half-width of the box in the Y dimension.
*   `float dZ`: The half-width of the box in the Z dimension.

**Algorithm Steps:**
1.  **Construct Coordinate:** Form the 3D sampling coordinate `vx` using the fragment's texture coordinates (`TexCoord.xy`) and the provided Z-coordinate.
    $$ vx = (TexCoord.x, TexCoord.y, coordZ) $$
2.  **Define Offsets:** The box corners are defined by the combinations of offsets $\pm dX, \pm dY, \pm dZ$. There are 8 corners.
3.  **Sample Corners:** Sample the `intensityVol` at each of the 8 corners:
    $$ (vx.x \pm dX, vx.y \pm dY, vx.z \pm dZ) $$
4.  **Average:** Sum the 8 sampled values and divide by 8 (multiply by 0.125) to produce the final color.
    $$ FragColor = \left( \sum_{i=1}^{8} sample_i \right) \times 0.125 $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: TexCoord (vec2), coordZ (float), dX (float), dY (float), dZ (float), intensityVol (sampler3D)
# Output: FragColor (vec4)

def sample_3d_box_blur(TexCoord, coordZ, dX, dY, dZ, intensityVol):
    # 1. Construct 3D coordinate
    vx = vec3(TexCoord.x, TexCoord.y, coordZ)
    
    # 2. Define the 8 corner offsets
    offsets = [
        vec3(-dX, -dY, -dZ), vec3(-dX, -dY,  dZ),
        vec3(-dX,  dY, -dZ), vec3(-dX,  dY,  dZ),
        vec3( dX, -dY, -dZ), vec3( dX, -dY,  dZ),
        vec3( dX,  dY, -dZ), vec3( dX,  dY,  dZ)
    ]
    
    # 3. Sample and sum
    total = vec4(0.0)
    for offset in offsets:
        sample_pos = vx + offset
        # Assuming standard 3D texture sampling with wrapping/clamping handled by sampler
        total += texture(intensityVol, sample_pos)
        
    # 4. Average
    FragColor = total * 0.125
    
    return FragColor
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** If the box extends outside the volume bounds, texture sampling behavior (clamp, repeat, or border color) affects the result. Standard `sampler3D` behavior applies [S1, unofficial].
*   **Precision:** Use `highp` for the sampler and coordinates to prevent precision artifacts in the volume lookup [S1, unofficial].
*   **Degenerate Input:** If `dX`, `dY`, or `dZ` are zero, the box collapses to a line or point, effectively becoming a lower-dimensional blur or a single sample.

## Subtopics

### Box Blur
A 12-tap variant that samples along a line with varying weights based on alpha and distance [S2, unofficial]. It differs from the standard 8-tap box blur by using a weighted accumulation rather than a simple average, and by operating along a 1D line segment rather than a 3D cube [S2, unofficial]. Choose this when alpha-based weighting is required for metaball rendering or specific visual effects [S2, unofficial].

**Definition:** A 12-tap box blur that samples along a line with varying weights based on alpha and distance [S2, unofficial].

**Steps:**
1.  **Initialize:** Start with the center tap. Set `sampleCount` to the alpha of the center tap, and `sum` to the center tap multiplied by its alpha.
    $$ sampleCount = centerTap.a $$
    $$ sum = centerTap \times sampleCount $$
2.  **Calculate Step:** Determine the step size for the 12 taps based on the center tap's alpha and the number of taps (`TAPS`).
    $$ lenStep = centerTap.aa \times \left( \frac{1.0}{TAPS - 1.0} \right) $$
    *(Note: `centerTap.aa` likely refers to a specific component or property of the center tap, possibly alpha or a derived value, as per source [S2, unofficial])*
3.  **Loop and Sample:** Iterate `TAPS - 1` times. In each iteration, sample two points along the step vector.
    $$ sampleUV = i.uv1.xyxy + steps \times l $$
    where `l` is the loop index.
4.  **Calculate Weights:** Use `smoothstep` to determine weights based on the difference between the alpha and the step distance `r`.
    $$ weight01 = smoothstep(vec2(-0.4, -0.4), vec2(0.0, 0.0), maxLen01 - r) $$
5.  **Accumulate:** Add the weighted samples to the sum and the weights to the sample count.
    $$ sum += sample0 \times weight01.x + sample1 \times weight01.y $$
    $$ sampleCount += dot(weight01, vec2(1.0)) $$
6.  **Normalize:** Divide the sum by the total weight (sample count) plus a small epsilon to avoid division by zero.
    $$ returnValue = \frac{sum}{1e-5f + sampleCount} $$

**Parameters:**
*   `TAPS`: Number of taps (implied 12 from the name, but the loop runs `TAPS-1` times sampling two points, suggesting 1 + 2*(TAPS-1) samples? The source says "12-tap" and "loop TAPS-1 times... sampling two points". If TAPS=12, loop runs 11 times, sampling 22 points? This contradicts "12-tap". The source [S2, unofficial] is ambiguous. We report the steps as given.)

**Edge Cases:**
*   **Division by Zero:** The epsilon `1e-5f` prevents NaN results if `sampleCount` is zero [S2, unofficial].
*   **Alpha Dependency:** The weights depend heavily on the alpha value of the center tap, making this method sensitive to the input volume's alpha channel [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment, as it performs a fixed number of texture lookups (8 for the standard method, 12+ for the variant).
*   **Memory Complexity:** $O(1)$ auxiliary memory, only requiring uniforms and local variables.
*   **Performance:** Extremely fast due to the fixed, small kernel size and lack of complex weighting calculations in the standard version [S1, unofficial]. The weighted variant [S2, unofficial] is slightly more expensive due to the loop and smoothstep calculations.
*   **Strengths:** Simple implementation, low computational cost, effective for mild smoothing and anti-aliasing [S1, unofficial].
*   **Limitations:** The standard box blur can introduce aliasing artifacts if the box size is not chosen carefully relative to the voxel size [S1, unofficial]. The weighted variant [S2, unofficial] is specific to certain use cases (metaballs) and may not generalize well to standard volume rendering [S2, unofficial].
*   **When to Use:** Use the standard 8-tap box blur for general-purpose, low-cost smoothing in direct volume ray casting [S1, unofficial]. Use the weighted variant [S2, unofficial] only for specific effects like metaball rendering where alpha-based weighting is beneficial [S2, unofficial].

## Background
The standard 3D box blur is a common post-processing technique in volume rendering, often implemented in fragment shaders for real-time applications [S1, unofficial]. The weighted 12-tap variant appears in specialized contexts such as metaball rendering [S2, unofficial]. Both methods are part of the broader family of separable ping-pong blur post-processing techniques used to improve the visual quality of ray-casted volumes [Parent].

## Sources
[S1] rordenlab/MRIcroWeb, https://github.com/rordenlab/MRIcroWeb, fetched 2026-10-26 (unofficial)
[S2] dario-zubovic/metaballs, https://github.com/dario-zubovic/metaballs, fetched 2026-10-26 (unofficial)
