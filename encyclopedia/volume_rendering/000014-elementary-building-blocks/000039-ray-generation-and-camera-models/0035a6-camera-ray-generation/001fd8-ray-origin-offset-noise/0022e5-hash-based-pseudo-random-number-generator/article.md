# Hash-based Pseudo-Random Number Generator

Also called: Hash Functions, Hash-based PRNG, Integer to Float Mapping, Pseudo-Random Number Generator (Hash), Random Number Generation for Noise, hash12, hash13

## Overview
A hash-based pseudo-random number generator maps integer or floating-point coordinates to a deterministic pseudo-random value in the range $[0, 1)$ or $[-1, 1]$ using transcendental functions (typically sine) or multiplicative mixing [S4]. It generates independent values based on spatial coordinates, ensuring that the same coordinate always yields the same random value across different frames and devices [S1, unofficial][S4]. Unlike [Incremental Random Number Generation](4898-incremental-random-number-generation.md), which generates a sequence of correlated random numbers for a single ray, this method computes values on demand from input coordinates [S1, unofficial][S4].

## How it works
### Inputs and Outputs
- **Input**: A coordinate vector $p$. Common variants accept 2D ($vec2$) or 3D ($vec3$) inputs [S1, unofficial][S4].
- **Output**: A single floating-point scalar $r \in [0, 1)$ [S1, unofficial][S4].

### Data Structures
No persistent data structures are required for the basic hash functions [G]. The computation is stateless and depends only on the input coordinate $p$ and hardcoded constants [G].

### Formulas and Constants
Two primary families of hash functions are used: sine-based and sin-free [S4].

#### 1. Sine-based Hash (Standard)
This is the most common implementation in shaders due to its brevity [S1, unofficial][S4].

**2D Variant (`hash12` or `rnd`):**
$$ r = \text{fract}(\sin(\text{dot}(p, \mathbf{a})) \times C) $$

**3D Variant (`hash13`):**
$$ r = \text{fract}(\sin(\text{dot}(p, \mathbf{b})) \times C) $$

**Constants:**
- $\mathbf{a} = (127.1, 311.7)$ for `hash12` [S1, unofficial][S4].
- $\mathbf{a}' = (12.9898, 78.233)$ for alternative `rnd` [S2, unofficial][S4].
- $\mathbf{b} = (127.1, 311.7, 74.7)$ for `hash13` [S1, unofficial][S2, unofficial][S4].
- $C = 43758.5453123$ [S1, unofficial][S4].
- $C' = 43758.5453$ for `rnd` [S2, unofficial].

**Definitions:**
- $\text{dot}(u, v)$: Dot product of vectors $u$ and $v$.
- $\sin(x)$: Sine function (radians).
- $\text{fract}(x)$: Fractional part of $x$, defined as $x - \lfloor x \rfloor$ [G].

#### 2. Sin-free Hash (Cross-platform Stable)
Used to avoid precision issues with sine on certain GPUs [S4].

**2D Variant (`hash12`):**
$$
\begin{aligned}
p_3 &= \text{fract}(p_{xyx} \times 0.1031) \\
p_3 &= p_3 + \text{dot}(p_3, p_3^{zyx} + 33.33) \\
r &= \text{fract}((p_3.x + p_3.y) \times p_3.z)
\end{aligned}
$$

**Definitions:**
- $p_{xyx}$: Swizzle of $p$ to $(p.x, p.y, p.x)$.
- $p_3^{zyx}$: Swizzle of $p_3$ to $(p_3.z, p_3.y, p_3.x)$.
- Constants: $0.1031$ and $33.33$ are multiplicative constants chosen to spread bits [S4].

