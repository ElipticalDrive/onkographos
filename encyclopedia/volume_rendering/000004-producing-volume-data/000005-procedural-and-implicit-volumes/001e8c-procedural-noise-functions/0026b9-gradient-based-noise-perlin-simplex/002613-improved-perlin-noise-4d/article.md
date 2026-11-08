# Improved Perlin Noise (4D)

Also called: 4D Perlin Noise

## Overview
Improved Perlin Noise (4D) is a procedural noise function that generates smooth, continuous random values in four dimensions by interpolating dot products of gradient vectors at the corners of a 4D hypercube [S1, unofficial]. It extends the 3D Improved Perlin Noise algorithm to support time-varying texture generation and 4D volume rendering, where the fourth dimension typically represents time or an additional spatial coordinate [S1, unofficial]. Unlike its 3D sibling, it utilizes a 4D gradient texture and optimized 2D permutation tables to maintain performance on GPUs, avoiding branching and expensive modulo operations [S2, unofficial].

## How it works
The method evaluates the noise value at a query point $\mathbf{P} = (x, y, z, w)$ by identifying the unit hypercube containing $\mathbf{P}$, computing gradient contributions at its 16 corners, and interpolating these contributions using quintic fade functions [S1, unofficial].

### Inputs and Outputs
*   **Input**: A 4D coordinate vector $\mathbf{P} = (x, y, z, w)$ [S1, unofficial].
*   **Output**: A scalar noise value $n \in [-1, 1]$ [S1, unofficial].

### Data Structures
The implementation relies on GPU textures to store precomputed permutation and gradient data, avoiding runtime array lookups and modulo operations [S2, unofficial].

1.  **Permutation Table 1D (`_PermTable1D`)**:
    *   Format: `Texture2D` of size $SIZE \times 1$ (e.g., $256 \times 1$).
    *   Layout: Stores 1D permutation indices in the Alpha channel, normalized to $[0, 1]$ [S2, unofficial].
    *   Access: Sampled at `float2(x, 0)` [S1, unofficial].

2.  **Permutation Table 2D (`_PermTable2D`)**:
    *   Format: `Texture2D` of size $SIZE \times SIZE$ (e.g., $256 \times 256$).
    *   Layout: Stores precomputed 2D permutation sums in RGBA channels. Each pixel at $(u, v)$ contains:
        *   $R$ (AA): $perm(perm(u) + v)$
        *   $G$ (AB): $perm(perm(u) + v + 1)$
        *   $B$ (BA): $perm(perm(u + 1) + v)$
        *   $A$ (BB): $perm(perm(u + 1) + v + 1)$
    *   Values are normalized to $[0, 1]$ [S2, unofficial].

3.  **Gradient 4D Table (`_Gradient4D`)**:
    *   Format: `Texture2D` of size $SIZE \times 1$ (e.g., $32 \times 1$ or $256 \times 1$ depending on gradient count).
    *   Layout: Stores 4D gradient vectors in RGBA channels. Each texel contains one gradient vector $\mathbf{g}$, with components normalized to $[0, 1]$ [S2, unofficial].
    *   Access: Sampled at `float2(x, 0)` [S1, unofficial].

### Algorithm Steps

1.  **Locate Hypercube and Relative Position**:
    Compute the integer grid coordinates of the hypercube corner $(0,0,0,0)$ and the relative position of $\mathbf{P}$ within the unit hypercube [S1, unofficial].
    $$ \mathbf{P}_{int} = \text{fmod}(\lfloor \mathbf{P} \rfloor, 256.0) $$
    $$ \mathbf{p}_{rel} = \mathbf{P} - \lfloor \mathbf{P} \rfloor $$
    [S1, unofficial]

2.  **Compute Fade Values**:
    Apply the quintic fade function to each component of $\mathbf{p}_{rel}$ to ensure $C^1$ continuity [S1, unofficial].
    $$ f(t) = 6t^5 - 15t^4 + 10t^3 $$
    $$ \mathbf{f} = \text{fade}(\mathbf{p}_{rel}) $$
    [S1, unofficial]

3.  **Normalize for Texture Lookup**:
    Normalize the integer coordinates to the $[0, 1]$ range for texture sampling [S1, unofficial].
    $$ \mathbf{P}_{norm} = \mathbf{P}_{int} / 256.0 $$
    [S1, unofficial]

