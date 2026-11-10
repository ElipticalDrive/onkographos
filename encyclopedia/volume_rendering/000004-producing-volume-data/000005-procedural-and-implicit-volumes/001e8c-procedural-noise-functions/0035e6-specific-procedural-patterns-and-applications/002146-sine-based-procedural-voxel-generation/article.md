# Sine-based procedural voxel generation

Also called: Trigonometric voxel pattern

## Overview
Sine-based procedural voxel generation creates volumetric data by evaluating trigonometric functions against spatial coordinates to determine voxel occupancy and color. It solves the problem of generating deterministic, periodic, and smooth geometric structures without storing explicit mesh or voxel grid data. Unlike [Simple Heightmap Voxel Generation](4649-simple-heightmap-voxel-generation.md) which focuses on planar terrain, or [Procedural Tree Voxel Generation](4445-procedural-tree-voxel-generation.md) which uses hierarchical branching, this method relies on global periodic wave interference to define shape boundaries and internal density variations.

## How it works
**Inputs and Outputs**
*   **Input**: 3D integer voxel coordinates $(pos_x, pos_y, pos_z)$, grid dimensions (width, height, depth), and a scaling constant.
*   **Output**: A voxel state (empty or filled) and an RGBA color value.

**Data Structures**
*   **Coordinates**: Integer triplets representing voxel indices.
*   **Color**: 32-bit integer or struct with 8-bit channels (R, G, B, A).

**Parameters**
*   `mod`: The periodicity of the pattern, typically derived from the grid width divided by a divisor (e.g., 8) [S1, unofficial].
*   `scale`: A constant multiplier for color channels, typically `0x20` (32) [S1, unofficial].

**Algorithm Steps**
1.  **Periodicity Calculation**: Compute the modulo value `mod` based on the grid width. For a width $W$, `mod = W / 8` [S1, unofficial].
2.  **Normalization**: Convert integer coordinates to a normalized range $[-1, 1]$ within the periodic cell:
    $$ xx = \frac{(pos_x \pmod{\text{mod}}) \cdot 2}{\text{mod}} - 1 $$
    $$ yy = \frac{(pos_y \pmod{\text{mod}}) \cdot 2}{\text{mod}} - 1 $$
    $$ zz = \frac{(pos_z \pmod{\text{mod}}) \cdot 2}{\text{mod}} - 1 $$
    [S1, unofficial]
3.  **Radius Calculation**: Compute a variable radius $r$ that defines the shape boundary using a sum of normalized coordinates:
    $$ r = \left( \sin\left( \frac{pos_x + pos_y + pos_z}{5 \cdot \text{mod}} \right) + 1 \right) \cdot 0.3 $$
    [S1, unofficial]
4.  **Occupancy Test**: A voxel is filled if it satisfies two conditions:
    *   It lies inside the spherical boundary defined by the normalized coordinates:
        $$ xx^2 + yy^2 + zz^2 \le r^2 $$
    *   It passes a secondary frequency filter:
        $$ \sin\left( \frac{pos_x \cdot 16 + pos_y \cdot 20 + pos_z \cdot 30}{\text{width}} \right) \ge 0 $$
    [S1, unofficial]
    If either condition fails, the voxel is empty.