### Numbered Steps (Sine-based 2D)
1. **Dot Product**: Compute $h = p.x \cdot a_1 + p.y \cdot a_2$, where $a_1, a_2$ are the components of vector $\mathbf{a}$ [S1, unofficial][S4].
2. **Sine**: Compute $s = \sin(h)$ [S1, unofficial][S4].
3. **Scale**: Compute $scaled = s \cdot C$ [S1, unofficial][S4].
4. **Fraction**: Compute $r = scaled - \lfloor scaled \rfloor$ [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash12_sin(p_x, p_y):
    """
    Generates a pseudo-random float in [0, 1) from 2D coordinates using sine.
    Constants from [S1, unofficial][S4].
    """
    a1, a2 = 127.1, 311.7
    C = 43758.5453123
    h = p_x * a1 + p_y * a2
    return math.sin(h) * C - math.floor(math.sin(h) * C)

def hash13_sin(p_x, p_y, p_z):
    """
    Generates a pseudo-random float in [0, 1) from 3D coordinates using sine.
    Constants from [S1, unofficial][S4].
    """
    a1, a2, a3 = 127.1, 311.7, 74.7
    C = 43758.5453123
    h = p_x * a1 + p_y * a2 + p_z * a3
    return math.sin(h) * C - math.floor(math.sin(h) * C)

def hash12_sin_free(p_x, p_y):
    """
    Generates a pseudo-random float in [0, 1) from 2D coordinates using 
    multiplicative mixing, avoiding sine for stability [S4].
    """
    # p3 = fract(vec3(p.xyx) * 0.1031)
    p3_x = p_x * 0.1031 - math.floor(p_x * 0.1031)
    p3_y = p_y * 0.1031 - math.floor(p_y * 0.1031)
    p3_z = p3_x  # p.xyx -> x, y, x

    # p3 += dot(p3, p3.yzx + 33.33)
    # p3.yzx is (p3_z, p3_y, p3_x)
    dot_val = p3_x * (p3_z + 33.33) + p3_y * (p3_y + 33.33) + p3_z * (p3_x + 33.33)
    p3_x += dot_val
    p3_y += dot_val
    p3_z += dot_val

    return (p3_x + p3_y) * p3_z - math.floor((p3_x + p3_y) * p3_z)
```

### Parameters
- **Constants**: The specific irrational constants ($127.1, 311.7, 43758.5453123$) are critical for bit dispersion [S1, unofficial][S4].
- **Input Type**: Integer inputs are preferred for voxel grids to ensure exact reproducibility [G]. Floating-point inputs may suffer from precision drift if not handled carefully [S2, unofficial][S4].

### Edge Cases and Pitfalls
- **Precision Sensitivity**: The sine-based hash is sensitive to floating-point precision differences between GPUs and CPUs. On some hardware, the sine function may produce slightly different results, leading to visual artifacts or non-deterministic behavior [S4].
- **Statistical Quality**: The sine-based hash has known statistical shortcomings (e.g., correlations in higher dimensions). For high-quality renderers requiring rigorous randomness, integer/bitwise hashes (like Murmur) are recommended over sine-based ones [S1, unofficial].
- **Source Uncertainty**: The exact origin of the specific constants in the sine hash is unknown, possibly dating back to Rey 1998, cited by TestU01 [S3, unofficial].

## Subtopics

### [Deterministic Integer Hashing](4149-deterministic-integer-hashing.md)
Deterministic integer hashing provides a platform-independent method for generating pseudo-random integers from integer coordinates, ensuring identical bit-exact results between CPU (TypeScript/C++) and GPU (GLSL) environments. It solves the reproducibility problem in procedural content generation where floating-point sine-based hashing drifts due to library implementation differences. Unlike its sibling [Hash-based Pseudo-Random Number Generator](2896-hash-based-pseudo-random-number-generator.md) which relies on transcendental functions, this method uses only 32-bit bitwise operations and multiplication, guaranteeing consistent wrapping behavior for negative coordinates.

### [Permutation Table Generation](4174-permutation-table-generation.md)
Generates a shuffled permutation table of size 512 using a seeded random shuffle, then duplicates the first 256 entries to avoid modulo operations during hash-based pseudo-random number generation. It serves as the foundational data structure for [Hash-based Pseudo-Random Number Generator](2896-hash-based-pseudo-random-number-generator.md), ensuring deterministic yet uniformly distributed indices for noise functions. Unlike [Deterministic Integer Hashing](4149-deterministic-integer-hashing.md) which computes hashes on the fly, this method pre-computes a lookup table, trading memory for computational speed and avoiding expensive bitwise operations or trigonometric functions during ray generation.

## Operating characteristics
- **Time Complexity**: $O(1)$ per coordinate. The sine-based hash involves a dot product, one sine, one multiplication, and one fractional part extraction [S4]. The sin-free hash involves several multiplications and additions but no transcendental functions [S4].
- **Memory Complexity**: $O(1)$ for the basic hash functions. No memory is allocated [G].
- **Measured Performance**: Sine-based hashes are extremely fast on GPUs due to hardware-accelerated sine instructions [G]. Sin-free hashes are faster on hardware where sine is expensive or imprecise [S4].
- **Strengths**: Extremely low memory footprint; deterministic; easy to implement in shaders; no pre-computation required [G].
- **Limitations**: Statistical quality is inferior to cryptographic or high-quality PRNGs (e.g., PCG, Murmur); precision-dependent on sine implementation; not suitable for applications requiring rigorous statistical randomness [S1, unofficial].
- **When to Use**: Ray origin offset noise, voxel world generation, procedural texture coordinate hashing [S1, unofficial][S4].
- **When Not to Use**: Applications requiring high-quality statistical randomness, cryptographic security, or strict cross-platform bit-exactness without sin-free variants [S1, unofficial][S4].

## Background
The sine-based hash function `fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123)` is a staple in shader programming, popularized by blogs and StackOverflow answers for its brevity and speed [S1, unofficial][S4]. Its origin is uncertain, possibly dating back to Rey 1998, as cited by TestU01 [S3, unofficial]. It is used in voxel world shaders, terrain generation, and procedural noise implementations [S1, unofficial][S2, unofficial][S4]. The sin-free variant was developed to address cross-platform precision issues [S4].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Minecraft Voxel DDA Raycast - Shadertoy, https://www.shadertoy.com/view/WccyWl, fetched 2026-10-26 (unofficial)
[S3] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S4] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