4.  **Hash Corner Coordinates**:
    Compute the permutation indices for the 16 corners of the hypercube using the 2D and 1D permutation textures [S1, unofficial].
    *   Hash XY to get Z-offsets:
        $$ \mathbf{AA} = \text{perm2d}(\mathbf{P}_{norm}.xy) + \mathbf{P}_{norm}.z $$
        Where $\mathbf{AA}$ is a `float4` containing the permutation sums for the four Z-corners of the XY face [S1, unofficial].
    *   Hash remaining corners by adding W and incrementing X/Y/Z components:
        $$ \text{AAA} = \text{perm}(\mathbf{AA}.x) + \mathbf{P}_{norm}.w $$
        $$ \text{AAB} = \text{perm}(\mathbf{AA}.x + \text{one}) + \mathbf{P}_{norm}.w $$
        $$ \text{ABA} = \text{perm}(\mathbf{AA}.y) + \mathbf{P}_{norm}.w $$
        $$ \text{ABB} = \text{perm}(\mathbf{AA}.y + \text{one}) + \mathbf{P}_{norm}.w $$
        $$ \text{BAA} = \text{perm}(\mathbf{AA}.z) + \mathbf{P}_{norm}.w $$
        $$ \text{BAB} = \text{perm}(\mathbf{AA}.z + \text{one}) + \mathbf{P}_{norm}.w $$
        $$ \text{BBA} = \text{perm}(\mathbf{AA}.w) + \mathbf{P}_{norm}.w $$
        $$ \text{BBB} = \text{perm}(\mathbf{AA}.w + \text{one}) + \mathbf{P}_{norm}.w $$
        Where $\text{one} = 1.0 / 256.0$ [S1, unofficial].
    *   The 16 corners are: AAA, AAB, ABA, ABB, BAA, BAB, BBA, BBB (for $w=0$) and their counterparts with $+one$ added to the permutation index (for $w=1$) [S1, unofficial].

5.  **Compute Gradient Contributions**:
    For each corner hash index $h$, retrieve the gradient vector $\mathbf{g}$ from `_Gradient4D` and compute the dot product with the distance vector from the corner to $\mathbf{P}$ [S1, unofficial].
    $$ \mathbf{g} = (\text{tex2D}(\_Gradient4D, \text{float2}(h, 0)) \times 2.0 - 1.0) $$
    $$ \text{gradperm}(h, \mathbf{p}_{rel}) = \mathbf{g} \cdot \mathbf{p}_{rel} $$
    [S1, unofficial]
    Note: The gradient vector is unpacked from $[0, 1]$ to $[-1, 1]$ [S1, unofficial].

6.  **Interpolate**:
    Perform nested linear interpolation (`lerp`) across the 4D hypercube using the fade values $\mathbf{f}$ [S1, unofficial].
    $$ n = \text{lerp}( \text{lerp}( \text{lerp}( \text{lerp}( \text{gradperm}(\text{AAA}, \mathbf{p}_{rel}), \text{gradperm}(\text{BAA}, \mathbf{p}_{rel} + (-1,0,0,0)), f.x ), \text{lerp}( \text{gradperm}(\text{ABA}, \mathbf{p}_{rel} + (0,-1,0,0)), \text{gradperm}(\text{BBA}, \mathbf{p}_{rel} + (-1,-1,0,0)), f.x ), f.y ), \text{lerp}( \text{gradperm}(\text{AAB}, \mathbf{p}_{rel} + (0,0,-1,0)), \text{gradperm}(\text{BAB}, \mathbf{p}_{rel} + (-1,0,-1,0)), f.x ), \text{lerp}( \text{gradperm}(\text{ABB}, \mathbf{p}_{rel} + (0,-1,-1,0)), \text{gradperm}(\text{BBB}, \mathbf{p}_{rel} + (-1,-1,-1,0)), f.x ), f.z ), f.w ), \text{lerp}( \text{lerp}( \text{lerp}( \text{gradperm}(\text{AAA}+one, \mathbf{p}_{rel} + (0,0,0,-1)), \text{gradperm}(\text{BAA}+one, \mathbf{p}_{rel} + (-1,0,0,-1)), f.x ), \text{lerp}( \text{gradperm}(\text{ABA}+one, \mathbf{p}_{rel} + (0,-1,0,-1)), \text{gradperm}(\text{BBA}+one, \mathbf{p}_{rel} + (-1,-1,0,-1)), f.x ), f.y ), \text{lerp}( \text{gradperm}(\text{AAB}+one, \mathbf{p}_{rel} + (0,0,-1,-1)), \text{gradperm}(\text{BAB}+one, \mathbf{p}_{rel} + (-1,0,-1,-1)), f.x ), \text{lerp}( \text{gradperm}(\text{ABB}+one, \mathbf{p}_{rel} + (0,-1,-1,-1)), \text{gradperm}(\text{BBB}+one, \mathbf{p}_{rel} + (-1,-1,-1,-1)), f.x ), f.z ), f.w ) $$
    [S1, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fade(t):
    # Quintic fade function for C1 continuity
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0)

def perm(x, perm_table_1d):
    # Lookup 1D permutation from texture (normalized alpha)
    # Assumes perm_table_1d is sampled at float2(x, 0)
    return perm_table_1d.sample(x, 0).a

