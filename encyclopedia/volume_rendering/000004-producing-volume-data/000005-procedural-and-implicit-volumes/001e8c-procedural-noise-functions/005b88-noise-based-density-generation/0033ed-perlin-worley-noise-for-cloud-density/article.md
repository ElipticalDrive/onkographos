# Perlin-Worley Noise for Cloud Density

Also called: Base Density Noise, Base Density Noise Calculation, Perlin-Worley Noise

## Overview
This method generates the base volumetric density field for cloud rendering by combining the structural continuity of Perlin noise with the billowy, cellular detail of Worley noise [S2]. It differs from standard procedural noise by storing multiple frequency levels of Worley noise in separate texture channels to allow for high-frequency detail reconstruction during runtime remapping, rather than baking all detail into a single low-precision channel [S2].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D texture containing the base density noise and high-frequency Worley components.
*   **Output:** A single scalar density value for a given 3D coordinate $\mathbf{P}$.

**Data Structures**
The base density is stored in an RGBA32 texture (or equivalent floating-point format) [S2]. The layout is defined as follows [S2]:
*   **Red (R) Channel:** Stores the low-frequency Perlin-Worley noise value.
*   **Green (G) Channel:** Stores Worley noise at frequency level 1.
*   **Blue (B) Channel:** Stores Worley noise at frequency level 2.
*   **Alpha (A) Channel:** Stores Worley noise at frequency level 3.

The texture must be tileable to allow sampling at varying frequencies without edge artifacts [S1, unofficial]. A typical implementation uses a $128 \times 128 \times 128$ texture, which is small enough for fast loading but large enough to contain 2–3 Fractal Brownian Motion (FBM) levels [S1, unofficial].

**Algorithm Steps**
1.  **Generate Perlin Fractal:** Compute a low-frequency Perlin fractal noise value, $PerlinFractal$, using standard FBM summation [S2].
2.  **Generate Worley Fractals:** Compute three separate Worley fractal noise values at increasing frequencies: $Worley_1$, $Worley_2$, and $Worley_3$ [S2].
3.  **Pack Noise:** Store $PerlinFractal$ in the Red channel and $Worley_1, Worley_2, Worley_3$ in the Green, Blue, and Alpha channels, respectively [S2].
4.  **Unpack and Combine Worley:** Retrieve the G, B, and A channels from the texture. Compute the high-frequency Worley sum, $WorleyHighFrequencies$, using weighted summation [S2]:
    $$ WorleyHighFrequencies = 0.625 \cdot Worley_1 + 0.25 \cdot Worley_2 + 0.125 \cdot Worley_3 $$
5.  **Remap Perlin-Worley:** Retrieve the Red channel value, $PerlinWorley$. Add a configurable density offset, $\Delta$, to it [S2]. Remap this value from the range $[0, 1]$ to $[WorleyFractal, 1]$. This step brightens the noise where Worley noise is large, enhancing billowiness [S2].
    $$ PerlinWorley_{remapped} = \text{Remap}(PerlinWorley + \Delta, 0, 1, WorleyFractal, 1) $$
    *Note: The source notes mention $WorleyFractal$ in the formula context but use $WorleyHighFrequencies$ in the final base noise calculation. The remapping step described in the text ("brightens the noise where Worley noise is large") implies the use of the high-frequency sum for the final shaping.*
