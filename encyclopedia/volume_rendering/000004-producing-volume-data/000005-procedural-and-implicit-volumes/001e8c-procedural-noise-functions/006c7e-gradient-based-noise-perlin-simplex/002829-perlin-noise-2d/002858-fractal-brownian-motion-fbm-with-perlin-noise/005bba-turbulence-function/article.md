# Turbulence Function

Also called: Improved Perlin Noise for Fire, Micro Fire Effects via Noise, Modified Noise, Procedural Noise for Micro-level Details, Turbulence, Improved Perlin Noise, M-Noise

## Overview
The Turbulence Function is a procedural noise technique that generates complex, fractal-like micro-details for volumetric effects such as fire, smoke, and fog [S1, S2][S3, unofficial]. It works by summing multiple octaves of a base gradient noise function with increasing frequency and decreasing amplitude [S2][S3, unofficial]. It builds on Fractal Brownian Motion (fBm) by specifically applying summed noise octaves to simulate natural fractal phenomena in real-time graphics [S2][S3, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `position`: A 3D or 4D vector representing spatial (and optionally temporal) coordinates in noise space [S2].
*   `octaves`: The number of noise octaves to sum (integer) [S1, S2].
*   `gain`: A scalar controlling the relative amplitude of each octave [S2][S3, unofficial].
*   `lacunarity`: A scalar controlling the relative frequency of each octave [S2][S3, unofficial].
*   `noise_scale`: An application-specific frequency multiplier to correct for inherent frequency differences in noise functions and scale noise space to world coordinates [S2].
*   `time`: A scalar for the temporal dimension (if using 4D noise) [S2].
*   `offset`: A constant vector to ensure uniqueness per fire instance [S2].

**Outputs:**
*   `turb`: A scalar turbulence value representing the accumulated noise magnitude [S1, S2].

### Data Structures
The method relies on scalar arithmetic and the evaluation of a base noise function $N(p)$ [G]. No complex data structures are required.

### Steps
1.  **Scale Position**: Compute the base noise position by scaling the input coordinates by `noise_scale` and adding the instance `offset` [S2].
2.  **Initialize Accumulator**: Set `turb = 0` [G].
3.  **Loop Octaves**: For each octave $i$ from $0$ to `octaves` - 1:
    a.  Evaluate the base noise function at the scaled position: $val = N(position \cdot lacunarity_i)$ [S1, S2].
    b.  Take the absolute value of the noise result: $val = |val|$ [S1].
    c.  Multiply by the gain for this octave: $val = val \cdot gain_i$ [S1, S2].
    d.  Add to the accumulator: $turb = turb + val$ [G].
4.  **Return**: Output `turb` [G].

### Formulas
The turbulence value $turb$ is defined as [S1]:
$$
turb = \sum_{i=0}^{n} gain_i \cdot |noise(position \cdot lacunarity_i)|
$$
Where:
*   $n$ is the number of octaves [S1].
*   $gain_i$ is the gain for octave $i$. Typically, $gain = 1/lacunarity$ or a constant like $0.5$ [S1, S2][S3, unofficial].
*   $lacunarity_i$ is the frequency multiplier for octave $i$. Typically $lacunarity = 2$ [S2][S3, unofficial].
*   $position$ is the noise coordinate vector [S2].
*   $noise(p)$ is the base gradient noise function (e.g., M-Noise) [S1, S2].
*(Note: Sources [S2][S3, unofficial] define the turbulence function without the absolute value operator, but [S1] explicitly includes it for the standard turbulence application.)*

**Note on M-Noise Optimization**: When using M-Noise, hash functions can be computed for up to two octaves together, and flerp functions for up to four octaves together in 3D turbulence, or three octaves for hash and four for flerp in 4D turbulence [S1].

### Parameters
*   **Gain**: Controls relative amplitude. Common value: $0.5$ [S2][S3, unofficial].
*   **Lacunarity**: Controls relative frequency. Common value: $2$ [S2][S3, unofficial].
*   **Octaves**: Number of octaves. $3+$ for realistic fire, $\le 2$ for stylized/cartoon environments [S2][S3, unofficial].
*   **Noise Scale**: Application-specific frequency multiplier to correct for inherent frequency differences in noise functions and scale noise space to world coordinates [S2].

### Edge Cases and Pitfalls
*   **Stretching Artifacts**: Enlarging the fire volume (elongating curves or adjusting radius) can stretch noise effects, creating visual artifacts. This is addressed by dynamically scaling noise space according to the radius and length of the fire volume [S2].
*   **Precision**: The method relies on the stability of the base noise function. M-Noise is preferred for its dimension reducibility and purely computable nature, avoiding texture-dependent lookups [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def turbulence(position, octaves, gain, lacunarity, noise_func):
    """
    Computes the turbulence value by summing octaves of noise.
    
    Args:
        position (vec3 or vec4): The input coordinate in noise space.
        octaves (int): Number of octaves to sum.
        gain (float): Amplitude multiplier for each octave.
        lacunarity (float): Frequency multiplier for each octave.
        noise_func (function): A function that takes a position and returns a scalar noise value.
        
    Returns:
        float: The accumulated turbulence value.
    """
    turb = 0.0
    current_freq = 1.0
    
    for i in range(octaves):
        # Evaluate base noise at scaled position
        val = noise_func(position * current_freq)
        
        # Turbulence typically uses the absolute value of noise [S1]
        val = abs(val)
        
        # Apply gain
        val = val * gain
        
        # Accumulate
        turb = turb + val
        
        # Increase frequency for next octave
        current_freq = current_freq * lacunarity
        
    return turb
```

## Subtopics

### M-Noise
A GPU-tailored adaptation of Perlin Noise that picks gradients from the corners of a unit-n cube instead of edge centers, allowing dimension reducibility and purely computable noise without texture lookups [S1, S2]. It is more efficient than Improved Perlin Noise, using ~40 pixel shader instructions per octave compared to ~64, and supports Pixel Shader 2.0 [S2][S3, unofficial].

### Improved Perlin Noise
A CPU-optimized version of Perlin Noise adapted for graphics hardware. It uses six dependent texture look-ups and ~64 instructions per octave, requiring Pixel Shader 2.a or newer [S2][S3, unofficial]. It is less efficient than M-Noise but may more closely approximate real Perlin Noise [S2][S3, unofficial].

### Perlin Noise Modulation
A stylization technique that modulates the final rendering color or shading disk sizes using a turbulent 3D Perlin noise texture mapped to the model [S6, unofficial]. At shading time, the final color of a disk is modulated with the noise value at the nearest location in the noise texture, or shading disk sizes are disturbed for halftone renderings [S6, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of octaves $n$. Each octave requires evaluating the base noise function (M-Noise uses two independent look-ups per octave) [S2][S3, unofficial].
*   **Memory Complexity**: Constant, $O(1)$, as the method is purely computable and stores no large data structures [S1, S2].
*   **Performance**: M-Noise uses approximately 40 pixel shader instructions per octave and can be compiled for Pixel Shader 2.0 hardware [S2][S3, unofficial]. Improved Perlin Noise uses approximately 64 instructions per octave and requires Pixel Shader 2.a or newer [S2][S3, unofficial].
*   **Strengths**: Efficient GPU implementation (especially M-Noise), produces realistic fractal details for fire and smoke, easily customizable via gain and lacunarity [S2][S3, unofficial].
*   **Limitations**: Visual artifacts can occur if noise space is not dynamically scaled with the fire volume geometry [S2]. M-Noise may have a different visual character than classic Perlin Noise, though it is often preferred for its efficiency [S1][S3, unofficial].
*   **When to Use**: Real-time volumetric fire, smoke, and fog rendering where performance is critical and realistic micro-details are desired [S2][S3, unofficial].
*   **When Not to Use**: When classic Perlin Noise visual characteristics are strictly required and hardware supports only older shaders, or when offline rendering allows for more expensive noise functions [G].

## Background
The Turbulence Function, in the context of GPU-accelerated procedural noise, was described by Marc Olano in 2005 in "Modified noise for evaluation on graphics hardware" (HWWS '05) [S3, unofficial]. It was developed to address the inefficiencies of classic and Improved Perlin Noise on graphics hardware, specifically by reducing dependent texture lookups and leveraging GPU parallelism [S1, S2]. The technique is used in pixel shaders to generate micro fire effects such as individual flame shape, location, and flicker [S2][S4, S5, unofficial].

## Sources
[S1] Pseudorandom Noise for Real-Time Volumetric Rendering of Fire in a Production System, https://doi.org/10.2312/vg/vg-pbg08/129-136, fetched 2026-10-26
[S2] Real-time procedural volumetric fire, https://doi.org/10.1145/1230100.1230131, fetched 2026-11-05
[S3] PDF Real-time Procedural Volumetric Fire - UC Davis, https://web.cs.ucdavis.edu/~hamann/FullerKrishnanMahrousHamannJoyFirePaperFor_I3D2007AsSubmitted11012006.pdf, fetched 2026-10-27 (unofficial)
[S4] Realistic Real-Time Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-proposals/YVanzine_Proposal.pdf, fetched 2026-10-28 (unofficial)
[S5] Realistic Real-Time Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-proposals/YVanzine_Proposal.pdf, fetched 2026-10-28 (unofficial)
[S6] GMS | GMS Current Topics in Computer and Robot Assisted Surgery | Expressive anatomical illustrations based on scanned patien..., https://www.egms.de/static/de/journals/curac/2006-1/curac000009.shtml, fetched 2026-10-30 (unofficial)