def perm2d(uv, perm_table_2d):
    # Lookup 2D permutation from texture (RGBA)
    # Returns float4 containing AA, AB, BA, BB permutation sums
    return perm_table_2d.sample(uv, 0)

def gradperm(h, p_rel, gradient_table_4d):
    # Retrieve gradient vector from texture and scale to [-1, 1]
    g = gradient_table_4d.sample(h, 0) * 2.0 - 1.0
    return dot(g, p_rel)

def inoise(p, perm_table_1d, perm_table_2d, gradient_table_4d):
    # 1. Locate hypercube and relative position
    P_int = fmod(floor(p), 256.0)
    p_rel = p - floor(p)
    
    # 2. Compute fade values
    f = fade(p_rel)
    
    # 3. Normalize for texture lookup
    P_norm = P_int / 256.0
    one = 1.0 / 256.0
    
    # 4. Hash corner coordinates
    # AA contains permutation sums for Z=0 face corners (AA, AB, BA, BB)
    AA = perm2d(P_norm.xy, perm_table_2d) + P_norm.z
    
    # Compute 1D permutation indices for all 16 corners
    # Corners with w=0
    AAA = perm(AA.x, perm_table_1d) + P_norm.w
    AAB = perm(AA.x + one, perm_table_1d) + P_norm.w
    ABA = perm(AA.y, perm_table_1d) + P_norm.w
    ABB = perm(AA.y + one, perm_table_1d) + P_norm.w
    BAA = perm(AA.z, perm_table_1d) + P_norm.w
    BAB = perm(AA.z + one, perm_table_1d) + P_norm.w
    BBA = perm(AA.w, perm_table_1d) + P_norm.w
    BBB = perm(AA.w + one, perm_table_1d) + P_norm.w
    
    # Corners with w=1 (increment permutation index by one)
    AAA_o = AAA + one
    AAB_o = AAB + one
    ABA_o = ABA + one
    ABB_o = ABB + one
    BAA_o = BAA + one
    BAB_o = BAB + one
    BBA_o = BBA + one
    BBB_o = BBB + one
    
    # 5. Compute gradient contributions
    # Note: p_rel adjustments correspond to the vector from the corner to P
    # Corner (0,0,0,0) -> p_rel
    # Corner (1,0,0,0) -> p_rel + (-1, 0, 0, 0)
    # etc.
    
    g_AAA = gradperm(AAA, p_rel, gradient_table_4d)
    g_BAA = gradperm(BAA, p_rel + float4(-1, 0, 0, 0), gradient_table_4d)
    g_ABA = gradperm(ABA, p_rel + float4(0, -1, 0, 0), gradient_table_4d)
    g_BBA = gradperm(BBA, p_rel + float4(-1, -1, 0, 0), gradient_table_4d)
    g_AAB = gradperm(AAB, p_rel + float4(0, 0, -1, 0), gradient_table_4d)
    g_BAB = gradperm(BAB, p_rel + float4(-1, 0, -1, 0), gradient_table_4d)
    g_ABB = gradperm(ABB, p_rel + float4(0, -1, -1, 0), gradient_table_4d)
    g_BBB = gradperm(BBB, p_rel + float4(-1, -1, -1, 0), gradient_table_4d)
    
    g_AAA_o = gradperm(AAA_o, p_rel + float4(0, 0, 0, -1), gradient_table_4d)
    g_BAA_o = gradperm(BAA_o, p_rel + float4(-1, 0, 0, -1), gradient_table_4d)
    g_ABA_o = gradperm(ABA_o, p_rel + float4(0, -1, 0, -1), gradient_table_4d)
    g_BBA_o = gradperm(BBA_o, p_rel + float4(-1, -1, 0, -1), gradient_table_4d)
    g_AAB_o = gradperm(AAB_o, p_rel + float4(0, 0, -1, -1), gradient_table_4d)
    g_BAB_o = gradperm(BAB_o, p_rel + float4(-1, 0, -1, -1), gradient_table_4d)
    g_ABB_o = gradperm(ABB_o, p_rel + float4(0, -1, -1, -1), gradient_table_4d)
    g_BBB_o = gradperm(BBB_o, p_rel + float4(-1, -1, -1, -1), gradient_table_4d)
    
    # 6. Interpolate
    # Interpolate along X
    lerp_x_000 = lerp(g_AAA, g_BAA, f.x)
    lerp_x_001 = lerp(g_ABA, g_BBA, f.x)
    lerp_x_010 = lerp(g_AAB, g_BAB, f.x)
    lerp_x_011 = lerp(g_ABB, g_BBB, f.x)
    lerp_x_100 = lerp(g_AAA_o, g_BAA_o, f.x)
    lerp_x_101 = lerp(g_ABA_o, g_BBA_o, f.x)
    lerp_x_110 = lerp(g_AAB_o, g_BAB_o, f.x)
    lerp_x_111 = lerp(g_ABB_o, g_BBB_o, f.x)
    
    # Interpolate along Y
    lerp_y_00 = lerp(lerp_x_000, lerp_x_001, f.y)
    lerp_y_01 = lerp(lerp_x_010, lerp_x_011, f.y)
    lerp_y_10 = lerp(lerp_x_100, lerp_x_101, f.y)
    lerp_y_11 = lerp(lerp_x_110, lerp_x_111, f.y)
    
    # Interpolate along Z
    lerp_z_0 = lerp(lerp_y_00, lerp_y_01, f.z)
    lerp_z_1 = lerp(lerp_y_10, lerp_y_11, f.z)
    
    # Interpolate along W
    result = lerp(lerp_z_0, lerp_z_1, f.w)
    
    return result
