# Box with Smoothed/Eroded Edges

Also called: FunctionMakeBox

## Overview
This method converts a signed distance field (SDF) value into a volumetric density for a box primitive, applying a sigmoid-based erosion at the boundaries and adding Perlin noise to roughen the surface [S1, unofficial]. It differs from standard SDF-to-density conversions by explicitly expanding the box geometry to create a specific "eroded" falloff zone rather than simply smoothing the existing boundary, and it integrates procedural noise directly into the density calculation [S1, unofficial].

## How it works
**Inputs:**
- `InDensity`: The input density value (float).
- `P`: The 3D point in world space (`FVector` or `float[3]`).
- `InBox`: The bounding box defining the primitive (`FBox` or equivalent struct with min/max coordinates).

**Outputs:**
- `R`: The computed density value (float), clamped to $[0, 1]$.

**Parameters:**
- `E = 50`: Box expansion and erosion margin in world units [S1, unofficial].
- `D = 100`: Sigmoid steepness controlling the smoothness of the eroded edge [S1, unofficial].
- `NoisePositionScale = 0.005f`: Spatial frequency multiplier for Perlin noise [S1, unofficial].
- `NoiseValueScale = 0.1`: Amplitude multiplier for Perlin noise [S1, unofficial].

**Data Structures:**
- `FVector`: A 3D vector with components `X`, `Y`, `Z` (or `x`, `y`, `z`).
- `FBox`: A bounding box defined by minimum and maximum coordinates. Let `Min` and `Max` be the bounds. The box extends from `Min.X` to `Max.X`, `Min.Y` to `Max.Y`, and `Min.Z` to `Max.Z`.

**Steps:**
1. If `InDensity < 0.5f`, return `InDensity` immediately [S1, unofficial].
2. Define the expanded box boundaries by adding `E` to the maximums and subtracting `E` from the minimums:
   - `ExtendXP = Max.X + E`
   - `ExtendXN = Min.X - E`
   - `ExtendYP = Max.Y + E`
   - `ExtendYN = Min.Y - E`
   - `ExtendZP = Max.Z + E`
   - `ExtendZN = Min.Z - E`
3. Check if point `P` intersects the expanded box. If `P` is outside the expanded box (i.e., `P.X < ExtendXN` or `P.X > ExtendXP` or `P.Y < ExtendYN` or `P.Y > ExtendYP` or `P.Z < ExtendZN` or `P.Z > ExtendZP`), return `0.0f` [S1, unofficial].
4. Initialize `R = 0.0f` [S1, unofficial].
5. For each axis (X, Y, Z), if `P` is within `E` units of the box face (i.e., inside the expanded region but potentially outside the original box), compute the sigmoid density falloff:
   - For the X-axis positive face: $D_{XP} = \frac{1}{1 + e^{(ExtendXP - P.X) / D}}$
   - For the X-axis negative face: $D_{XN} = \frac{1}{1 + e^{(-ExtendXN + P.X) / D}}$
   - Sum the contributions: $D_X = D_{XP} + D_{XN}$
   - Repeat for Y and Z axes to get $D_Y$ and $D_Z$.
   - Note: The source code example only explicitly shows X-axis calculation, but the text states "For each axis" [S1, unofficial].
