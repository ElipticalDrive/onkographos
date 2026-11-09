# Perlin-Worley Noise for Cloud Density

Also called: Base Density Noise, Base Density Noise Calculation, Perlin-Worley Noise

## Overview
This method generates the base density field for volumetric cloud rendering by combining the structural continuity of Perlin noise with the billowy, cellular detail of Worley noise [S2]. It differs from standard gradient-based noise by using Worley noise to modulate the amplitude of Perlin noise, creating organic cloud shapes that retain large-scale connectedness while exhibiting high-frequency billowiness at the edges [S2]. It serves as the primary input for the cloud density calculation in real-time volumetric rendering pipelines [S1, unofficial].

## How it works
The method constructs a single-channel density value by remapping a low-frequency Perlin fractal noise using high-frequency Worley fractal noise values, stored in a multi-channel 3D texture.

**Inputs and Outputs**
*   **Input:** A 3D texture containing four channels (RGBA) of pre-computed noise values.
*   **Output:** A scalar density value $D$ for a given 3D coordinate $\mathbf{P}$.

**Data Structures**
The base density noise is stored in a 3D texture with the following layout [S2]:
*   **Red (R) Channel:** Stores the low-frequency Perlin-Worley base noise.
*   **Green (G) Channel:** Stores Worley noise at frequency $f_1$.
*   **Blue (B) Channel:** Stores Worley noise at frequency $f_2$ ($f_2 > f_1$).
*   **Alpha (A) Channel:** Stores Worley noise at frequency $f_3$ ($f_3 > f_2$).
*   **Format:** The texture must be tileable [S1, unofficial]. A typical implementation uses a $128 \times 128 \times 128$ texture, which is small enough for web performance but large enough to hold 2–3 Fractal Brownian Motion (FBM) levels [S1, unofficial].

**Parameters**
*   **Density Offset ($\delta$):** A configurable scalar added to the Perlin-Worley value before final remapping. [S2]
*   **Worley Weights:** The high-frequency Worley combination uses fixed weights: $w_1 = 0.625$, $w_2 = 0.25$, $w_3 = 0.125$ [S2].

**Steps**
1.  **Sample Channels:** Retrieve the four noise values at coordinate $\mathbf{P}$:
    *   $N_{PW} = \text{Texture}(\mathbf{P}).R$
    *   $N_{W1} = \text{Texture}(\mathbf{P}).G$
    *   $N_{W2} = \text{Texture}(\mathbf{P}).B$
    *   $N_{W3} = \text{Texture}(\mathbf{P}).A$
2.  **Compute High-Frequency Worley:** Combine the three Worley channels into a single high-frequency detail term $N_{WHF}$ [S2]:
    $$ N_{WHF} = 0.625 N_{W1} + 0.25 N_{W2} + 0.125 N_{W3} $$
3.  **Apply Density Offset:** Add the configurable offset $\delta$ to the base Perlin-Worley value [S2]:
    $$ N'_{PW} = N_{PW} + \delta $$
4.  **Remap to Base Noise:** Remap $N'_{PW}$ from its original range $[0, 1]$ to a new range where the minimum is determined by $N_{WHF}$ and the maximum is $1$. This brightens the noise where Worley noise is large, enhancing billowiness [S2]:
    $$ D = \text{Remap}(N'_{PW}, N_{WHF}, 1, 0, 1) $$

**Formulas**
The core remapping operation is defined as [S2]:
$$ \text{Remap}(v, \text{oldMin}, \text{oldMax}, \text{newMin}, \text{newMax}) = \frac{v - \text{oldMin}}{\text{oldMax} - \text{oldMin}} \times (\text{newMax} - \text{newMin}) + \text{newMin} $$

Substituting the specific values for the base density calculation [S2]:
$$ D = \frac{(N_{PW} + \delta) - 0}{1 - 0} \times (1 - N_{WHF}) + N_{WHF} $$
$$ D = (N_{PW} + \delta)(1 - N_{WHF}) + N_{WHF} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cloud_density(P, texture, density_offset):
    """
    Computes cloud density using Perlin-Worley noise stored in a 3D texture.
    
    Args:
        P: 3D vector (x, y, z) coordinate to sample.
        texture: 3D texture object with .sample(x, y, z) returning (r, g, b, a).
        density_offset: Scalar float added to the base Perlin-Worley value.
        
    Returns:
        float: The final base density value.
    """
    # 1. Sample the four channels
    r, g, b, a = texture.sample(P.x, P.y, P.z)
    
    # 2. Compute High-Frequency Worley (N_WHf)
    # Weights: 0.625, 0.25, 0.125
    n_whf = 0.625 * g + 0.25 * b + 0.125 * a
    
    # 3. Apply density offset to Perlin-Worley (R channel)
    n_pw = r + density_offset
    
    # 4. Remap: [0, 1] -> [n_whf, 1]
    # Formula: (n_pw - 0) / (1 - 0) * (1 - n_whf) + n_whf
    density = n_pw * (1.0 - n_whf) + n_whf
    
    return density
```

**Edge Cases and Pitfalls**
*   **Precision:** Storing noise in 8-bit channels (RGB) limits detail to 256 discrete values. For higher fidelity, 32-bit floating-point formats (e.g., RFloat) are recommended, though they increase memory usage [S2].
*   **Tileability:** The 3D texture must be tileable to allow sampling at different frequencies in the shader without visible seams [S1, unofficial].
*   **Resolution:** A $128^3$ texture is considered small for web performance but may lack fine features; it is sufficient for 2–3 FBM levels [S1, unofficial].

## Operating characteristics
*   **Memory:** The base density texture requires 4 channels per voxel. For a $128^3$ texture with 32-bit floats, this is $128^3 \times 4 \times 4 \text{ bytes} \approx 512 \text{ MB}$. Using 8-bit integers reduces this to $128 \text{ MB}$ [S2].
*   **Performance:** Sampling is a single 3D texture lookup followed by simple arithmetic. The method is efficient enough for real-time rendering, particularly on web platforms where texture size is constrained [S1, unofficial].
*   **Strengths:** Combines the best of both noise types: Perlin provides structural continuity, while Worley adds organic billowiness. The multi-channel packing allows high-frequency detail without increasing texture resolution [S2].
*   **Limitations:** The fixed weights for Worley combination may not suit all artistic styles. The method relies on pre-computed textures, requiring offline generation or complex runtime generation [S2].

## Background
This technique is derived from the Nubis cloud rendering engine used by Guerrilla Games [S2]. It was popularized in tutorials and resources discussing efficient volumetric clouds, such as those found on the three.js forum and GitHub repositories detailing volumetric cloud implementations [S1, unofficial][S2]. The approach leverages Fractal Brownian Motion (FBM) to combine noise frequencies, a standard technique in procedural generation [S2].

## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S2] Noise Textures, https://github.com/TylerDodds/VolumetricCloudsTutorial/blob/master/Assets/VolumetricCloudsTutorial/CloudDensity/NoiseTextures/NoiseTextures.md, fetched 2026-10-29