```

### Parameters
*   **Frequency (`_Frequency`)**: Scaling factor for the input coordinate. Typical value: $1.0$ [S2, unofficial]. In marching cubes contexts, default may be $0.02$ [S1, unofficial].
*   **Lacunarity (`_Lacunarity`)**: Multiplier for frequency in each octave of fractal noise. Typical value: $2.0$ [S2, unofficial].
*   **Gain (`_Gain`)**: Multiplier for amplitude in each octave. Typical value: $0.5$ [S2, unofficial].
*   **Noise Style (`_NoiseStyle`)**: Integer selector for fractal combination (0=fBm, 1=turbulence, 2=ridgedmf) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Gradient Scaling**: Gradient vectors must be normalized or scaled appropriately (typically unpacked from $[0, 1]$ to $[-1, 1]$) to ensure consistent noise amplitude [S2, unofficial].
*   **Precision**: Using `fmod` with large coordinates can lead to precision loss. The modulo $256$ ensures indices fit in the permutation table, but the input $\mathbf{P}$ should ideally be kept within reasonable bounds or wrapped [S1, unofficial].
*   **Boundary Conditions**: The use of `PointRepeat` sampler state on textures handles boundary wrapping for permutation and gradient lookups, avoiding edge artifacts [S1, unofficial].

## Subtopics

### Simplex Noise
A gradient-based noise function that uses a simplex grid (triangular/hexagonal lattice) instead of a cubic grid, reducing the number of corners from 16 to 8 in 4D, improving performance and reducing directional artifacts [Siblings].

### 3D Perlin-like Noise and Fractal Brownian Motion (FBM)
The 3D variant of this method, often combined with FBM to create multi-octave noise for more complex textures [Siblings].

### Perlin Noise 2D
The 2D version of this method, using a 2D grid and 4 corners, serving as the foundational algorithm for higher dimensions [Siblings].

### Perlin Gradient and Interpolation Functions
Details on the specific gradient sets and fade functions used in Perlin variants, including the quintic fade function used here [Siblings].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per noise evaluation, as it involves a fixed number of arithmetic operations and texture lookups (16 gradient evaluations, 15 lerp operations) [S1, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory, dominated by the size of the permutation and gradient textures (e.g., $256 \times 256$ for 2D perm, $32 \times 1$ for 4D gradients) [S2, unofficial].
*   **Performance**: GPU-based implementations using texture lookups for permutation and gradients are significantly faster than CPU-based array lookups due to cache efficiency and avoidance of branching [S2, unofficial].
*   **Strengths**: Smooth $C^1$ continuity, isotropic appearance, efficient GPU implementation, support for 4D (time-varying) noise [S1, unofficial].
*   **Limitations**: Higher computational cost than 3D due to 16 corners and 4D interpolation; potential for directional artifacts if gradient set is not carefully chosen (though 4D gradient texture mitigates this) [S2, unofficial].
*   **When to use**: When 4D procedural noise is required, such as for time-varying textures, 4D volume rendering, or fluid simulation noise [S1, unofficial].
*   **When not to use**: When performance is critical and 3D noise suffices, or when simpler noise functions (like value noise) are adequate [Siblings].

## Background
Improved Perlin Noise (4D) is an extension of Ken Perlin's Improved Perlin Noise, which was introduced in 2002 to fix artifacts in the original 1985 Perlin Noise [S1, unofficial]. The 4D variant is commonly used in computer graphics for generating time-varying textures and in volume rendering for creating dynamic noise fields [S1, unofficial]. The GPU-optimized implementation using texture-based permutation and gradient tables was popularized in open-source projects such as "Marching Cubes On The GPU" [S1, S2, unofficial].

## Sources
[S1] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S2] keijiro/MarchingCubesOnGPU, https://github.com/keijiro/MarchingCubesOnGPU, fetched 2026-10-26 (unofficial)
