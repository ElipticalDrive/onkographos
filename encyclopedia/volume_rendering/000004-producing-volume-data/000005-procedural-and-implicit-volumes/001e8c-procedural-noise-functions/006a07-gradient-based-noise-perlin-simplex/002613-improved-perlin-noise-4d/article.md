# Improved Perlin Noise (4D)

Also called: 4D Perlin Noise

## Overview
Improved Perlin Noise (4D) is a procedural noise function that generates smooth, continuous random values by interpolating dot products of gradient vectors at the corners of a 4D hypercube [S1, unofficial]. It extends the 3D Improved Perlin Noise algorithm to four dimensions, primarily to support time-varying texture generation and animation by treating time as the fourth spatial axis [S1, unofficial]. Unlike its 3D sibling, it requires handling 16 corners and 4D gradient vectors, often optimized for graphics hardware using texture-based lookup tables to avoid branching [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 4D query point $\mathbf{p} = (x, y, z, w)$, typically where $w$ represents time or a seed offset [S1, S4, unofficial].
*   **Output:** A scalar noise value $n \in [-1, 1]$ (or $[0, 1]$ depending on post-processing) [S1, unofficial].

**Data Structures**
The implementation relies on precomputed lookup tables stored in textures to optimize GPU performance [S1, S2, unofficial]:
1.  **Permutation Table 1D (`_PermTable1D`):** A `Texture2D` of size $SIZE \times 1$ (e.g., $256 \times 1$), format `Alpha8`. Stores permutation indices normalized to $[0, 1]$ in the alpha channel [S1, S2, unofficial].
2.  **Permutation Table 2D (`_PermTable2D`):** A `Texture2D` of size $SIZE \times SIZE$ (e.g., $256 \times 256$), format `ARGB32`. Each texel stores four precomputed permutation sums: $(AA, AB, BA, BB)$ where $AA = \text{perm}(\text{perm}(x)+y)$, $AB = \text{perm}(\text{perm}(x)+y+1)$, etc., normalized to $[0, 1]$ [S1, S2, unofficial].
3.  **Gradient Table 4D (`_Gradient4D`):** A `Texture2D` of size $SIZE \times 1$ (e.g., $256 \times 1$), format `ARGB32`. Stores 32 gradient vectors normalized to $[0, 1]$ in RGBA channels. Each texel represents one 4D gradient vector [S1, S2, unofficial].

**Algorithm Steps**
1.  **Grid Location:** Compute the integer coordinates of the unit hypercube containing $\mathbf{p}$:
    $$ \mathbf{P} = \text{fmod}(\lfloor \mathbf{p} \rfloor, 256.0) $$
    Compute the relative position within the hypercube:
    $$ \mathbf{p}_{rel} = \mathbf{p} - \lfloor \mathbf{p} \rfloor $$
    [S1, S2, unofficial]

2.  **Fade Function:** Compute the fade curve for each dimension to ensure $C^1$ continuity:
    $$ f(t) = 6t^5 - 15t^4 + 10t^3 $$
    Applied component-wise to $\mathbf{p}_{rel}$ to get $\mathbf{f}$ [S1, S2, unofficial].

3.  **Hashing Corners:** Normalize $\mathbf{P}$ for texture lookup: $\mathbf{P}_{norm} = \mathbf{P} / 256.0$. Use the 2D permutation table to hash the XY coordinates and add the Z offset:
    $$ \mathbf{AA} = \text{perm2d}(\mathbf{P}_{norm}.xy) + \mathbf{P}_{norm}.z $$
    $\mathbf{AA}$ is a 4D vector containing the permutation indices for the base Z-slice [S1, unofficial].

4.  **Corner Indices:** Compute the permutation indices for all 16 corners of the 4D hypercube. Let $\text{one} = 1.0/256.0$. The indices are derived by adding the W coordinate ($\mathbf{P}_{norm}.w$) and incrementing the X, Y, Z, or W components of the permutation indices by $\text{one}$:
    $$
    \begin{aligned}
    \text{AAA} &= \text{perm}(\mathbf{AA}.x) + \mathbf{P}_{norm}.w, & \text{AAB} &= \text{perm}(\mathbf{AA}.x + \text{one}) + \mathbf{P}_{norm}.w \\
    \text{ABA} &= \text{perm}(\mathbf{AA}.y) + \mathbf{P}_{norm}.w, & \text{ABB} &= \text{perm}(\mathbf{AA}.y + \text{one}) + \mathbf{P}_{norm}.w \\
    \text{BAA} &= \text{perm}(\mathbf{AA}.z) + \mathbf{P}_{norm}.w, & \text{BAB} &= \text{perm}(\mathbf{AA}.z + \text{one}) + \mathbf{P}_{norm}.w \\
    \text{BBA} &= \text{perm}(\mathbf{AA}.w) + \mathbf{P}_{norm}.w, & \text{BBB} &= \text{perm}(\mathbf{AA}.w + \text{one}) + \mathbf{P}_{norm}.w
    \end{aligned}
    $$
    The "one" variants (e.g., $\text{AAA}+\text{one}$) correspond to the corners at $+1$ in the respective dimension [S1, unofficial].

5.  **Gradient Contributions:** For each corner, retrieve the gradient vector $\mathbf{g}$ from `_Gradient4D` and compute the dot product with the distance vector from the corner to $\mathbf{p}$. The gradient is unpacked from $[0, 1]$ to $[-1, 1]$:
    $$ \mathbf{g} = \text{tex2D}(\text{\_Gradient4D}, \text{float2}(x, 0)) \times 2.0 - 1.0 $$
    $$ n_i = \mathbf{g}_i \cdot (\mathbf{p} - \mathbf{V}_i) $$
    where $\mathbf{V}_i$ is the coordinate of corner $i$ relative to the hypercube origin (e.g., $\mathbf{0}$ for AAA, $\mathbf{-1}$ for BAA) [S1, S2, unofficial].

6.  **Interpolation:** Interpolate the 16 gradient contributions using nested linear interpolation (`lerp`) along the X, Y, Z, and W axes, weighted by the fade values $\mathbf{f}$ [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fade(t):
    # Quintic fade function for C1 continuity
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0)

def perm(x):
    # Lookup 1D permutation from texture, normalized to [0, 1]
    # In a CPU implementation, this would be a lookup into a precomputed array
    return texture_lookup(_PermTable1D, float2(x, 0)).a

def perm2d(uv):
    # Lookup 2D permutation from texture, returns float4 (AA, AB, BA, BB)
    return texture_lookup(_PermTable2D, uv)

def gradperm(x, p_rel):
    # Retrieve 4D gradient vector from texture, scale to [-1, 1]
    g = texture_lookup(_Gradient4D, float2(x, 0)) * 2.0 - 1.0
    return dot(g, p_rel)

def inoise(p):
    # 1. Find unit hypercube and relative position
    P = fmod(floor(p), 256.0)
    p_rel = p - floor(p)
    
    # 2. Compute fade curves
    f = fade(p_rel)
    
    # 3. Normalize for texture lookup
    P_norm = P / 256.0
    one = 1.0 / 256.0
    
    # 4. Hash coordinates of the 16 corners
    AA = perm2d(P_norm.xy) + P_norm.z
    
    AAA = perm(AA.x) + P_norm.w
    AAB = perm(AA.x + one) + P_norm.w
    ABA = perm(AA.y) + P_norm.w
    ABB = perm(AA.y + one) + P_norm.w
    BAA = perm(AA.z) + P_norm.w
    BAB = perm(AA.z + one) + P_norm.w
    BBA = perm(AA.w) + P_norm.w
    BBB = perm(AA.w + one) + P_norm.w
    
    # 5. Interpolate gradient contributions
    # Lerp along X, then Y, then Z, then W
    return lerp(
        lerp(
            lerp(
                lerp(gradperm(AAA, p_rel), gradperm(BAA, p_rel + float4(-1, 0, 0, 0)), f.x),
                lerp(gradperm(ABA, p_rel + float4(0, -1, 0, 0)), gradperm(BBA, p_rel + float4(-1, -1, 0, 0)), f.x),
            f.y),
            lerp(
                lerp(gradperm(AAB, p_rel + float4(0, 0, -1, 0)), gradperm(BAB, p_rel + float4(-1, 0, -1, 0)), f.x),
                lerp(gradperm(ABB, p_rel + float4(0, -1, -1, 0)), gradperm(BBB, p_rel + float4(-1, -1, -1, 0)), f.x),
            f.y),
        f.z),
        lerp(
            lerp(
                lerp(gradperm(AAA + one, p_rel + float4(0, 0, 0, -1)), gradperm(BAA + one, p_rel + float4(-1, 0, 0, -1)), f.x),
                lerp(gradperm(ABA + one, p_rel + float4(0, -1, 0, -1)), gradperm(BBA + one, p_rel + float4(-1, -1, 0, -1)), f.x),
            f.y),
            lerp(
                lerp(gradperm(AAB + one, p_rel + float4(0, 0, -1, -1)), gradperm(BAB + one, p_rel + float4(-1, 0, -1, -1)), f.x),
                lerp(gradperm(ABB + one, p_rel + float4(0, -1, -1, -1)), gradperm(BBB + one, p_rel + float4(-1, -1, -1, -1)), f.x),
            f.y),
        f.z),
    f.w)
```

**Parameters**
*   **Frequency:** Scaling factor for the input coordinate. Typical value: $1.0$ [S2, unofficial].
*   **Lacunarity:** Multiplier for frequency in each octave of fractal noise. Typical value: $2.0$ [S1, S2, unofficial].
*   **Gain:** Multiplier for amplitude in each octave. Typical value: $0.5$ [S1, S2, unofficial].
*   **Seed:** A value used to offset the 4th dimension ($w$) to simulate different seeds, preventing floating-point precision errors (e.g., `seed % 1000`) [S4, unofficial].

**Edge Cases and Pitfalls**
*   **Gradient Scaling:** Gradient vectors must be normalized or scaled appropriately (e.g., mapping texture $[0,1]$ to $[-1,1]$) to ensure consistent noise amplitude [S2, unofficial].
*   **Precision:** When using the 4th dimension for time or seeds, large coordinate values can cause floating-point precision errors. Offsetting the 4th dimension by a seed-derived value modulo 1000 is a common mitigation [S4, unofficial].
*   **Hardware Limits:** The algorithm requires dependent texture lookups, which may limit compatibility with older hardware (Pixel Shader 2.0) compared to M-Noise [S3, unofficial].

## Operating characteristics
*   **Complexity:** The algorithm performs 16 gradient lookups and 15 linear interpolations per octave.
*   **Performance:** On graphics hardware, a single octave uses approximately 64 pixel shader instructions [S3, unofficial]. This is higher than M-Noise (40 instructions) [S3, unofficial].
*   **Hardware Support:** Requires Pixel Shader 2.a or newer due to dependent texture lookups [S3, unofficial].
*   **Strengths:** Closer approximation to ideal Perlin Noise than M-Noise; supports 4D for animation [S3, unofficial].
*   **Limitations:** Higher instruction count; dependent texture lookups can be slower on some architectures [S3, unofficial].

## Background
Improved Perlin Noise was described by Green in 2005 [S3, unofficial]. The 4D variant was adapted for graphics hardware by Olano in 2005 in "Modified noise for evaluation on graphics hardware" [S3, unofficial]. It is used in real-time procedural volumetric fire simulations and other applications requiring time-varying noise [S3, unofficial].

## Sources

[S1] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S2] keijiro/MarchingCubesOnGPU, https://github.com/keijiro/MarchingCubesOnGPU, fetched 2026-10-26 (unofficial)
[S3] PDF Real-time Procedural Volumetric Fire - UC Davis, https://web.cs.ucdavis.edu/~hamann/FullerKrishnanMahrousHamannJoyFirePaperFor_I3D2007AsSubmitted11012006.pdf, fetched 2026-10-27 (unofficial)
[S4] sixthsurge/volume-noise-generator, https://github.com/sixthsurge/volume-noise-generator, fetched 2026-10-27 (unofficial)
