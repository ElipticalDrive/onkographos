# Terrain Heightmap Generation via Noise

Also called: GetHeightAtWorldLocation, Heightmap Generation via Noise, Perlin Noise Height Generation, Procedural terrain heightmap, Terrain Height Generation, World Height Noise

## Overview
This method generates a 2D terrain heightmap by sampling a 3D scalar noise field at a fixed vertical coordinate, effectively projecting the layered noise onto the X-Y plane. It solves the problem of creating continuous, organic-looking elevation data for procedural terrain without requiring pre-baked mesh data. It differs from its parent, the [Noise Field Generator](1302-noise-field-generator.md), by fixing the Z-dimension (ignoring the vertical gradient offset) to produce a single scalar height value per X/Y coordinate, rather than a full 3D volume. It is distinct from [FBM Combinations](3962-fbm-combinations.md) and [Domain Warped FBM](4493-domain-warped-fbm.md) by using standard additive fractal summation without domain warping or complex combination logic.

## How it works
The method takes 2D world-space coordinates and returns a single height value. It relies on the underlying [Noise Field Generator](1302-noise-field-generator.md) which provides a 3D scalar field using layered Simplex noise. However, the specific implementation details in the sources describe a 2D Perlin noise variant for this specific heightmap application. The process involves scaling the input coordinates, computing a Fractal Brownian Motion (FBM) sum, and remapping the result to a target vertical range.

### Inputs and Outputs
*   **Inputs**:
    *   `WorldX`, `WorldY`: 2D world-space coordinates (float).
    *   `Settings`: A configuration struct containing `Seed`, `Scale`, `Octaves`, `Persistence`, `Lacunarity`, `BaseHeight`, and `HeightScale`.
*   **Outputs**:
    *   `Height`: A single float representing the terrain elevation at `(WorldX, WorldY)`.

### Data Structures
The implementation relies on a permutation table `P` of size 512 (two copies of 256) to handle wrap-around for gradient indices. The `Settings` struct contains:
*   `Seed` (int32): Random seed for noise generation. Default: 1337 [S1, unofficial].
*   `Scale` (float): World units per noise cell. Controls smoothness. Typical range: 1000-5000. Default: 2000.0f [S1, unofficial].
*   `Octaves` (int32): Number of noise layers summed. Range: 1-8. Default: 4 [S1, unofficial].
*   `Persistence` (float): Amplitude falloff per octave. Range: 0.0-1.0. Default: 0.5f [S1, unofficial].
*   `Lacunarity` (float): Frequency multiplier per octave. Default: 2.0f [S1, unofficial].
*   `BaseHeight` (float): Final output offset. Default: 0.0f [S1, unofficial].
*   `HeightScale` (float): Final output amplitude. Default: 500.0f [S1, unofficial] [S2, unofficial].

### Steps
1.  **Calculate Scale Factor**: Ensure the scale is not zero to avoid division by zero.
    $$ Scale = \max(Settings.Scale, \epsilon) $$
    where $\epsilon$ is a small number (e.g., $10^{-6}$) [S1, unofficial].
2.  **Normalize Coordinates**: Convert world coordinates to noise space.
    $$ NoiseX = \frac{WorldX}{Scale} $$
    $$ NoiseY = \frac{WorldY}{Scale} $$
    [S1, unofficial]
3.  **Compute Raw Noise**: Calculate the fractal noise value in the range $[-1, 1]$ using the `FractalPerlin2D` function (see pseudocode).
    $$ Noise = FractalPerlin2D(NoiseX, NoiseY, Settings) $$
    [S1, unofficial]
4.  **Remap to Target Range**: Map the noise value to the final height range. Two variations exist in the sources:
    *   **Linear Offset/Scale (Source 1)**:
        $$ Height = BaseHeight + Noise \times HeightScale $$
        [S1, unofficial]
    *   **Min/Max Mapping (Source 2)**:
        $$ Height = minHeight + \lfloor (Noise + 1.0) \times 0.5 \times (maxHeight - minHeight) \rfloor $$
        [S2, unofficial]
    *Note: Source 1 uses `BaseHeight` and `HeightScale` to define the range $[BaseHeight - HeightScale, BaseHeight + HeightScale]$. Source 2 uses explicit `minHeight` and `maxHeight`.*