5.  **Color Assignment**: If filled, compute RGB values scaled by the position relative to `mod`:
    $$ R = \frac{pos_x}{\text{mod}} \cdot 0x20 $$
    $$ G = \frac{pos_y}{\text{mod}} \cdot 0x20 $$
    $$ B = \frac{pos_z}{\text{mod}} \cdot 0x20 $$
    $$ A = 255 $$
    [S1, unofficial]

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sine_voxel_generation(pos_x, pos_y, pos_z, width):
    # Step 1: Compute mod
    mod = width // 8
    
    # Step 2: Normalize coordinates to [-1, 1]
    xx = ((pos_x % mod) * 2.0 / mod) - 1.0
    yy = ((pos_y % mod) * 2.0 / mod) - 1.0
    zz = ((pos_z % mod) * 2.0 / mod) - 1.0
    
    # Step 3: Compute radius
    # Note: The source divides the sum by 5, but the normalization 
    # suggests the argument should be dimensionless. 
    # We follow the source formula exactly: sin((pos.x/mod + pos.y/mod + pos.z/mod)/5)
    # However, the source code snippet says: sin((pos.x/mod + pos.y/mod + pos.z/mod)/5)
    # Let's stick to the explicit formula in the notes:
    # radius = (sin((pos.x/mod + pos.y/mod + pos.z/mod)/5) + 1) * 0.3
    
    # Re-evaluating source [S1, unofficial]: "sin((pos.x/mod + pos.y/mod + pos.z/mod)/5)"
    # This implies the inputs to sin are already normalized by mod.
    
    arg = (pos_x / mod + pos_y / mod + pos_z / mod) / 5.0
    radius = (sin(arg) + 1.0) * 0.3
    
    # Step 4: Occupancy Test
    dist_sq = xx*xx + yy*yy + zz*zz
    
    # Check spherical boundary
    if dist_sq > radius * radius:
        return None # Empty
        
    # Check secondary filter
    filter_arg = (pos_x * 16 + pos_y * 20 + pos_z * 30) / width
    if sin(filter_arg) < 0:
        return None # Empty
        
    # Step 5: Color Assignment
    scale = 0x20 # 32
    r = int((pos_x / mod) * scale)
    g = int((pos_y / mod) * scale)
    b = int((pos_z / mod) * scale)
    a = 255
    
    return (r, g, b, a)
```

**Edge Cases and Pitfalls**
*   **Division by Zero**: If `mod` is 0 (e.g., width < 8), the normalization fails. Guard against small grid sizes [G].
*   **Precision**: Floating point errors in `sin` arguments can cause boundary jitter at high resolutions. Using integer arithmetic for the modulo and division where possible improves determinism [G].
*   **Boundary Continuity**: The pattern is periodic with `mod`. Ensure the grid dimensions are multiples of `mod` to avoid visible seams at the edges [G].

## Subtopics

### Displacement Noise
Harmonic trigonometric displacement noise adds organic or porous textures to volumes by modulating density. It is distinct from the primary generation method by applying a multiplicative amplitude factor to the product of sines along each axis. It is chosen when a more complex, non-spherical internal structure is required.
Formula: $amp \cdot \sin(freq \cdot x) \cdot \sin(freq \cdot y) \cdot \sin(freq \cdot z)$ [S2, unofficial].
Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def displacement_noise(p: Vec3, freq: f32, amp: f32) -> f32:
    return amp * sin(freq * p.x) * sin(freq * p.y) * sin(freq * p.z)
```
[S2, unofficial]

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel. The calculation involves a constant number of trigonometric and arithmetic operations.
*   **Memory Complexity**: $O(1)$ auxiliary space. No large lookup tables or recursion stacks are required.
*   **Performance**: Extremely fast on modern CPUs/GPUs due to the lack of branching dependencies between voxels (embarrassingly parallel).
*   **Strengths**: Deterministic, seamless tiling, low memory footprint, easy to animate by varying time in the sine arguments.
*   **Limitations**: Limited to periodic, smooth shapes. Cannot easily represent sharp features, arbitrary topology, or non-repeating organic structures without complex superposition.
*   **When to use**: Ideal for generating regular patterns, crystals, or simple organic blobs where performance is critical and explicit geometry storage is undesirable.
*   **When not to use**: When complex, non-periodic, or hierarchical structures (like trees or terrain) are needed; use [Procedural Tree Voxel Generation](4445-procedural-tree-voxel-generation.md) or [Simple Heightmap Voxel Generation](4649-simple-heightmap-voxel-generation.md) instead.

## Background
This method is derived from early procedural texture generation techniques that utilized trigonometric functions for their smoothness and periodicity. The specific implementation cited here originates from the `zazen` voxel renderer project, which demonstrates efficient runtime voxel generation using simple harmonic functions [S1, unofficial]. It relates to the broader family of [Specific Procedural Patterns and Applications](7615-specific-procedural-patterns-and-applications.md) which includes noise-based and fractal approaches.

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
