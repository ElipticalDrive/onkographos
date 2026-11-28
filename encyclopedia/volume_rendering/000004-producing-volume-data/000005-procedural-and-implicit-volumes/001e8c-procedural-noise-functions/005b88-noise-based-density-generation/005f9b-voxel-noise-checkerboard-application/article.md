# Voxel Noise/Checkerboard Application

Also called: Checker Pattern, Checkerboard Cell Noise, Procedural Voxel Noise, checker

## Overview
Voxel Noise/Checkerboard Application generates a deterministic, binary 3D checkerboard pattern by evaluating the parity of the integer grid coordinates of the input position [S1, unofficial]. It serves as a high-frequency, low-cost procedural density or material mask, distinct from smooth gradient noise methods like Perlin or Simplex by providing sharp, discontinuous transitions between cells [S1, S2, unofficial]. Unlike [Perlin-Worley Noise for Cloud Density](7139-perlin-worley-noise-for-cloud-density.md) which blends distances, this method relies on discrete integer arithmetic to switch between two states [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D query position $\mathbf{P} = (x, y, z)$ and a frequency scale factor $s$.
*   **Output:** A scalar value $V \in \{-1.0, 1.0\}$ (or $\{0.0, 1.0\}$ depending on implementation) representing the checkerboard state [S1, unofficial].

**Data Structures:**
No complex data structures are required. The method relies on integer arithmetic on the input coordinates.

**Steps:**
1.  **Scale and Integerize:** Compute the integer grid coordinates $(i_x, i_y, i_z)$ by scaling the input position by the frequency $s$ and truncating to integers.
    $$ i_x = \lfloor x \cdot s \rfloor, \quad i_y = \lfloor y \cdot s \rfloor, \quad i_z = \lfloor z \cdot s \rfloor $$
    Note: Some implementations use casting `(int)` which truncates toward zero, while others use `floor` which truncates toward negative infinity. For positive coordinates, these are identical. For negative coordinates, `floor` preserves the grid alignment relative to the origin, whereas `(int)` may shift the pattern [S1, unofficial].
2.  **Compute Parity:** Calculate the sum of the integer coordinates.
    $$ S = i_x + i_y + i_z $$
3.  **Determine State:** Return the binary value based on the parity of $S$.
    $$ V = \begin{cases} 1.0 & \text{if } S \pmod 2 == 0 \\ -1.0 & \text{if } S \pmod 2 \neq 0 \end{cases} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_checkerboard(pos: Vector3, scale: float) -> float:
    # Step 1: Scale and integerize coordinates
    # Using floor to ensure consistent grid alignment for negative coordinates
    ix = floor(pos.x * scale)
    iy = floor(pos.y * scale)
    iz = floor(pos.z * scale)
    
    # Step 2: Compute parity
    # Sum of integer coordinates
    coord_sum = ix + iy + iz
    
    # Step 3: Return binary value based on parity
    if coord_sum % 2 == 0:
        return 1.0
    else:
        return -1.0
```

**Parameters:**
*   `scale` (float): Frequency multiplier. Typical values depend on the desired voxel size relative to the scene units. Not given in the sources.
*   `pos` (Vector3): Input 3D position.

**Edge Cases and Pitfalls:**
*   **Boundary Discontinuities:** The function is discontinuous at integer boundaries. This is intentional for the checkerboard effect but can cause aliasing if sampled at high frequencies without mipmapping or if used in ray marching without proper step size control.
*   **Negative Coordinates:** As noted, the choice between `floor` and truncation `(int)` affects the pattern's phase for negative coordinates. `floor` is generally preferred for mathematical consistency in grid-based systems [S1, unofficial].
*   **Zero Scale:** Division by zero or infinite scaling if `scale` is 0. Guard against `scale <= 0`.

## Subtopics

### Terrain Voxel Shader
A spatial shader implementation that decodes terrain material IDs from vertex colors and renders textured biome materials using Fractal Brownian Motion (FBM) noise for variation [S3, unofficial]. It uses vertex colors to store material IDs instead of separate texture lookups or uniform arrays, allowing per-vertex material variation [S3, unofficial].
*   **Parameters:** `texture_scale` (default 0.08), `terrain_brightness` (default 1.0), `texture_detail_strength` (default 0.65), `slope_darkening` (default 0.08), `large_variation_strength` (default 0.10) [S3, unofficial].
*   **Mechanism:** It employs a hash function for noise, value noise interpolation, and an FBM loop with 4 octaves to generate variation. Material IDs for Grass, Clay, Stone, Iron, Path, Sand, Sulfur, and Volcanic Stone are decoded from vertex color channels [S3, unofficial].
*   **Comparison:** Unlike the pure binary checkerboard, this variant integrates noise for organic variation and uses vertex attributes for material selection, making it suitable for complex terrain rendering rather than simple binary masking [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The operation involves basic arithmetic (multiplication, floor, addition, modulo) and a conditional branch. It is significantly faster than gradient-based noise functions which require multiple dot products and interpolations [S1, unofficial].
*   **Memory Complexity:** $O(1)$. No lookup tables or textures are required for the basic checkerboard pattern [S1, unofficial].
*   **Strengths:** Extremely fast evaluation; deterministic; no memory bandwidth overhead; ideal for binary masks or high-frequency detail where smoothness is not required [S1, unofficial].
*   **Limitations:** Discontinuous; prone to aliasing at high frequencies; lacks the organic blending characteristics of gradient noise [S1, unofficial].
*   **When to use:** When a sharp, grid-aligned binary pattern is needed (e.g., voxel terrain base layers, checkerboard masks) [S1, unofficial].
*   **When not to use:** When smooth transitions or organic variation are required; use [Perlin-Worley Noise for Cloud Density](7139-perlin-worley-noise-for-cloud-density.md) or [Noise-based Density Generation](16925-noise-based-density-generation.md) instead [S1, S2, unofficial].

## Background
The checkerboard pattern is a fundamental concept in computer graphics and procedural generation, often used as a base for voxel-based worlds. The specific implementation described here, evaluating the parity of integer grid coordinates, is a standard technique for generating binary 3D checkerboards efficiently [S1, unofficial]. It appears in various voxel engine implementations and procedural terrain systems, such as those found in open-source voxel games and engines [S1, S3, unofficial]. The integration of such patterns with vertex-colored material decoding represents a modern optimization in shader-based terrain rendering [S3, unofficial].

## Sources
[S1] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
[S2] aresrpg/aresrpg-engine, https://github.com/aresrpg/aresrpg-engine, fetched 2026-10-27 (unofficial)
[S3] TsokasGit/primitive-voxels-godot, https://github.com/TsokasGit/primitive-voxels-godot, fetched 2026-10-27 (unofficial)