6. Compute Perlin noise $N$ at position `P` using `NoisePositionScale` and `NoiseValueScale` [S1, unofficial].
7. Add the noise to the density sum: $R = D_X + D_Y + D_Z + N$ (assuming summing all axes as implied by "For each axis" and the structure of similar functions, though the code snippet only sums X. Given the text "For each axis... add Perlin noise", it is safer to assume the noise is added once to the total density, and the density contributions from all axes are summed. The code snippet `R = DensityX + N` suggests `DensityX` might be the sum of all axes or just X. However, standard box SDFs usually sum contributions or take the max. The text says "compute a sigmoid density falloff and add Perlin noise" for *each* axis. This implies $R = \sum_{axis} D_{axis} + N$. Let's stick to the explicit code snippet for the final combination if possible, but the text is ambiguous. The code snippet `const float DensityX = (DensityXP + DensityXN);` and `R = DensityX + N;` suggests only X is calculated in that snippet. However, the step says "For each axis". I will implement the full axis summation as per the text description, as the code snippet is likely partial [S1, unofficial].
8. Clamp `R` to $[0, 1]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def FunctionMakeBox(InDensity, P, InBox, E=50.0, D=100.0, NoisePositionScale=0.005, NoiseValueScale=0.1):
    # Step 1: Early exit for low density
    if InDensity < 0.5:
        return InDensity

    # Step 2: Define expanded box boundaries
    Min = InBox.Min
    Max = InBox.Max
    
    ExtendXP = Max.X + E
    ExtendXN = Min.X - E
    ExtendYP = Max.Y + E
    ExtendYN = Min.Y - E
    ExtendZP = Max.Z + E
    ExtendZN = Min.Z - E

    # Step 3: Check intersection with expanded box
    if P.X < ExtendXN or P.X > ExtendXP or \
       P.Y < ExtendYN or P.Y > ExtendYP or \
       P.Z < ExtendZN or P.Z > ExtendZP:
        return 0.0

    # Step 4: Initialize R
    R = 0.0

    # Step 5: Compute sigmoid falloff for each axis
    # X Axis
    DensityXP = 1.0 / (1.0 + exp((ExtendXP - P.X) / D))
    DensityXN = 1.0 / (1.0 + exp((-ExtendXN + P.X) / D))
    DensityX = DensityXP + DensityXN

    # Y Axis
    DensityYP = 1.0 / (1.0 + exp((ExtendYP - P.Y) / D))
    DensityYN = 1.0 / (1.0 + exp((-ExtendYN + P.Y) / D))
    DensityY = DensityYP + DensityYN

    # Z Axis
    DensityZP = 1.0 / (1.0 + exp((ExtendZP - P.Z) / D))
    DensityZN = 1.0 / (1.0 + exp((-ExtendZN + P.Z) / D))
    DensityZ = DensityZP + DensityZN

    # Sum axis contributions
    DensitySum = DensityX + DensityY + DensityZ

    # Step 6: Compute Perlin noise
    N = PerlinNoise(P, NoisePositionScale, NoiseValueScale)

    # Step 7: Add noise to density
    R = DensitySum + N

    # Step 8: Clamp result
    R = max(0.0, min(1.0, R))
    
    return R
```

**Edge Cases and Pitfalls:**
- If `P` is far outside the expanded box, the function returns 0.
- The sigmoid function can suffer from overflow/underflow if the exponent is very large. Using `exp(-x)` for large positive `x` is safer.
- The noise function must be deterministic and consistent with other parts of the terrain generator.
- The parameter `E` defines the width of the eroded region. If `E` is too small, the erosion effect is negligible. If `E` is too large, the box appears significantly larger than intended.

## Subtopics

### Vertical Cylinder with Smoothed Edges
Computes density for a vertical cylinder, applying a sigmoid-based radial erosion and Perlin noise to the cylinder walls [S1, unofficial]. It differs from the box method by using radial distance for the sigmoid falloff and checking vertical bounds separately [S1, unofficial]. Choose this when generating cylindrical structures like trees or pillars [S1, unofficial]. Parameters: `E=50`, `NoisePositionScale=0.007f`, `NoiseValueScale=0.1` [S1, unofficial]. Formula: $D_{cyl} = \frac{1}{1 + e^{(R_{target} - R_{dist}) / 100}} + N$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, as it involves a fixed number of arithmetic operations and a single noise evaluation.
- **Memory Complexity:** $O(1)$, no additional data structures are allocated.
- **Performance:** Fast enough for real-time procedural generation. The sigmoid and noise calculations are lightweight.
- **Strengths:** Provides smooth, natural-looking edges with controlled erosion. The noise adds visual detail without requiring high-resolution geometry.
- **Limitations:** The fixed parameters (`E`, `D`, noise scales) may not suit all artistic styles. The method is specific to the UE4VoxelTerrain context and may not be directly portable to other engines without adaptation.
- **When to use:** When generating voxel terrain with procedural objects like boxes and cylinders that need smooth, eroded edges.
- **When not to use:** When exact hard edges are required, or when the performance cost of noise evaluation is prohibitive (though it is generally low).

## Background
This method originates from the UE4VoxelTerrain project, an unofficial voxel terrain generator for Unreal Engine 4 [S1, unofficial]. It is implemented in the `UCudaTerrainGeneratorComponent` class, specifically in the `FunctionMakeBox` method [S1, unofficial]. The method is part of a larger system for generating procedural voxel terrain, including foliage and cave layers [S1, unofficial].

## Sources
[S1] bw2012/UE4VoxelTerrain, https://github.com/bw2012/UE4VoxelTerrain, fetched 2026-10-26 (unofficial)