6.  **Compute Base Noise:** Remap the result from Step 5 such that its minimum becomes $WorleyHighFrequencies$ and its maximum becomes 1 [S2]:
    $$ BaseNoise = \text{Remap}(PerlinWorley_{remapped}, WorleyHighFrequencies, 1, 0, 1) $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cloud_density(texture, p, density_offset):
    """
    Computes cloud density using Perlin-Worley noise stored in an RGBA texture.
    
    Args:
        texture: A 3D texture object with sample(x, y, z) returning (r, g, b, a).
        p: 3D vector coordinate.
        density_offset: Configurable float added to Perlin-Worley value.
        
    Returns:
        float: The final base density value.
    """
    # 1. Sample the packed noise texture
    r, g, b, a = texture.sample(p)
    
    # 2. Unpack Worley components
    worley_1 = g
    worley_2 = b
    worley_3 = a
    
    # 3. Combine Worley frequencies
    # Weights: 0.625, 0.25, 0.125
    worley_high_freq = 0.625 * worley_1 + 0.25 * worley_2 + 0.125 * worley_3
    
    # 4. Get Perlin-Worley base value
    perlin_worley = r
    
    # 5. Add density offset
    perlin_worley_adjusted = perlin_worley + density_offset
    
    # 6. Remap Perlin-Worley based on Worley high frequencies
    # The source formula: Remap(PerlinWorley, WorleyHighFrequencies, 1, 0, 1)
    # This maps the range [WorleyHighFrequencies, 1] to [0, 1]
    # However, the text says "Remap PerlinWorley such that its minimum becomes WorleyHighFrequencies"
    # Standard remap(x, in_min, in_max, out_min, out_max)
    # Here, we treat the PerlinWorley value as the input x.
    # The source formula for BaseNoise is: Remap(PerlinWorley, WorleyHighFrequencies, 1, 0, 1)
    
    base_noise = remap(perlin_worley_adjusted, worley_high_freq, 1.0, 0.0, 1.0)
    
    return base_noise

def remap(value, in_min, in_max, out_min, out_max):
    """Linear remapping function."""
    if in_max == in_min:
        return out_min
    t = (value - in_min) / (in_max - in_min)
    # Clamp t to [0, 1] to handle out-of-bounds values
    t = max(0.0, min(1.0, t))
    return out_min + t * (out_max - out_min)
```

**Parameters**
*   **Texture Size:** $128 \times 128 \times 128$ is cited as a practical minimum for web performance, though larger sizes allow for more FBM levels [S1, unofficial].
*   **Worley Weights:** $0.625$ for the lowest frequency, $0.25$ for the middle, and $0.125$ for the highest [S2].
*   **Density Offset:** A configurable scalar added to the Perlin-Worley value before remapping [S2].

**Edge Cases and Pitfalls**
*   **Precision:** Storing noise in 8-bit channels limits detail. Using 32-bit floating-point textures (e.g., RFloat) is recommended to preserve the full range of $[0, 1]$ values and avoid banding [S2].
*   **Tileability:** The texture must be tileable to support sampling at different frequencies without discontinuities [S1, unofficial].
*   **Channel Usage:** The Alpha channel is used for Worley noise in the base density texture, whereas the detail density texture may leave the Alpha channel unused [S2].

## Operating characteristics
*   **Memory:** Requires storing a 3D texture. A $128^3$ RGBA32 texture consumes approximately 8 MB ($128^3 \times 4$ bytes) [S1, unofficial].
*   **Performance:** Sampling is fast as it involves a single 3D texture lookup and simple arithmetic [S1, unofficial]. The method is designed to be efficient for real-time rendering, particularly on web platforms [S1, unofficial].
*   **Quality:** The combination of Perlin (connectedness) and Worley (billowiness) produces natural-looking cloud structures [S2]. The multi-channel Worley remapping allows for high-frequency detail without requiring extremely large textures [S2].

## Background
This technique is derived from the Nubis cloud rendering engine used by Guerrilla Games [S2]. It is implemented in the "Volumetric Clouds Tutorial" by Tyler Dodds, which provides source code for generating and unpacking these noise textures [S2]. The approach is also discussed in the context of efficient volumetric clouds for WebGL/three.js [S1, unofficial].

## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S2] Noise Textures, https://github.com/TylerDodds/VolumetricCloudsTutorial/blob/master/Assets/VolumetricCloudsTutorial/CloudDensity/NoiseTextures/NoiseTextures.md, fetched 2026-10-29