### Pseudocode
The following pseudocode implements the logic from [S1, unofficial], which is the more detailed procedural implementation provided. It assumes a standard 2D Perlin noise function `RawPerlin2D` is available, which returns a value roughly in $[-1, 1]$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TerrainHeightmapGeneration(WorldX, WorldY, Settings):
    """
    Generates terrain height at WorldX, WorldY.
    
    Args:
        WorldX (float): X coordinate in world space.
        WorldY (float): Y coordinate in world space.
        Settings (dict): Configuration object containing:
            - Seed (int): Noise seed.
            - Scale (float): World units per noise cell.
            - Octaves (int): Number of FBM octaves.
            - Persistence (float): Amplitude falloff.
            - Lacunarity (float): Frequency multiplier.
            - BaseHeight (float): Center height offset.
            - HeightScale (float): Vertical amplitude.
            
    Returns:
        float: The calculated height.
    """
    # Step 1: Calculate Scale Factor
    scale = max(Settings.Scale, 1e-6)
    
    # Step 2: Normalize world coordinates
    noise_x = WorldX / scale
    noise_y = WorldY / scale
    
    # Step 3: Compute raw noise value in [-1, 1]
    noise = FractalPerlin2D(noise_x, noise_y, Settings)
    
    # Step 4: Remap to target range
    # Using the formula from [S1, unofficial]: Height = BaseHeight + Noise * HeightScale
    height = Settings.BaseHeight + noise * Settings.HeightScale
    
    return height

def FractalPerlin2D(X, Y, Settings):
    """
    Computes Fractal Brownian Motion using 2D Perlin noise.
    Returns a value roughly in [-1, 1].
    """
    total = 0.0
    frequency = 1.0
    amplitude = 1.0
    max_value = 0.0
    
    # Clamp octaves to valid range [1, 8] as per [S1, unofficial]
    octaves = max(1, min(Settings.Octaves, 8))
    
    for i in range(octaves):
        # Call raw Perlin noise with current frequency
        # Note: RawPerlin2D is assumed to be implemented per standard Perlin noise
        # See [S1, unofficial] for the underlying permutation and gradient logic.
        raw_noise = RawPerlin2D(X * frequency, Y * frequency, Settings.Seed)
        
        total += raw_noise * amplitude
        max_value += amplitude
        
        # Update amplitude and frequency for next octave
        amplitude *= Settings.Persistence
        frequency *= Settings.Lacunarity
        
    # Normalize to ensure range is roughly [-1, 1]
    if max_value > 0:
        return total / max_value
    else:
        return 0.0

def RawPerlin2D(X, Y, Seed):
    """
    Standard 2D Perlin Noise implementation.
    Based on the logic in [S1, unofficial] (UWorldHeightNoise::Noise).
    """
    # Find unit cube that contains point
    xi = int(math.floor(X)) & 255
    yi = int(math.floor(Y)) & 255
    
    # Find relative x, y of point in cube
    xf = X - math.floor(X)
    yf = Y - math.floor(Y)
    
    # Compute fade curves
    u = Fade(xf)
    v = Fade(yf)
    
    # Hash coordinates of the 4 corners of the cube
    # Permutation table P is initialized with Seed in the Noise Field Generator
    # For brevity, we assume P is accessible. In a full implementation, 
    # P is generated once per Seed.
    
    aa = P[P[xi] + yi]
    ab = P[P[xi] + yi + 1]
    ba = P[P[xi + 1] + yi]
    bb = P[P[xi + 1] + yi + 1]
    
    # Gradient functions
    # Grad(hash, x, y) computes dot product of gradient vector and distance vector
    # Standard Perlin gradients for 2D: (1,1), (-1,1), (1,-1), (-1,-1), etc.
    
    g_aa = Grad(aa, xf, yf)
    g_ba = Grad(ba, xf - 1.0, yf)
    g_ab = Grad(ab, xf, yf - 1.0)
    g_bb = Grad(bb, xf - 1.0, yf - 1.0)
    
    # Interpolate
    x1 = Lerp(u, g_aa, g_ba)
    x2 = Lerp(u, g_ab, g_bb)
    
    result = Lerp(v, x1, x2)
    
    return result

def Fade(t):
    return t * t * t * (t * (t * 6 - 15) + 10)

def Lerp(t, a, b):
    return a + t * (b - a)

def Grad(hash, x, y):
    h = hash & 3
    if h == 0: return x + y
    elif h == 1: return -x + y
    elif h == 2: return x - y
    else: return -x - y
```

### Edge Cases and Pitfalls
*   **Division by Zero**: If `Settings.Scale` is 0, the coordinate normalization will fail. The scale must be clamped to a small positive number [S1, unofficial].
*   **Octave Clamping**: The number of octaves should be clamped to the range [1, 8] to prevent excessive computation or undefined behavior [S1, unofficial].
*   **Precision**: The noise function returns values "roughly" in $[-1, 1]$. The normalization step in `FractalPerlin2D` (dividing by `MaxValue`) ensures the output is strictly bounded, preventing overflow in the final remapping step [S1, unofficial].
*   **Coordinate Wrap**: The permutation table lookup uses bitwise AND with 255 (`& 255`) to handle negative coordinates and large values, ensuring the index stays within the 256-entry table [S1, unofficial].

## Subtopics

### FBM Combinations
[FBM Combinations](3962-fbm-combinations.md) extends the basic fractal summation by allowing non-standard combinations of noise layers (e.g., multiplication, subtraction) rather than simple additive summation. Choose this when you need specific spectral shapes or non-Gaussian height distributions.

### Domain Warped FBM
[Domain Warped FBM](4493-domain-warped-fbm.md) applies a secondary noise function to distort the input coordinates of the primary noise. Choose this for creating more complex, organic features like ridges or swirls that standard FBM cannot produce.

## Operating characteristics
*   **Time Complexity**: $O(O)$, where $O$ is the number of octaves. Each octave requires one evaluation of the base Perlin noise function. The base Perlin noise is $O(1)$ per sample.
*   **Memory Complexity**: $O(1)$ for the noise generation itself, assuming the permutation table is pre-computed and stored. The table size is fixed at 512 integers [S1, unofficial].
*   **Performance**: The method is computationally inexpensive per sample, making it suitable for real-time height queries (e.g., placing objects at ground level) [S1, unofficial]. However, generating a full heightmap for a large area requires $N \times M$ samples, which can be costly if not cached or streamed.
*   **Strengths**: Simple to implement, produces smooth and organic terrain, easily adjustable via parameters (Scale, Octaves, Persistence).
*   **Limitations**: The resulting terrain is purely mathematical and may lack geological realism (e.g., erosion, tectonic features) without post-processing. The 2D projection ignores the 3D structure provided by the parent [Noise Field Generator](1302-noise-field-generator.md), potentially losing vertical variation details if the Z-dimension was significant [S1, unofficial].
*   **When to use**: Procedural terrain generation, placing objects at ground level, foliage scatter without a physical landscape actor [S1, unofficial].
*   **When not to use**: When high-fidelity, geologically accurate terrain is required, or when 3D volumetric details (caves, layers) are needed instead of a surface heightmap.

## Background
This method is derived from procedural noise techniques popularized by Ken Perlin in the 1980s. The specific implementation in the sources is part of a Minecraft-like voxel game clone built in Unreal Engine 5 [S1, unofficial]. The use of Perlin noise for terrain generation is a standard technique in procedural content generation (PCG) for games, allowing for infinite, deterministic landscapes. The parent method, [Noise Field Generator](1302-noise-field-generator.md), suggests an evolution towards 3D volumetric noise, but this heightmap method remains a common 2D projection for surface-level interactions.

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
